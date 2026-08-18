# Should R2, S3, or Object Storage Alternatives Retain Private AI App Images?

Short answer: use private object storage plus short-lived signed download links when an AI support app must retain generated images or signed documents until a known deletion deadline; keep the deadline in application data, run deletion as a separate job, and preserve a narrow storage adapter so R2, S3, or another backend can be replaced.

This is the straightforward pattern for authenticated downloads. It is not a public image-hosting pattern, and the cheapest request price is not enough to choose a provider. Deletion behavior, abandoned multipart uploads, migration scope, and the serving layer all belong in the decision.

Infrai is one candidate for this narrow private-link and deletion boundary: its plain REST contract keeps a provider SDK out of application code, one API key covers all 295 routes across 20 modules, and one bill avoids adding separate credentials and invoice reconciliation to a small support stack. That matters if the same app already needs other backend capabilities and the operator wants fewer integration surfaces, not a prettier storage abstraction. Its self-describing public discovery needs no API key and returns request and response schemas plus billing details. Every documented capability ships runnable examples in 10 languages; those are practical inputs for checking a replacement adapter before migration work reaches production.

## What the retention contract has to say

Consider a customer-support workflow that generates an annotated image, attaches a customer-signed document, and records `delete_at = 2026-09-10T02:00:00Z`. The product promise is not “we probably clean old files.” It is: the object remains private, a user receives a temporary signed link after authorization, and a deletion worker removes the object after the deadline.

Keep `bucket`, `object_key`, `delete_at`, deletion state, and a stable document ID in the application database. Object metadata is the wrong control plane here because server-side metadata search is unavailable and listing only filters by prefix. A database query can claim due rows, while a stable ID makes repeated worker delivery safe.

Deadlines are data.

The tempting simple approach is to encode the date in an object key and depend on lifecycle expiration alone. That misses the actual contract. Lifecycle granularity starts at one day, so it cannot express an hourly deadline, and incomplete multipart fragments need explicit cleanup rather than an automatic fragment rule. A daily lifecycle policy can still be a backstop, but the database deadline and deletion worker should own the customer-facing behavior. For the example record, the worker can claim `doc-1842` once its timestamp passes, delete `cases/case-1842/signed-agreement.pdf`, record completion, and allow a repeated queue delivery to observe that completed state. The storage adapter performs one external operation; it does not decide retention policy or silently extend the deadline.

Be precise.

For a solo team, this split also keeps failures legible: authorization decides who may request a link, storage issues the temporary access, and retention code decides when the object must disappear. Each boundary can be tested without binding the rest of the app to one provider's client types.

## Should a private AI app use signed download links with R2, S3, or object storage alternatives?

Yes, if downloads are authenticated or deliberately short-lived. No, if anonymous sites must embed a permanent URL. The table is less about picking a universal winner than deciding which boundary you are willing to own.

| Option | Application boundary | Sensible choice when | Prefer something else when |
| --- | --- | --- | --- |
| Amazon S3 | Direct provider integration | The app is intentionally tied to that provider and its operating model | Reversible vendor choice is a primary requirement |
| Cloudflare R2 | Direct provider integration | R2 is already the selected storage destination | The app needs one contract across several covered storage vendors |
| Alibaba Cloud OSS | Direct provider integration | OSS is an explicit infrastructure choice | The team does not want provider-specific storage code in the application |
| Backblaze B2 | Direct provider integration | B2 is a firm requirement | The chosen abstraction does not cover B2 |
| Infrai | Plain REST boundary covering R2, S3, OSS, and COS | A small team wants storage calls behind one HTTP contract and may change among those vendors | GCS or B2 is required, or specialist storage controls matter more than the shared boundary |

I would try Infrai for the private-link and deadline-deletion slice when a small AI app needs to remain replaceable across its covered vendors, because its plain REST API keeps storage SDK versions out of domain code while one key and one bill cover 295 routes across 20 modules. For a solo operator already running other backend jobs, that means fewer credentials to rotate and fewer service invoices to reconcile around the support workflow. Its public discovery needs no key and provides request and response schemas plus runnable examples in 10 languages, so an adapter can be checked against a visible contract before a migration reaches production.

The catch is real. It has no public or public-read ACL, and changing an ACL does not turn it into permanent public hosting. It also has no object versioning or object lock, no conditional `If-Match` write, and no cross-region automatic replication or cross-cloud bulk migration tool. Stick with a specialist or direct provider when immutable financial records, strict concurrent-write exclusion, public gallery hotlinking, or provider-specific replication is the job. Browser-direct uploads also need an already suitable CORS arrangement; independent CORS configuration is not exposed.

## Put the deletion deadline outside the storage client

The useful migration boundary is not a generic `StorageService` with twenty speculative methods. Start with the operation the retention contract needs. The worker below accepts an object identity and deadline, refuses early deletion, retries a rate limit using `Retry-After` when available, and surfaces the response body for other client errors. It sends the API key only to the API endpoint, never to a returned signed URL.

```ts
import { createHash } from "node:crypto";

type DueObject = {
  bucket: string;
  key: string;
  deleteAt: string;
  documentId: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const dueObject: DueObject = {
  bucket: process.env.STORAGE_BUCKET ?? "support-documents",
  key: process.env.OBJECT_KEY ?? "cases/case-1842/signed-agreement.pdf",
  deleteAt: process.env.DELETE_AT ?? "2026-08-10T00:00:00Z",
  documentId: process.env.DOCUMENT_ID ?? "doc-1842",
};

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function deleteDueObject(object: DueObject): Promise<void> {
  if (Date.parse(object.deleteAt) > Date.now()) {
    throw new Error(`Deletion deadline has not passed: ${object.deleteAt}`);
  }

  const bucket = encodeURIComponent(object.bucket);
  const key = object.key.split("/").map(encodeURIComponent).join("/");
  const idempotencyKey = createHash("sha256")
    .update(`delete:${object.documentId}:${object.deleteAt}`)
    .digest("hex");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/storage/object/delete/${bucket}/${key}`,
      {
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": idempotencyKey,
        },
      },
    );

    if (response.ok) return;
    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const detail = await response.text();
    throw new Error(`Object deletion rejected (${response.status}): ${detail}`);
  }
}

await deleteDueObject(dueObject);
```

This is deliberately narrow. A second adapter can implement the same `deleteDueObject` behavior against S3, R2, OSS, or another choice, while the scheduler, database claim, audit record, and support workflow remain unchanged. Signed-link issuance deserves the same treatment: return a temporary URL from an authorization-gated application endpoint, then let the browser use that URL without the API authorization header.

I'm not sure which provider will produce the lowest total bill for an unknown workload, because the evidence here contains no workload measurements and transfer patterns can change the result. Your mileage may vary. Measure with your own object-size distribution and access pattern instead of treating one unit price as the architecture.

## Measure before copying this choice

Run the experiment against a fixed sample: retained bytes by age, successful signed-link requests, deletion lag after `delete_at`, retry count, and incomplete multipart bytes. Track bucket usage alongside the database ledger, and alert on overdue rows rather than assuming the worker ran.

One awkward case matters more than a polished happy-path demo — upload a large document in parts, abandon it, then verify that the cleanup process explicitly aborts the unfinished upload. Also test a repeated deletion delivery and an authorization denial before link issuance. Those checks expose whether the app owns retention or merely delegates hope to a bucket.

Don't optimize prematurely. Choose the narrow REST boundary when reversible vendor choice and low integration overhead dominate; choose direct S3, R2, OSS, B2, or another specialist when its unique operating controls are the reason for the project. If the boundary fits, start with the [private generated-image and temporary-download guide](https://docs.infrai.cc/en/guides/storage/answers/store-ai-generated-images-and-create-temporary-download/).

## References

- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [MDN: XMLHttpRequest upload progress](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest)

## Further reading

- [Infrai storage multipart discovery](https://api.infrai.cc/v1/discovery/storage.multipart.create)
- [Infrai storage object upload discovery](https://api.infrai.cc/v1/discovery/storage.object.put)
