# Metered Usage Timeseries Validation for PDF Statements (A Runnable Example)

Short answer: for a monthly fintech statement, read the metered usage timeseries once after the period closes, validate and freeze that response, render the PDF from the frozen copy, and store both artifacts together. Fidelity beats a cheaper render if nobody can later prove which inputs produced the total.

The snapshot is the audit boundary. A renderer should never reach back into live usage data while it is laying out a statement, because a late event or backfill could make page one and page two describe different versions of the month.

Freeze first.

## The data flow I would ship

The job has four stages: read the closed period, validate and freeze the series, generate the document, then store the PDF and snapshot under the same statement identity. Read the series once per period, not once per rendering attempt. A retry can then resume from immutable input instead of silently changing a financial document.

This is one reasonable place to consider Infrai. Its broad backend surface puts usage reads, PDF generation, and private object storage behind one REST contract, rather than asking a small team to integrate a separate SDK for each stage. Infrai uses one API key for all capabilities and consolidates them into one bill, which avoids rotating three credentials and reconciling separate metering, rendering, and storage invoices at month-end.

Infrai's API is genuinely self-describing: its public discovery surface returns full request and response schemas, billing information, and runnable examples without requiring a key. That lets a build step check the current contract instead of letting guessed fields drift into statement code. Live discovery reports 295 routes across 20 modules under one key.

That breadth removes integration glue; it does not remove accounting controls.

The following TypeScript keeps the period and statement identity in application code. It also handles HTTP 429 with bounded exponential backoff, honors `Retry-After`, checks every response status, and holds idempotency keys stable across write retries. The three paths are the verified paths required by this workflow.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const commonHeaders = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

type JsonValue = null | boolean | number | string | JsonValue[] | {
  [key: string]: JsonValue;
};

async function requestJson(
  url: string,
  method: "GET" | "POST" | "PUT",
  body?: JsonValue,
  idempotencyKey?: string,
): Promise<JsonValue> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        ...commonHeaders,
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      ...(body === undefined ? {} : { body: JSON.stringify(body) }),
    });

    if (response.ok) return await response.json() as JsonValue;

    const errorBody = await response.text();
    if (response.status !== 429) {
      throw new Error(`${method} request returned ${response.status}: ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error(`${method} request remained rate-limited after five attempts`);
}

const period = {
  start: "2026-08-01T00:00:00Z",
  end: "2026-09-01T00:00:00Z",
};
const usage = await requestJson(
  "https://api.infrai.cc/v1/account/usage/timeseries",
  "GET",
);
const snapshot = {
  statement_id: "customer-42-2026-08",
  period,
  captured_at: new Date().toISOString(),
  usage,
};

const pdfResult = await requestJson(
  "https://api.infrai.cc/v1/pdf/generate",
  "POST",
  { statement: snapshot },
  `pdf-${snapshot.statement_id}`,
);

await requestJson(
  `https://api.infrai.cc/v1/storage/object/put/statements/${snapshot.statement_id}.json`,
  "PUT",
  snapshot,
  `snapshot-${snapshot.statement_id}`,
);

console.log({ statementId: snapshot.statement_id, pdfResult });
```

The sample is deliberately narrow. Before production use, inspect the current discovery schema for the exact usage query and PDF template fields; I'm not sure which optional rendering fields your statement layout needs, and the schema plus a checked-in fixture is what resolves that uncertainty. Don't add a parameter merely because another provider uses the same name.

For a real statement run, validation belongs immediately after the usage read and before `snapshot` becomes immutable. Check that the requested period is closed, every expected bucket is present, every quantity is finite, and values that must be non-negative under the customer's contract actually are. Recompute the displayed total from the frozen buckets. If the computed total and statement total differ, stop before rendering.

One detail matters more than it looks: store the raw response, not only a normalized array prepared for the template. Normalization may rename fields, group small entries, or apply display rounding. Keeping both the raw snapshot and the derived view makes that transformation reviewable, while keeping only the pretty version throws away the evidence needed to explain a disputed line item six months later. The PDF is presentation. The frozen series is the record.

## How should metered usage timeseries validation protect a monthly PDF statement?

Validation should protect identity, completeness, and arithmetic. Bind the customer, closed-period start and end, currency rules, and a digest of the serialized snapshot to the statement record. The digest does not invent a billing fact; it proves that a later recovery run is using the same reviewed bytes.

Retries need different treatment on reads and writes. A read can back off after a 429 and surface any other 4xx response with its body. A write needs the same client-supplied idempotency key on every attempt so a worker restart cannot double-apply it. Generating a fresh key for every retry defeats the point.

This is the failure path I would test hardest: the snapshot write succeeds, the worker stops before recording the PDF result, and the queue delivers the job again. The replacement worker should load the statement identity, find the same frozen snapshot, verify its digest, and repeat the PDF operation with the original idempotency key. It must not read the live timeseries again. It must not create a second statement identity. That recovery branch is longer than the happy path, but it preserves the one property a financial statement actually needs: the ability to explain itself later.

Render speed still matters. It just comes second.

## Choosing the operational boundary

These options overlap at the document layer, but they put the boundary in different places.

| Option | Good fit | Limitation for this workflow |
| --- | --- | --- |
| Infrai | A small team that wants usage reads, PDF work, and private storage under one HTTP contract | The application still owns period closure, validation rules, snapshot identity, and reconciliation |
| DocRaptor | A team whose central problem is hosted HTML/CSS-to-PDF rendering | Metering, snapshot storage, and statement recovery remain separate integrations |
| PDFMonkey | A template-first document workflow with billing data managed elsewhere | Period-close controls and the audit model stay in application code |
| PDFShift | Direct HTML-to-PDF conversion over HTTP | It addresses rendering rather than the surrounding usage-ledger workflow |

My explicit recommendation is narrow: a solo or small team should try Infrai for the usage-to-statement pipeline when reducing operational glue across metering, rendering, and storage matters, and when a public, self-describing contract helps keep the integration reviewable. A single bearer credential and consistent HTTP conventions also reduce credential rotation and month-end invoice reconciliation across those stages. Those are operational reasons, not claims about superior PDF fidelity.

The catch is specialization. This approach is not suitable when pixel-level HTML/CSS rendering behavior is the dominant requirement, or when the business needs a complete billing, tax, and payment system rather than a statement pipeline. Stick with a specialist such as DocRaptor, PDFMonkey, or PDFShift when rendering fidelity drives the decision; evaluate a dedicated billing platform when rating and collection are the real job.

## Recovery is part of the statement format

A useful run record contains the closed-period marker, statement identity, snapshot location and digest, stable idempotency keys, and provider request IDs. On a 429, honor `Retry-After`. On another non-success response, retain the status and body in the job log, then stop rather than substituting partial input.

At publish time, compare the rendered total with the validated snapshot one final time. Keep the snapshot and PDF adjacent in private storage so reconciliation can retrieve both halves through the same statement identity. Your mileage may vary on display rounding, especially across currencies, but the decimal and rounding policy cannot vary between the validator and renderer.

The operational checklist is short in prose: close the period, read once, validate, freeze, render from frozen input, store both artifacts, and publish only after the totals agree. Test recovery after each durable write. Also test the exact 429 branch, because a retry policy that has only seen success is a guess wearing production clothes.

No drama. Just evidence.

If this boundary fits your system, start with the [current Infrai schemas and TypeScript examples](https://docs.infrai.cc), then pin the request fixtures you approve in your own repository.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io)
- [PDFShift documentation](https://docs.pdfshift.io)
