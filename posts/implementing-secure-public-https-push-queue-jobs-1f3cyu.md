# Implementing Secure Public HTTPS Push Queue Jobs: Background Worker Recovery in 2026

Short answer: make a queued job recoverable before making the worker fast. For nightly logistics reconciliation, a public HTTPS receiver should authenticate the original request bytes, reject stale deliveries, validate a narrow envelope, and durably record a stable job ID before returning success. The reconciliation process can then resume from committed payment-provider cursors without trusting one HTTP attempt to finish the work.

The data flow is deliberately plain. A nightly scheduler creates one job for a business date; a push queue delivers it to a small Node.js receiver; the receiver writes an inbox record; and a separate worker compares provider settlements with shipment charges. Cron is a time-based scheduler, so it can start the flow, but it cannot by itself prove that page three of a reconciliation finished.

That proof is the design problem.

## Govern retention with a reconciliation control ledger

Start with the question an operator will ask after a process exits: "What can run again without changing a result twice?" A useful answer needs more than a `done` flag. Give every delivery a stable `job_id`, keep the explicit `businessDate`, store the next provider cursor beside committed comparison results, and lease work for a bounded period. The durable states can stay small: `pending`, `running`, `succeeded`, and `failed`.

The acceptance boundary matters. A successful HTTP response means only that the authenticated envelope is now durable and owned by the application. It does not mean that settlement reconciliation succeeded. Keeping those meanings separate lets the receiver remain quick while a worker handles pagination, provider pacing, retries, and operator review on its own schedule.

Consider a provider settlement for `2026-08-19` with three pages. The worker commits page one and its next cursor, claims page two, writes comparison rows, and exits before advancing the cursor. When its lease expires, another worker will see page two again. Each comparison therefore needs a stable key such as `(job_id, provider_transaction_id, shipment_id)`, and the page results must commit with the cursor. Repeating the page becomes an update or a no-op instead of another discrepancy. A longer HTTP timeout would only postpone this ambiguity; it would not remove it.

Page two is the test.

This recovery-first model also sets a practical cost boundary. Store identifiers, transitions, attempts, cursor position, and compact error classifications. Don't copy full settlement payloads into every log line: that increases retained data and complicates deletion requests. GDPR Article 17 establishes a right to erasure under specified conditions, so retention and deletion paths belong in the design rather than in a later cleanup project.

## What should a secure Node.js public HTTPS push queue background worker implement?

Use one narrow `POST` route. Authenticate the exact raw body before JSON parsing, bind the signature to a delivery timestamp, reject requests outside the agreed replay window, validate the job schema, and insert under a unique queue job ID. Express and Fastify can both host this boundary. The important framework detail is preserving the unmodified request bytes until verification completes.

The following Express example is runnable once `express`, `better-sqlite3`, and their TypeScript types are installed. It uses a compact shared-secret delivery contract so the security boundary is visible. A real publisher may specify another header layout or an asymmetric scheme; use that documented contract exactly rather than assuming these sample header names.

```ts
import crypto from "node:crypto";
import express from "express";
import Database from "better-sqlite3";

type ReconciliationJob = {
  businessDate: string;
  provider: "primary-payment-provider";
  settlementCursor: string;
};

const app = express();
const db = new Database("reconciliation-inbox.db");
const signingSecret = process.env.QUEUE_SIGNING_SECRET;

if (!signingSecret) throw new Error("QUEUE_SIGNING_SECRET is required");

db.pragma("journal_mode = WAL");
db.exec(`
  CREATE TABLE IF NOT EXISTS reconciliation_inbox (
    job_id TEXT PRIMARY KEY,
    payload TEXT NOT NULL,
    state TEXT NOT NULL CHECK (state IN ('pending', 'running', 'succeeded', 'failed')),
    attempts INTEGER NOT NULL DEFAULT 0,
    available_at TEXT NOT NULL,
    lease_until TEXT,
    last_error TEXT,
    created_at TEXT NOT NULL
  )
`);

const insertJob = db.prepare(`
  INSERT OR IGNORE INTO reconciliation_inbox
    (job_id, payload, state, available_at, created_at)
  VALUES (?, ?, 'pending', ?, ?)
`);

function parseJob(raw: Buffer): ReconciliationJob {
  const value: unknown = JSON.parse(raw.toString("utf8"));
  if (!value || typeof value !== "object") throw new Error("invalid envelope");

  const job = value as Record<string, unknown>;
  const validDate =
    typeof job.businessDate === "string" &&
    /^\d{4}-\d{2}-\d{2}$/.test(job.businessDate);
  const validCursor =
    typeof job.settlementCursor === "string" &&
    job.settlementCursor.length > 0 &&
    job.settlementCursor.length <= 200;

  if (!validDate || !validCursor || job.provider !== "primary-payment-provider") {
    throw new Error("invalid reconciliation fields");
  }
  return job as ReconciliationJob;
}

app.post(
  "/queue/reconciliation",
  express.raw({ type: "application/json", limit: "32kb" }),
  (request, response) => {
    const jobId = request.header("x-queue-job-id") ?? "";
    const sentAt = request.header("x-queue-sent-at") ?? "";
    const suppliedHex = request.header("x-queue-signature") ?? "";
    const raw = request.body as Buffer;
    const sentAtMs = Date.parse(sentAt);

    if (
      !jobId ||
      !Number.isFinite(sentAtMs) ||
      Math.abs(Date.now() - sentAtMs) > 300_000
    ) {
      response.sendStatus(401);
      return;
    }

    const expected = crypto
      .createHmac("sha256", signingSecret)
      .update(sentAt)
      .update(".")
      .update(raw)
      .digest();
    const supplied = Buffer.from(suppliedHex, "hex");

    if (
      supplied.length !== expected.length ||
      !crypto.timingSafeEqual(supplied, expected)
    ) {
      response.sendStatus(401);
      return;
    }

    try {
      const job = parseJob(raw);
      const now = new Date().toISOString();
      insertJob.run(jobId, JSON.stringify(job), now, now);
      response.sendStatus(204);
    } catch {
      response.sendStatus(400);
    }
  }
);

app.listen(8080, "0.0.0.0");
```

`INSERT OR IGNORE` makes repeated authenticated delivery boring: the same ID still produces `204`, while the primary key prevents a second inbox item. Don't derive that ID from connection time. It must survive every delivery attempt.

I'm not sure a five-minute replay window is right for every deployment; publisher retry timing, clock controls, and the threat model decide that. The invariant is clearer than the number — authenticate before parsing, cover the unchanged body, and enforce a documented freshness rule. Fastify users should implement the same contract with raw-body access rather than verifying a reserialized object.

## Test the page-two recovery contract

A handler test that receives one `204` proves intake, not recovery. The higher-value drill repeats the same signed envelope and expects one inbox row, changes one signed byte and expects `401`, sends an invalid business envelope and expects `400`, and then simulates a worker exit between page output and cursor advancement. Use an injected clock for timestamp and lease tests so the `300,000` ms edge is exact; real sleeps make this suite slow and hard to reproduce.

The delivery and business retry loops should remain distinct. Delivery retry ends when the envelope is durable. Business retry begins when a worker leases that inbox row, and it owns provider calls, bounded backoff, cursor advancement, and the final state. If an attempt budget is exhausted, retain the row as `failed` for deliberate review rather than retrying forever.

The drill should exercise these observations in order:

1. Two identical authenticated requests leave one `pending` row.
2. A worker claim changes that row to `running`, increments `attempts`, and sets `lease_until` atomically.
3. Page output and the next cursor commit together.
4. An expired lease permits another worker to reclaim the same job.
5. Replaying the interrupted page creates no duplicate comparison record.
6. An operator can move corrected work through an explicit, audited replay path.

Midnight deserves its own case. The envelope's `businessDate`, not the server's local clock, should select the ledger window. Run tests on both sides of midnight in the business timezone and verify that retries keep the original date. It's a small field with a large blast radius.

## Budget the public ingress operating surface

Public push adds work that doesn't appear in the handler: credential rotation, replay controls, ingress monitoring, synthetic delivery, and failed-job review. For a solo team, those are recurring costs in attention even when request volume is low. Deploy the receiver and worker as independently stoppable processes even if they share a repository. Give the receiver only the database access needed to record inbox work; the worker gets the permissions needed to lease jobs and write reconciliation results.

Logs need restraint too. Watch the age of the oldest unfinished business date, attempt counts, lease age, and response-code totals. Queue depth alone cannot distinguish a poisoned job from a healthy burst, while full settlement payloads create storage and deletion work without improving the first recovery decision. A compact cursor trail and stable identifiers usually answer the useful question: where can processing resume?

Push is a poor bargain when nobody owns failed-job review. A queue can preserve attempts, but it cannot decide whether a discrepancy represents a delayed payment, a duplicate shipment charge, or incorrect source data.

Ownership costs more than polling.

## Choose delivery from the runbook

The catch is that push delivery requires an Internet-reachable HTTPS receiver. It is not suitable when policy forbids public ingress even behind authentication. In that environment, stick with a queue workers can pull through approved egress or a private endpoint. For one noncritical task on one machine, cron may remain the smaller system if missed-run evidence and durable state exist elsewhere.

| Choice | Useful when | Operational cost to accept |
| --- | --- | --- |
| Public HTTPS push | Workers should receive jobs without polling | Public ingress, identity verification, and key rotation |
| Worker pull | Public inbound traffic is prohibited | Polling lifecycle, worker credentials, and visibility into idle consumers |
| Local cron | One host runs a low-criticality task | Separate missed-run detection and durable progress tracking |

Express versus Fastify is secondary. Pick the one already operated by the team, then prove raw-body verification, maximum body size, response codes, and shutdown behavior in that runtime. Switching routers won't repair a missing idempotency key.

Before enabling the nightly trigger, the runbook should confirm that one business date maps to one stable job identity, acceptance happens only after a durable insert, expired leases can be reclaimed, each provider page has a committed cursor, and irreconcilable work has an owner. After deployment, send a signed synthetic job against a test account, stop the worker during a middle page, let the lease expire, and verify the final comparison counts and cursor trail. During recovery, pause new claims before editing state, preserve the original job ID, and record manual changes.

Keep it recoverable.

## References

- https://en.wikipedia.org/wiki/Cron
- https://gdpr-info.eu/art-17-gdpr/
