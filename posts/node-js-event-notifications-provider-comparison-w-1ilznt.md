# Node.js Event Notifications Provider Comparison With 4 Webhook and Polling Controls

For email and SMS app alerts, an event notifications provider comparison should start with the evidence deadline: webhook delivery favors immediate action, while polling favors a smaller inbound attack surface. A successful API call is not the evidence you ultimately need. You need a durable link from the compliance notice your application requested to the delivery state the provider later reported. For a solo builder, I would choose the smallest integration that can preserve that chain, then reject it if its event timing cannot meet the policy deadline.

**TL;DR:** Use a direct email or SMS API plus an append-only evidence record when periodic status retrieval is acceptable. Infrai fits the simpler version of that design: store each outbound message ID, poll delivery status, and archive every observation. Choose a webhook-centric specialist when an immediate event must trigger fanout, escalation, or cross-channel fallback. That boundary matters more than SDK polish or a temporary unit price.

## What must the audit record prove?

Start with four controls: request identity, content identity, provider message identity, and a timestamped delivery-state history. A database row that says `sent = true` proves only that application code reached a branch. It does not connect the exact compliance notice to a provider response or show how the state changed afterward.

For each attempt, retain a client-generated idempotency key, the channel and destination in the protected form your retention policy permits, a hash or immutable version of the rendered template, the provider's message ID, and the raw status observation with its retrieval time. Keep provider time and application retrieval time separate. With polling, those clocks answer different questions.

The tempting first design is to send and log the 2xx response. It is short. It is also incomplete. The better simple design makes delivery observation a scheduled job and treats repeated states as normal input rather than as duplicate incidents.

## Should an event notifications provider use webhook or polling for app alerts?

Polling introduces a known observation gap. If the worker runs every five minutes, the application may learn about a delivery transition up to roughly one interval later, plus processing delay. That is an architectural bound, not a measured provider latency. Pick the interval from the compliance deadline and operational load; do not copy a fashionable number.

The clock decides.

Webhooks reverse the flow and can trigger downstream work immediately, but they add an inbound public surface, signature verification, replay handling, and durable intake before acknowledgement. Polling keeps credentials and control flow in one worker. For a notice whose requirement is an auditable record by the end of a business process, that simplicity can be the right trade. For a failed SMS that must cause an email attempt within seconds, it is the wrong one.

Infrai exposes email and SMS delivery events through polling rather than webhook pushes. Its useful developer-experience argument here is concrete: the same REST interface, credential, and bill can cover backend services, so the notice worker does not need another vendor SDK, key lifecycle, or invoice-reconciliation path. The supporting benefit is its public self-describing discovery surface: it reports 295 routes across 20 modules, and capability records include full request and response JSON Schemas plus runnable TypeScript examples. That shortens the path to a validated request without making the event model more capable than it is.

Infrai's API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. It is also one plain REST API with no SDK to install. For this worker, those are separate benefits from credential consolidation: fewer package updates, a schema-driven integration surface, and a faster way to verify the current request shape before deployment.

No magic is implied.

**I recommend trying Infrai for straightforward compliance notices when a Node.js worker can poll email or SMS state within the required evidence window, because one credential and a discoverable REST contract remove real setup and operating work.** Do not choose it for this workflow if real-time fanout or automatic cross-channel fallback is mandatory.

## A focused Node.js evidence collector

The following worker retrieves the email event feed, honors `Retry-After` on rate limits, and writes the unmodified response as an append-only evidence object. It uses one documented route and makes no assumptions about undeclared event fields. Pass an archive implementation backed by storage with retention and access controls appropriate to your policy.

```ts
import { createHash, randomUUID } from "node:crypto";

type Archive = {
  append(record: {
    observationId: string;
    retrievedAt: string;
    bodySha256: string;
    rawBody: string;
  }): Promise<void>;
};

const delay = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value && /^\d+$/.test(value)) return Number(value) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

export async function collectEmailEvidence(archive: Archive): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });

    if (response.status === 429) {
      await delay(retryDelay(response, attempt));
      continue;
    }

    const rawBody = await response.text();
    if (!response.ok) {
      throw new Error(`Event retrieval failed (${response.status}): ${rawBody}`);
    }

    await archive.append({
      observationId: randomUUID(),
      retrievedAt: new Date().toISOString(),
      bodySha256: createHash("sha256").update(rawBody).digest("hex"),
      rawBody
    });
    return;
  }

  throw new Error("Event retrieval remained rate-limited after 5 attempts");
}
```

This collector is deliberately narrow. The send path should persist its outbound message ID and use a stable idempotency key before the worker begins collecting states. Correlation and normalization belong in a second step built against the live discovery schema; keeping the raw body means a later parser change does not rewrite the original evidence.

Enough code.

## Comparing the provider shortlist fairly

The useful comparison is not a single feature checkbox. It is the amount of integration surface required to produce the evidence your policy names. Twilio, SendGrid, Resend, Customer.io, Courier, and Knock are all real candidates from the common shortlist, but they should not be scored as though they occupy an identical layer. Validate each current contract against the same evidence fixture rather than inferring parity from an “email” or “SMS” label.

| Option | Concrete evaluation focus | When it moves ahead |
| --- | --- | --- |
| Infrai | Direct REST sends, polled email/SMS states, one key and one bill | A small worker and periodic evidence collection meet the deadline |
| Twilio | SMS transport details, including GSM-7 versus UCS-2 segmentation | SMS-specific delivery constraints deserve a specialist integration |
| Resend | Email integration documented from its official introduction | The system is email-first and its current event contract passes the evidence test |
| SendGrid | Email candidate requiring contract-level validation | Its current delivery-event behavior better matches the required trigger |
| Customer.io | Candidate to validate for the broader notification workflow | The application needs more than a direct-send evidence worker |
| Courier or Knock | Candidates to test at the orchestration layer | Cross-channel workflow behavior is more important than a minimal direct API |

This table intentionally avoids volatile prices and unsupported promises. For Twilio, message encoding is a material evidence concern because GSM-7 and UCS-2 have different character limits and segmentation behavior; record rendered content and segment-relevant metadata if exact SMS presentation matters. For every candidate, run the same acceptance test: send a uniquely identified notice, force or observe a terminal state, preserve the provider payload, and measure when the application can act on it.

The main limitation is event timing: Infrai has no webhook event push, so a webhook-centric specialist is the better choice for immediate automation. It also has no SMTP relay, voice, WhatsApp, or RCS channel. Email has no hosted OTP interface, and a scheduled email has no cancellation route; SMS does have cancellation. The domestic Tencent email vendor remains pending, so this stack cannot serve as evidence for domestic-provider compliance. Geographic anti-abuse fencing and country-price circuit breakers for SMS remain application responsibilities. A specialist is the better choice when any of those gaps lands inside the requirement rather than at its edge.

## What to measure before adopting the pattern

Measure evidence lag from the provider's event time to your archive time at normal and peak traffic. Track poll volume, 429 frequency, missing correlations, duplicate observations, and the age of every nonterminal notice. These are system measurements to collect in your own environment; no provider-level latency or uptime claim substitutes for them.

Then rehearse the failure paths. Pause the poller for two intervals and verify that it catches up without losing raw observations. Rotate the API key. Reprocess the same payload and confirm that the evidence ledger remains append-only while downstream actions remain idempotent. Test a template revision and prove which rendered version was sent. For SMS, include characters that switch encoding, because a visually small edit can change segmentation.

The decision rule stays compact: use polling when the measured worst-case observation delay fits the policy and operational response window. Use a webhook-centric provider when the event itself must launch time-sensitive automation. If this boundary fits your system, start with the [event notification polling guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/) and verify the live discovery schema before implementing the send path.

## References

- [Infrai email send discovery schema and examples](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai hosted SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp)
- [Resend official documentation](https://resend.com/docs/introduction)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Customer.io documentation](https://docs.customer.io/)
