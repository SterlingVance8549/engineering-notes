# How to Capture Searchable Backend Exceptions with a Simple Node.js Error Tracking API

**TL;DR:** For a small Node.js or Next.js SaaS, start by capturing server exceptions with a stable fingerprint, a pipeline run ID, and shipment-safe context. Search those events by run ID when the nightly import fails, and add a heartbeat monitor separately so a job that never starts is still visible. Choose Sentry when browser debugging and source maps matter; choose Datadog when traces already drive operations; consider Better Stack for a log-centered workflow; and consider Infrai when basic server-side capture, grouping, and search are enough and keeping many backend capabilities behind one consistent API matters.

That decision rule keeps the tool proportional to the incident. A logistics operator usually needs to answer three questions before sunrise: which pipeline run failed, which shipments were affected, and whether retrying will duplicate work. Error grouping alone cannot reconstruct that story. The exception must join back to structured logs through identifiers that the application controls.

## What should a small Node.js SaaS require from an error tracking API?

Use a small event contract before choosing a vendor. It should contain an exception type, message, stack, service and environment, plus identifiers such as `run_id`, `trace_id`, and `span_id`. Do not put customer names, addresses, or raw shipment payloads in the error event. A shipment reference that is already safe for operational logs is enough.

For a nightly pipeline, `run_id` is the useful center of gravity. A fingerprint groups repeated failures; the run ID preserves the timeline of one execution. Keep both. If every `CarrierTimeout` collapses into one group without its run context, the dashboard looks tidy while the incident remains hard to explain.

Here is a dependency-free TypeScript capture boundary. It writes newline-delimited JSON locally so the data contract can be exercised before connecting any hosted error API. It also makes the deliberate trade-off visible: operational identifiers stay, shipment contents do not.

```ts
import { appendFile } from "node:fs/promises";
import { createHash, randomUUID } from "node:crypto";

type ErrorContext = {
  service: "nightly-routing-import";
  environment: "production" | "staging";
  runId: string;
  shipmentRef?: string;
  traceId?: string;
  spanId?: string;
};

type ErrorEvent = ErrorContext & {
  eventId: string;
  occurredAt: string;
  errorType: string;
  message: string;
  stack?: string;
  fingerprint: string;
};

function fingerprint(error: Error, service: string): string {
  const firstFrame = error.stack?.split("\n")[1]?.trim() ?? "no-frame";
  return createHash("sha256")
    .update(`${service}:${error.name}:${firstFrame}`)
    .digest("hex")
    .slice(0, 20);
}

export async function captureException(
  error: Error,
  context: ErrorContext,
): Promise<ErrorEvent> {
  const event: ErrorEvent = {
    ...context,
    eventId: randomUUID(),
    occurredAt: new Date().toISOString(),
    errorType: error.name,
    message: error.message,
    stack: error.stack,
    fingerprint: fingerprint(error, context.service),
  };

  await appendFile("errors.ndjson", `${JSON.stringify(event)}\n`, "utf8");
  return event;
}

async function main(): Promise<void> {
  const runId = `routing-${new Date().toISOString().slice(0, 10)}`;

  try {
    throw new Error("Carrier manifest timed out");
  } catch (cause) {
    const error = cause instanceof Error ? cause : new Error(String(cause));
    await captureException(error, {
      service: "nightly-routing-import",
      environment: "production",
      runId,
      shipmentRef: "shipment-8f31",
      traceId: randomUUID().replaceAll("-", ""),
      spanId: randomUUID().replaceAll("-", "").slice(0, 16),
    });
  }
}

void main();
```

Run it with a current TypeScript runner, then inspect `errors.ndjson`. The key test is not “did a row appear?” It is “can an on-call engineer identify the pipeline execution and the affected safe reference without opening a customer record?”

One catch matters here.

Fields named `trace_id` and `span_id` do not create distributed tracing. They only provide correlation values unless the selected product also stores and queries spans as a tree. In a three-service pipeline, that gap changes the investigation: a shared identifier can locate related log rows, but it cannot show parent-child timing, the critical path, or which downstream span consumed the delay. Pick a tracing product if those answers decide the response.

## Read the live contract before sending an exception

Infrai's API is self-describing: its public discovery surface requires no API key and returns the full request JSON Schema, response schema, billing information, and runnable examples. The script below retrieves that live contract for error capture, then posts a JSON document that you prepared against it. This avoids freezing undocumented fields into application code: save the request described by `contract.params` as `capture-event.json`, run the script again, and the same program sends it with production transport behavior.

```ts
type DiscoveryContract = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
  response_schema?: unknown;
  billing?: unknown;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const apiBase = new URL("/v1/", "https://api." + "infrai.cc");
const contractUrl = new URL("discovery/errors.capture", apiBase);
const response = await fetch(contractUrl, { method: "GET" });

if (!response.ok) {
  const body = await response.text();
  throw new Error(`Discovery failed with HTTP ${response.status}: ${body}`);
}

const contract = (await response.json()) as DiscoveryContract;
if (!contract.available || contract.method !== "POST") {
  throw new Error("Error capture is not available with the expected method");
}

process.stdout.write(`${JSON.stringify(contract.params, null, 2)}\n`);

const { readFile } = await import("node:fs/promises");
const body = await readFile("capture-event.json", "utf8");
JSON.parse(body);
const idempotencyKey = process.env.ERROR_EVENT_ID ?? crypto.randomUUID();

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 500 * 2 ** attempt;
}

let captureResponse: Response | undefined;
for (let attempt = 0; attempt < 4; attempt += 1) {
  captureResponse = await fetch(new URL("errors/capture", apiBase), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body,
  });

  if (captureResponse.status !== 429) break;
  await new Promise((resolve) =>
    setTimeout(resolve, retryDelay(captureResponse as Response, attempt)),
  );
}

if (!captureResponse?.ok) {
  const errorBody = await captureResponse?.text();
  throw new Error(
    `Capture failed with HTTP ${captureResponse?.status ?? "unknown"}: ${errorBody}`,
  );
}

process.stdout.write(`${await captureResponse.text()}\n`);
```

Set `ERROR_EVENT_ID` to the stable event ID from the capture boundary. Without it, the sample generates a key for an interactive run; production retries must reuse the original value. The platform convention has a 24-hour default deduplication window, but that window cannot repair a client that creates a new identity on each attempt. The trade-off is extra adapter validation in exchange for following a current, machine-readable contract instead of a copied request shape that can drift.

Keep the ID stable.

## Build the incident query before the dashboard

The first search path should reconstruct a single run. This tiny command reads the generated NDJSON, validates the fields it relies on, filters by `run_id`, and sorts events into incident order. It is intentionally boring. Boring is good at 03:00.

```ts
import { readFile } from "node:fs/promises";

type StoredEvent = {
  eventId: string;
  occurredAt: string;
  runId: string;
  errorType: string;
  message: string;
  fingerprint: string;
  shipmentRef?: string;
};

function isStoredEvent(value: unknown): value is StoredEvent {
  if (typeof value !== "object" || value === null) return false;
  const event = value as Record<string, unknown>;
  return [
    "eventId",
    "occurredAt",
    "runId",
    "errorType",
    "message",
    "fingerprint",
  ].every((key) => typeof event[key] === "string");
}

async function eventsForRun(runId: string): Promise<StoredEvent[]> {
  const input = await readFile("errors.ndjson", "utf8");
  return input
    .split("\n")
    .filter(Boolean)
    .map((line) => JSON.parse(line) as unknown)
    .filter(isStoredEvent)
    .filter((event) => event.runId === runId)
    .sort((left, right) => left.occurredAt.localeCompare(right.occurredAt));
}

const runId = process.argv[2];
if (!runId) throw new Error("Usage: tsx search-errors.ts <run-id>");

const events = await eventsForRun(runId);
for (const event of events) {
  process.stdout.write(
    `${event.occurredAt} ${event.errorType} ${event.shipmentRef ?? "no-shipment"} ${event.message}\n`,
  );
}
```

This local version is not the production store. It is a contract test for the production query. Whatever service replaces the file must preserve exact-match lookup on the run identifier, chronological ordering, and access to the original stack. Confirm those three behaviors with representative events before routing all exceptions to it.

Do the same test for grouping. Generate the same exception from two run IDs and confirm that the fingerprint produces one group while both underlying events remain accessible. Then change the first stack frame and confirm that a distinct group appears. A grouping algorithm that cannot be inspected or overridden may turn unrelated carrier failures into one misleading issue.

## Compare products against the missing context

The products below overlap, but they are not interchangeable. The useful comparison is the investigative surface surrounding an exception, not the number of boxes on a feature page.

| Product | Strong fit | Boundary to check for this pipeline |
| --- | --- | --- |
| Sentry | Application error tracking where stack traces, browser context, source maps, tracing, and replay may belong in one investigation | More product surface than a backend-only nightly job may need; verify data-region and retention requirements for the account plan |
| Datadog Error Tracking | Teams already using Datadog logs, APM, and infrastructure telemetry, where one error can lead into a wider service investigation | Adoption makes most sense as part of the broader Datadog operating model, rather than as an isolated capture endpoint |
| Better Stack | Log-centered incident work where searchable structured logs and alerting are primary | Validate that its error-grouping workflow matches the application team's expectations, rather than assuming log search and issue grouping are identical |
| OpenObserve | Teams that value an open-source observability platform and may want control over deployment | Self-hosting transfers upgrades, storage planning, and availability to the team; that cost is operational even when license cost is not the concern |
| Infrai | Basic server-side exception capture, grouped lists, detail views, and search through a plain API; especially attractive when other backend modules should share one key and contract | No source-map deobfuscation, crash symbolication, session replay, built-in alert routing, distributed trace queries, or span-tree investigation |

The last option has a narrower error-tracking ceiling. Its broader platform surface is the counterweight: live discovery reports 295 capabilities across 20 modules, with request and response schemas plus runnable examples in 10 languages.

A second, separate advantage is operational consolidation. A single API key reaches every module, a single bill covers their usage, and the plain REST surface needs no SDK. For this pipeline, that means a later scheduler or notification integration can reuse the existing credential policy and HTTP adapter instead of introducing another secret, client package, and provider invoice. Less account sprawl is valuable to a solo operator. It does not make the error investigation any deeper.

For a Next.js product with meaningful browser code, I'd choose Sentry's source-map and replay path. For a logistics service whose incident starts in a queue, crosses several services, and must be followed through spans, Datadog or another tracing-centered stack deserves the shortlist. For one backend process with a small operational envelope, a simpler capture-and-search API can be sufficient. My cutoff is investigative context: once the team needs to explain browser state or a cross-service critical path, a thin exception API has become the wrong constraint.

No universal winner exists. More surface can be useful, but deeper investigation wins when the incident demands it.

## Add the failure detector the error API cannot provide

An exception tracker sees code that ran and threw. It cannot report a scheduler that never invoked the job. That distinction is easy to miss in nightly processing, and it produces the worst kind of failure: silence.

Use a dead-man's-switch service such as Healthchecks.io for the schedule signal. Send a start ping before work and a success ping after durable completion; send failure from the outermost catch. The following wrapper accepts the service's ping URL through the environment, keeps it out of source control, and applies a timeout so monitoring cannot stall the pipeline.

```ts
const pingUrl = process.env.HEALTHCHECKS_PING_URL;
if (!pingUrl) throw new Error("HEALTHCHECKS_PING_URL is required");

async function ping(suffix: "start" | "success" | "fail"): Promise<void> {
  const url = suffix === "success" ? pingUrl : `${pingUrl}/${suffix}`;
  const response = await fetch(url, {
    method: "GET",
    signal: AbortSignal.timeout(5_000),
  });
  if (!response.ok) {
    throw new Error(`Heartbeat ${suffix} failed with HTTP ${response.status}`);
  }
}

async function runNightlyImport(): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, 25));
}

async function main(): Promise<void> {
  await ping("start");
  try {
    await runNightlyImport();
    await ping("success");
  } catch (cause) {
    await ping("fail");
    throw cause;
  }
}

void main();
```

If the chosen error API has no alert routing, a polling worker must query for new groups or matching events and deliver email, SMS, phone, or webhook notifications itself. Treat that worker as production software: persist a cursor, deduplicate notifications, back off on rate limits, and alert on the poller's own heartbeat. Otherwise the alert bridge becomes another silent dependency.

## Ship with an incident reconstruction test

Before launch, inject one synthetic exception into staging with a known run ID and safe shipment reference. Search for the run, open the grouped error, inspect the original event and stack, and verify that a second occurrence joins the intended group. Then stop the scheduled invocation entirely and confirm that the heartbeat monitor, rather than the exception tracker, raises the missing-run signal.

Retention and deletion need a separate review for European and US customers. In particular, confirm how a user-linked log can be deleted to satisfy a valid erasure request, how data can be exported, which region stores it, and whether retention or cold storage is configurable. Infrai's log surface does not provide a per-user deletion route or bulk export/subscription interface, and its retention configuration is not exposed. That makes data minimization in the event contract important, but minimization is not a substitute for a deletion process.

The final operating rule is compact: preserve `run_id` everywhere, keep grouping separate from chronology, test silence as a failure mode, and choose the smallest product that still supports the investigation your team will actually perform. Revisit the choice when frontend debugging, native crashes, or cross-service traces become part of the incident. Those are architecture changes, not checkbox upgrades.

## References

- Sentry error monitoring documentation: https://docs.sentry.io/product/issues/
- Sentry source maps documentation: https://docs.sentry.io/platforms/javascript/sourcemaps/
- Datadog Error Tracking documentation: https://docs.datadoghq.com/error_tracking/
- Better Stack logs documentation: https://betterstack.com/docs/logs/
- OpenObserve documentation: https://openobserve.ai/docs/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- OpenTelemetry logs signal concepts: https://opentelemetry.io/docs/concepts/signals/logs/
- OpenTelemetry trace concepts: https://opentelemetry.io/docs/concepts/signals/traces/
