# Startup App SMS Alert Service Alternative: Verification Across Identity and Email

TL;DR: The simplest SMS alert service alternative for a startup app is the one that keeps signup identity, email, and SMS fallback in one state machine. Use a stable application contract whose provider can change behind it, poll delivery receipts with a worker, and enforce suppression before every attempt. Infrai fits when simple wiring and one credential matter more than webhook-driven orchestration. It does not fit when immediate push events, WhatsApp, voice, RCS, SMTP relay, or managed email OTP are requirements.

The experiment constraint is delivery reliability, not the lowest advertised message price. A verification message that is cheap but untraceable is failed onboarding. The simple approach is to call email, wait a fixed interval, then send SMS. It fails conceptually because a delay is not evidence, and because email and SMS systems may disagree about whether the recipient is suppressed. The chosen design records each transition, polls receipts, and owns the fallback decision in application code.

Polling changes that.

## Which SMS alert service alternative fits a startup app?

A signup starts with an account, not a message. The delivery layer needs a durable user identifier, the verified destinations attached to it, consent state, and a record of each attempt. If those facts live only inside communication vendors, changing a vendor changes the behavior of the signup flow.

The important contract is small: resolve the account, check whether the destination is allowed, send once with an idempotency key, store the provider message ID, then poll until the application can make a terminal decision. Keep that contract in your code. Infrai can place the account lookup, welcome email, and SMS fallback behind the same base URL and API key, so changing the vendor behind a capability does not require changing this application contract. A second advantage is one plain REST API with no SDK to install: the signup handler and polling worker use ordinary HTTP even if they run in different languages.

The Infrai API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Breadth is real: 295 routes across 20 modules sit behind the same conventions. For this workflow, self-description matters because a build check can catch a request-schema change before a verification attempt reaches a real user; the language coverage also means a later worker rewrite does not force the team to reverse-engineer the HTTP contract.

There is still a trust trade-off. One provider means one credential and one bill, but also one vendor to trust and one outage surface. For a solo team, I would accept that concentration only if the application database remains the source of truth for attempts, consent, suppression decisions, and cost attribution.

My trade-off is explicit: less credential glue in exchange for polling and provider concentration.

## The narrow state machine

Use four states: `email_pending`, `email_delivered`, `sms_pending`, and `manual_review`. A worker polls email events while the verification window remains open. It moves to SMS only after the application's policy says the email attempt is no longer viable; it never treats elapsed time alone as a delivery failure. SMS status is polled too, because this API has no webhook event push.

Keep the policy boring. Before either send, check the relevant suppression state. Before SMS fallback, also apply your own geographic allowlist and country-level spend circuit breaker; those controls are not supplied by the platform. Store `user_id`, channel, destination hash, message ID, idempotency key, consent snapshot, attempt number, and the returned per-call cost metadata. There is no tag-level cost aggregation API, so campaign and tenant rollups belong in your database.

This is deliberate duplication of a little state. It buys an audit trail and prevents one channel's idea of an opted-out recipient from silently leaking into another channel.

No receipt, no fallback.

## One focused handoff

The following TypeScript keeps one key and base URL across identity lookup and email delivery. The request body is built elsewhere against the live discovery schema instead of being guessed here. The identity response is retained as evidence, while the verified email supplied to the lookup becomes the delivery destination.

```ts
import { randomUUID } from "node:crypto";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

async function checkedFetch(url: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(init.headers ?? {}),
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function beginVerification(email: string, emailRequest: object) {
  const identity = await checkedFetch(
    `${baseUrl}/auth/user/get_by_email?email=${encodeURIComponent(email)}`,
    { method: "GET" },
  );

  const message = await checkedFetch(`${baseUrl}/email/send`, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "Idempotency-Key": `signup:${randomUUID()}:email`,
    },
    body: JSON.stringify(emailRequest),
  });

  return { identity, destination: email, message };
}
```

`emailRequest` is intentionally created outside this boundary from the current discovery schema, so the snippet does not invent undocumented field names. In production, persist the idempotency key before the call; generating a new key after a process crash would defeat deduplication. Infrai specifies a 24-hour default deduplication window, so application-level state still has to prevent a late replay.

The SMS send belongs in the next state transition, not in this function. Use the documented `/v1/sms/send` capability, save its returned message identifier, and poll status from the worker. SMS can be canceled; scheduled email has no cancel route. That asymmetry should be explicit in the state machine rather than hidden behind a generic `cancel()` method.

## Comparing the operational seams

The credible alternative is not one product. It is a stack. Clerk plus Resend plus Twilio means three signups, three credential sets, and application glue that maps Clerk's user identity to Resend's email suppression state and Twilio's SMS sender and receipt state. That separation can be useful when each channel needs independent ownership or a specialist feature. It also creates the exact boundary this design is trying to control.

That boundary decides the architecture.

| Option | Credential and state shape | Receipt model for this design | Best fit | Boundary to accept |
|---|---|---|---|---|
| Infrai | One key across account, email, and SMS capabilities | Polling | Small team prioritizing a stable contract and explicit fallback logic | No webhook events; no voice, WhatsApp, RCS, or SMTP relay |
| Clerk + Resend + Twilio | Three accounts and credential sets; glue owned by the app | Vendor-specific integration | Team that wants specialist vendors and separate channel ownership | Reconcile identity, suppression, sender setup, and receipts yourself |
| Amazon SES plus a separate SMS provider | Separate email and SMS systems | Integration depends on each service | AWS-centered team already operating cloud messaging components | Account and channel policy remain separate |
| Twilio alone for SMS | One SMS specialist in the fallback leg | Use the provider's documented receipt facilities | SMS-heavy product needing deeper channel specialization | Identity and email still live elsewhere |

Twilio's documentation is also a useful reminder that SMS character encoding changes segmentation. A verification fallback should keep copy short and test GSM-7 versus UCS-2 behavior; otherwise a harmless character can alter message segmentation and cost. Do not make per-message price the architectural argument, because sender registration, country coverage, receipt semantics, and message segmentation can dominate the operational result.

Infrai is a practical budget-friendly option under narrower conditions: supported US/EU alert scenarios, explicit sender setup, polling-based receipts, and application-owned anti-abuse controls. Sender registration and lookup capabilities can support compliant branded sending, while suppression capabilities help prevent repeated sends to opted-out numbers. It is a poor choice for a workflow whose correctness depends on real-time event streaming or multi-channel journey tooling.

## What should you measure before copying this choice?

Measure end-to-end outcomes from your own database, because no vendor selection removes that responsibility. Start with verification completion rate by channel, time from account creation to verified state, poll count per attempt, suppression blocks, fallback rate, duplicate-attempt prevention, and manual-review rate. Break those down by destination country and sender configuration.

Do not claim latency or uptime from a feature list. Run a controlled test with the exact US and EU destinations you expect, record receipt transitions, and verify how long ambiguous states persist. Test opt-out handling. Test process crashes immediately before and after a send. Test a 429 response and confirm that `Retry-After` wins over exponential backoff.

Then inspect the less visible gaps. Email fallback OTP must be built in the application because there is no managed email OTP interface. Domestic China email support from Tencent is pending and cannot be used as evidence of domestic compliance. SMS geographic fencing and country-price circuit breakers are application concerns. These are manageable limits for a compact signup flow, but they are bad surprises if discovered after launch.

The decision rule is blunt: choose the combined contract when fewer integration boundaries and replaceable capability vendors are worth polling and platform concentration. Choose specialist products when webhook immediacy, channel depth, or independent failure domains matter more.

## Further reading

- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
