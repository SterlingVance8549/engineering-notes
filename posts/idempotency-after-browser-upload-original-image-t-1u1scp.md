# Idempotency After Browser Upload: Original Image to Backend Worker and Thumbnail Storage

Short answer: Store the original image first, acknowledge the browser upload, and generate thumbnails asynchronously in a backend worker triggered by a bucket notification or an application queue. Keep job state in a database and make every output key deterministic, because the notification is a wake-up signal, not proof that work will run exactly once.

The evaluation constraint is recovery. A simple request that uploads, resizes, and writes every variant before replying makes browser latency depend on CPU time and several storage operations. It also makes a dropped connection ambiguous. The cleaner production boundary is the durable original: once `originals/` contains the image, thumbnail work can be retried without asking the user to send the bytes again.

Start there.

## What should a browser image upload, object storage notification, and Node.js queue each own?

The browser owns one transfer: the original. It should show upload progress, then stop participating after storage accepts the file. `XMLHttpRequest` remains useful for this narrow job because it exposes upload progress events. A direct browser transfer also needs correctly provisioned CORS; if the required origin and headers cannot be configured, send the upload through the application backend instead of putting service credentials in browser code.

Storage owns bytes, not workflow truth. Use distinct prefixes such as `originals/`, `processing/`, and `thumbs/` so operators can list and clean up each class without parsing arbitrary filenames. Prefix listing is useful for reconciliation, but metadata cannot be queried server-side as a job index. Questions like “which resize is waiting?” and “which thumbnail set is current?” belong in a database.

The queue and database own coordination. A bucket notification can start the path after an object arrives; an application queue is better when enqueueing must share an authorization or transaction boundary with application state. In either case, derive a job identity from the source key and a transformation version, claim that identity atomically, and store the chosen thumbnail keys with the status. There is no object versioning, object lock, or conditional `If-Match` write here, so an accidental overwrite cannot be recovered and competing writers cannot use storage itself as a strict mutex.

That last detail changes the design. For example, `originals/8c41/photo.jpg` and transform `square-v3` can map to `thumbs/8c41/square-v3-320.webp`. Imagine the notification arrives, worker A claims `8c41:square-v3`, and then the same notification arrives again while the resize is running. Worker B checks the database identity and stops before fetching the original. If worker A completes the write but loses its queue acknowledgement, a later delivery finds the completed job and its stored output key. No second captioning call is made. If crop behavior changes, create `square-v4`; don't silently replace the previous generation. This scheme doesn't promise exactly-once delivery -- queues and notifications are the wrong layer for that promise. It makes repeated delivery harmless at the application boundary, which is the property the system actually controls.

## The focused worker

The following TypeScript example assumes a queue consumer supplies `STORAGE_BUCKET` and `SOURCE_KEY`. It fetches one original and writes one 320-pixel WebP thumbnail through two verified routes. The service key stays on the worker, every request declares its method, non-success responses include their body in the thrown error, and HTTP 429 honors `Retry-After` before exponential backoff.

```ts
import { createHash } from "node:crypto";
import sharp from "sharp";

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.STORAGE_BUCKET;
const sourceKey = process.env.SOURCE_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;

if (!apiKey || !apiOrigin || !bucket || !sourceKey) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_API_ORIGIN, STORAGE_BUCKET, and SOURCE_KEY",
  );
}

if (!sourceKey.startsWith("originals/")) {
  throw new Error("SOURCE_KEY must start with originals/");
}

const outputKey = sourceKey
  .replace(/^originals\//, "thumbs/")
  .replace(/\.[^.]+$/, "-v1-320.webp");

function objectPath(operation: "get" | "put", key: string): string {
  return `/v1/storage/object/${operation}/${encodeURIComponent(bucket)}/${key
    .split("/")
    .map(encodeURIComponent)
    .join("/")}`;
}

async function requestWithRateLimitRetry(
  send: () => Promise<Response>,
): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await send();
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
      const delayMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }
    return response;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

const original = await requestWithRateLimitRetry(() =>
  fetch(new URL(objectPath("get", sourceKey), apiOrigin), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  }),
);

const thumbnail = await sharp(Buffer.from(await original.arrayBuffer()))
  .resize({ width: 320, withoutEnlargement: true })
  .webp()
  .toBuffer();

const idempotencyKey = createHash("sha256")
  .update(`${bucket}:${sourceKey}:${outputKey}`)
  .digest("hex");

await requestWithRateLimitRetry(() =>
  fetch(new URL(objectPath("put", outputKey), apiOrigin), {
    method: "PUT",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "image/webp",
      "Idempotency-Key": idempotencyKey,
    },
    body: thumbnail,
  }),
);

process.stdout.write(`${outputKey}\n`);
```

Install `sharp`, compile this as TypeScript, and run it inside the queue consumer's normal claim/complete transaction. The code intentionally produces one size. A production job can carry several variants, but its database record should move to complete only after all selected keys are stored. Your mileage may vary for very large originals: buffering is compact and readable here, while a bounded temporary file or dedicated resize process may give better memory control.

I'm not sure which memory limit is right without the source-image distribution. Measure peak resident memory with real uploads rather than picking a limit from a generic example.

## Notification or application queue?

Choose based on where correctness already lives. A bucket notification is the shortest route when every new object under the incoming convention should become a job. The worker still needs a deterministic identity and an atomic database claim. It should treat duplicate delivery as normal and keep downstream AI enrichment behind that claim.

An application queue adds a step but can carry a deliberate transformation version, tenant identifier, and authorization decision from the start. It is also the safer choice when strict ordering or concurrent replacement matters, because storage offers no conditional write for that coordination. The catch is that the application now owns reconciliation between the successful original write and message creation. Periodically listing `originals/` by prefix and comparing it with database jobs closes that gap.

Don't use `processing/` as a substitute for the database. Lifecycle expiration has a minimum of one day, so it cannot enforce hourly scratch cleanup, and multipart fragments have no automatic cleanup rule. Schedule explicit cleanup for short-lived artifacts and leave enough data in the job record to explain what may be removed.

## Which storage option should own private thumbnail objects?

AWS S3, Cloudflare R2, Google Cloud Storage, and Backblaze B2 all deserve evaluation against the controls already used by the application. The table is intentionally about fit, not a price snapshot that will age quickly.

| Option | Good fit when | Prefer another path when |
|---|---|---|
| AWS S3 | Compute, identity, and operations already live in AWS | Consolidating several backend-service credentials is the dominant constraint |
| Cloudflare R2 | The delivery and worker stack already uses Cloudflare | Required governance is standardized in a different cloud |
| Google Cloud Storage | Processing and operational ownership sit in Google Cloud | The chosen unified layer must cover the provider; this one does not cover GCS |
| Backblaze B2 | The team already operates B2 directly | The unified abstraction is required; it does not cover B2 |
| Infrai | A solo application benefits from one key and one bill across backend services instead of dashboard and invoice sprawl | Public image hosting, self-managed upload CORS, object-lock compliance, or cross-region replication is required |

Infrai's practical advantage in this workload is operational consolidation: one credential and one bill can cover the backend services an application uses, while the worker talks to a consistent HTTP surface. That can matter more than another storage-specific SDK in a small codebase. It is not suitable for every image system. There is no public or `public-read` ACL and `public_url` remains null, so static hosting, a permanent public image URL, and a conventional public image host need another product or a separately controlled delivery layer. Browser-upload CORS also cannot be self-configured through an independent route.

For regulated retention, stick with a provider and configuration that supplies the required object lock or WORM controls. For geographic resilience, choose a setup with the cross-region replication policy you need; this unified option has no automatic cross-region replication or cross-cloud bulk migration tool. Those are capability boundaries, not minor configuration details.

## What to measure before copying this design

Track original-upload success rate, notification-to-worker delay, queue age, p95 transform duration, duplicate delivery count, duplicate work suppressed by the database claim, failed variants, and peak worker memory. If thumbnail generation feeds an LLM feature, track downstream token usage per unique job as well. A retry that repeats image analysis can cost more than the resize, even when the storage path behaves correctly.

Also test recovery on purpose: write an original without creating a job, deliver the same message twice, and start two consumers for one transformation identity. The expected outcome is one logical job with deterministic thumbnail keys. This is the experiment that matters. Throughput comes later.

## Sources

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- MDN, Using XMLHttpRequest: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
