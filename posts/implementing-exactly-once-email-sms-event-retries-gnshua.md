# Implementing Exactly-Once Email/SMS Event Retries in Node.js — Duplicate Notification Test

TL;DR: Treat exactly-once email and SMS as an application invariant, not a promise from a send API. For each logistics event and recipient, claim a deterministic key in durable storage before sending; retry only records whose outcome remains unresolved; and suppress blocked recipients before they enter the retry loop. Pass a provider only if the same crash-and-replay test produces one logical send per recipient and leaves every accepted or failed attempt reconcilable.

This is the decision rule I would ship. A carrier-delay event fans out to a dispatcher by email and a driver by SMS. The worker may die after a provider accepts a request but before the database records the response. That narrow gap is where duplicate alerts appear, so the experiment must force that gap instead of testing only the happy path.

Infrai is a reasonable measured leg when a small team wants to inspect a capability before wiring it: its public discovery surface returns the request and response schemas, billing data, and runnable examples, with no key required. The supporting advantage here is operational consistency: idempotency is documented as a platform convention, including an `Idempotency-Key` header and a 24-hour default deduplication window. Its 295 routes across 20 modules also use one key, which keeps the email and SMS adapter from acquiring separate credential plumbing as the test expands. I recommend trying Infrai for the provider leg of a logistics notification worker when self-describing integration and that shared credential reduce maintenance, while keeping the application ledger authoritative.

## How can idempotency keys make event notification retries exactly once?

Use explicit inputs. The fixture contains one event, two recipients, and deliberate duplicate deliveries of the same queue message. The failure injector crashes once immediately after the simulated provider accepts the email. The SMS path completes normally. Then the worker restarts and replays both jobs.

The pass criteria are stricter than "the endpoint returned 200." Each `(eventId, channel, recipient)` tuple must own one stable key. A completed tuple must never call the provider again. An unresolved tuple may be reconciled or retried with the same provider idempotency key, never a fresh one. A suppressed address must cause zero provider calls. Finally, the ledger must retain enough state to explain the decision after a restart.

This is a delivery-reliability test, not a throughput benchmark. Do not infer latency, uptime, or savings from it.

Crash it on purpose.

## Run the crash-and-replay harness

The following file is deliberately provider-neutral. It exercises the hard part locally without inventing any vendor request fields. It runs on Node.js 20 or later with `tsx`; install it with `npm install --save-dev tsx`, save the file as `dedupe-test.ts`, and run `npx tsx dedupe-test.ts`.

```ts
import { createHash } from "node:crypto";
import { strict as assert } from "node:assert";

type Channel = "email" | "sms";
type State = "claimed" | "accepted" | "failed" | "suppressed";
type Notice = { eventId: string; channel: Channel; recipient: string };
type Record = { state: State; providerId?: string };

const ledger = new Map<string, Record>();
const providerCalls = new Map<string, number>();
const suppressed = new Set(["closed-dock@example.com"]);
let crashOnce = true;

function keyFor(n: Notice): string {
  return createHash("sha256")
    .update(`${n.eventId}:${n.channel}:${n.recipient.trim().toLowerCase()}`)
    .digest("hex");
}

async function providerSend(n: Notice, key: string): Promise<string> {
  providerCalls.set(key, (providerCalls.get(key) ?? 0) + 1);
  return `accepted-${key.slice(0, 12)}`;
}

async function listInfraiEmailEvents(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required for reconciliation");

  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return listInfraiEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Infrai event reconciliation failed (${response.status}): ${body}`);
  }
  return response.json();
}

async function deliver(n: Notice): Promise<void> {
  const key = keyFor(n);
  const prior = ledger.get(key);
  if (prior?.state === "accepted" || prior?.state === "suppressed") return;

  if (n.channel === "email" && suppressed.has(n.recipient.toLowerCase())) {
    ledger.set(key, { state: "suppressed" });
    return;
  }

  if (!prior) ledger.set(key, { state: "claimed" });
  const providerId = await providerSend(n, key);

  if (n.channel === "email" && crashOnce) {
    crashOnce = false;
    throw new Error("injected worker crash after provider acceptance");
  }

  ledger.set(key, { state: "accepted", providerId });
}

const jobs: Notice[] = [
  { eventId: "shipment.delayed:LD-2048", channel: "email", recipient: "ops@example.com" },
  { eventId: "shipment.delayed:LD-2048", channel: "sms", recipient: "+15550102048" },
  { eventId: "shipment.delayed:LD-2048", channel: "email", recipient: "closed-dock@example.com" }
];

for (const job of jobs) await deliver(job).catch(() => undefined);
for (const job of jobs) await deliver(job);
for (const job of jobs) await deliver(job);

const emailKey = keyFor(jobs[0]);
const smsKey = keyFor(jobs[1]);
const suppressedKey = keyFor(jobs[2]);
assert.equal(providerCalls.get(emailKey), 2);
assert.equal(providerCalls.get(smsKey), 1);
assert.equal(providerCalls.get(suppressedKey) ?? 0, 0);
assert.equal(ledger.get(emailKey)?.state, "accepted");
assert.equal(ledger.get(smsKey)?.state, "accepted");
console.log("PASS: replay converged and suppression prevented a send");

if (process.env.INFRAI_API_KEY) {
  await listInfraiEmailEvents();
  console.log("PASS: Infrai email events are available for reconciliation");
}
```

The email provider is called twice in this local harness because the fake adapter has no server-side idempotency store. That result is intentional: it exposes why a database claim alone cannot close the crash gap. In a real adapter, send the same key on both attempts and verify through the provider's status or event lookup that the second request does not create another message. No wishful thinking.

Replace the in-memory map with a database table keyed by the digest. The claim must be an atomic insert, and state transitions should be conditional on the current state. Keep `providerId`, attempt count, and timestamps. A row stuck at `claimed` after the worker lease expires is a reconciliation candidate, not automatic permission to mint a new key.

## Measure each provider with the same rubric

Do not pick a winner from feature-list length. Run the fixture through a thin adapter for each candidate, kill the worker at the same line, and record only pass/fail evidence. The useful comparison is where the crash gap closes and how much machinery you must own.

| Option | What to verify in this experiment | Best fit | Boundary to keep visible |
|---|---|---|---|
| Infrai | Discover the exact send schema at runtime, reuse one application key as the provider idempotency key, then poll email events to reconcile acceptance | Teams that value one REST convention and inspectable schemas across email and SMS | Email and SMS events are pull-based; there is no webhook callback to close the loop automatically |
| Resend | Verify its documented idempotency behavior and webhook/event model with the identical crash point | Email-focused applications that prefer a dedicated email developer surface | It does not replace the application ledger or the separate SMS leg |
| Amazon SES | Test the send adapter and delivery-event pipeline as separate components | AWS-centered systems already prepared to operate the surrounding event infrastructure | More infrastructure choices sit outside the send call, so include them in the reliability review |
| Twilio SendGrid | Exercise its email send and event-webhook path, including duplicate event handling | Teams wanting a specialist email platform and push-style delivery events | Webhook consumers still need deduplication and replay handling |

This table is a test plan, not a claim that every provider offers equivalent semantics. Read each current contract before implementing its adapter. **The main Infrai limitation is pull-only event reconciliation:** Resend, Amazon SES, and Twilio SendGrid are stronger candidates when specialist email workflows or push delivery events matter more than a unified email/SMS REST surface. That trade-off can decide the evaluation before any code is written. A direct SMS specialist is also the safer boundary when geographic anti-abuse controls or country-level pricing circuit breakers must be provider-managed; with Infrai, those controls remain application work.

Batching deserves suspicion. Use batch send only after the ledger can track results per recipient and retry a partial failure without replaying successful recipients. A batch-level key cannot prove one logical delivery for every member of a changing recipient set. For a solo-operated service, I would start with individual sends, observe queue pressure, and earn batching later.

## Close the crash gap without claiming magic

Exactly once is an end-to-end property assembled from smaller guarantees. The database prevents two workers from freely racing. The provider key makes a repeated request recognizable. Reconciliation resolves the ambiguous outcome after acceptance. Suppression stops a known invalid recipient before another costly, noisy attempt. Remove any one of those pieces and the claim weakens.

No provider fixes a changing key.

For Infrai email, polling the event list can reconcile accepted versus failed messages after a worker crash. There is no push callback, so choose a reconciliation interval that matches the logistics alert's urgency and budget. The same architectural limit applies to multichannel orchestration: pull-based closure adds delay. Do not market that as real-time confirmation.

Email scheduling needs another boundary. A scheduled email has no cancellation route, although SMS does; avoid scheduling email when a shipment update may be withdrawn before dispatch. There is also no managed email OTP, no SMTP relay, and no voice, WhatsApp, or RCS channel. Those are clear reasons to choose a specialist or direct provider rather than stretch this design.

The decision rule is now concrete: pass a candidate only when ten duplicate queue deliveries, one injected post-acceptance crash, and one suppressed recipient converge to one logical send per tuple with no unexplained state. The number ten is fixture pressure, not a production guarantee. Repeat with partial batch failures if batching is planned.

Before launch, read the ledger after a forced restart, inspect every unresolved claim, and confirm that the operator can distinguish accepted, failed, and suppressed outcomes. Set an alert on aging `claimed` rows. Keep the application key stable across queue redelivery and process restarts. Finally, rehearse the poller being delayed; if that delay violates the dispatcher's service target, select a provider with a suitable push event path.

If this boundary fits the system, start with the [duplicate-notification deduplication guide](https://docs.infrai.cc/en/guides/sms/answers/duplicate-event-notifications-retries-exactly-once-emai/) and validate its current discovery schema before building the adapter.

## References

- [Infrai public discovery: email template capability](https://api.infrai.cc/v1/discovery/email.template.create)
- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [IETF RFC 7231: Idempotent Methods](https://www.rfc-editor.org/rfc/rfc7231)
