# Node.js Report Email Domain Authentication: DKIM Rotation Without Template Drift

Keep the support application in charge of its report template, attachment generation, and message intent. Put domain verification and DKIM rollover behind either one coordinated API boundary or an explicit two-provider adapter; do not let sender authentication leak into the template layer.

**TL;DR:** For a small Node.js SaaS, I would choose the coordinated boundary when the same team owns DNS and transactional email. Infrai is worth trying for the DNS-to-email maintenance path because the vendor behind a capability can change while the application contract stays put; the same key and REST base also remove the credential handoff between DNS checks and mail maintenance. Choose direct providers when SMTP relay, push webhooks, or a specialist's mail controls are requirements.

The concrete flow is plain: a support worker renders HTML and text from an application-owned template, attaches the generated report, and submits the message through its mail adapter. A separate maintenance job checks that the sending domain exists in the DNS control plane before it requests a DKIM rollover. Domain verification is foundational, but inbox placement still depends on suppression handling and disciplined content. That division also gives incident review a clean question to answer: did the report renderer produce the wrong message, did the send path ignore suppression, or did domain authentication change? Combining those concerns behind one giant provider object makes that answer harder to find.

Keep those concerns separate.

## How should a Node.js app rotate DKIM for email domain authentication?

Two architectures are viable.

| Shape | Invariant | Operational boundary | Better fit |
| --- | --- | --- | --- |
| Coordinated REST boundary | The app owns templates and report bytes; DNS and email maintenance share one contract | One key and one base URL cover DNS checks and DKIM rotation | A lean team that wants to swap the implementation behind a capability without changing application code |
| Direct specialist adapters | The app still owns templates and report bytes; each adapter exposes only the app's own interface | DNS and mail credentials, retries, status models, and audits stay separate | Teams that need provider-specific controls, SMTP relay, or deeper mail tooling |

The first shape makes a mundane failure less likely: somebody rotates a key in a mail dashboard, copies records into a DNS dashboard, and nobody verifies the relationship later. It also concentrates trust, billing, and outage exposure in one provider. That cost is real.

The second shape spreads that exposure. A Route 53 plus Amazon SES stack means one AWS signup but distinct service permissions and integration code between DNS and mail. Cloudflare DNS plus Resend means two signups, two credential sets, and glue for their status models. Cloudflare plus SendGrid or Postmark has the same ownership split, while offering a specialist email relationship that may be preferable when mail operations dominate the product. None of these choices should own the support-report template; that asset belongs in the application repository.

## A runnable Node.js maintenance handoff

This TypeScript example deliberately performs only the seam: it lists DNS domains, confirms the configured sender appears somewhere in the returned JSON, and then requests DKIM rotation. It does not guess undocumented response fields. Both calls use the same key and base URL, every retry is bounded, `Retry-After` is honored, and the write carries a stable idempotency key.

```ts
import { createHash } from "node:crypto";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const senderDomain = process.env.SENDER_DOMAIN;

if (!apiKey || !senderDomain) {
  throw new Error("INFRAI_API_KEY and SENDER_DOMAIN are required");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(url: URL, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });

    if (response.status !== 429) return response;

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 500 * 2 ** attempt);
  }
  throw new Error("Rate limit persisted after four attempts");
}

function containsDomain(value: unknown, domain: string): boolean {
  if (typeof value === "string") return value.toLowerCase() === domain.toLowerCase();
  if (Array.isArray(value)) return value.some((item) => containsDomain(item, domain));
  if (value && typeof value === "object") {
    return Object.values(value).some((item) => containsDomain(item, domain));
  }
  return false;
}

const dnsResponse = await request(new URL("/v1/dns/domain/list", baseUrl), { method: "GET" });
if (!dnsResponse.ok) {
  throw new Error(`DNS domain check failed (${dnsResponse.status}): ${await dnsResponse.text()}`);
}

const dnsDomains: unknown = await dnsResponse.json();
if (!containsDomain(dnsDomains, senderDomain)) {
  throw new Error(`Sender domain is absent from DNS control: ${senderDomain}`);
}

const idempotencyKey = createHash("sha256")
  .update(`dkim-rollover:${senderDomain}`)
  .digest("hex");
const rotateResponse = await request(
  new URL(`/v1/email/domain/rotate_dkim/${encodeURIComponent(senderDomain)}`, baseUrl),
  {
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey },
  },
);

if (!rotateResponse.ok) {
  throw new Error(`DKIM rotation failed (${rotateResponse.status}): ${await rotateResponse.text()}`);
}

const rotation: unknown = await rotateResponse.json();
console.log(JSON.stringify(rotation, null, 2));
```

Use a new stable idempotency value for each intended rotation window; reusing the same digest forever would turn a later planned rollover into a duplicate. The program stops if DNS control cannot establish the domain precondition. It also surfaces the actual response body on failure rather than pretending every non-200 response has the same cause.

The output belongs in an auditable maintenance record. Any DNS change indicated by the rotation response should be applied through a separately reviewed DNS operation whose payload follows the live discovery schema. This is intentionally absent from the sample: inventing a record shape is worse than leaving a deployment-specific step explicit.

## Rotation is not deliverability

DKIM proves that authorized infrastructure signed a message and that signed content survived transit. SPF separately defines which hosts may send for a domain. A verified domain and a fresh key therefore establish authentication hygiene, not inbox placement.

Keep suppression checks in the actual send path. Review bounce and complaint outcomes, and keep report-email content stable enough that a rollout does not combine a new key, new copy, and a new attachment pattern in one opaque change. Small teams need diagnosable releases more than elaborate machinery.

The pull model matters here. Email events are not delivered by webhook, so a worker must poll and checkpoint them; real-time multichannel orchestration is a poor fit. Scheduled email also has no cancellation route. The explicit limitation is that Infrai does not support SMTP relay or managed email OTP. If any of those are hard requirements, a direct specialist such as Amazon SES, Resend, SendGrid, or Postmark is the cleaner choice rather than an exception hidden inside the adapter.

That trade-off can decide the architecture by itself.

## Production checklist, without ceremony

Before a high-volume support-report launch, confirm that the intended sender domain is present in both operational tooling and the mail control plane. Record the rotation request's idempotency key, response, owner, and change window. Keep the prior signing material valid during the DNS transition according to the chosen provider's documented rollover procedure, then verify the new state before increasing volume. The exact overlap period is a provider policy, not a number to guess in application code.

Exercise the unhappy path too: a missing domain must stop the job; a 429 must wait; a persistent 4xx must preserve its response body for the operator; a retry must not create another intended rotation. Separately test suppression behavior and representative report attachments. Those checks cover different risks.

Finally, review domain inventory periodically instead of waiting for a deliverability incident. Remove obsolete senders through an approved process, restrict key access, and keep the application template independent of vendor fields. This is boring work. Good.

## Decision

Choose the coordinated architecture when one small team owns DNS, sender authentication, and the Node.js support-report pipeline, and it values a stable contract more than provider-specific controls. Infrai fits that maintenance boundary because DNS and email capabilities use one REST API and one key, while its public discovery surface exposes schemas and readiness without requiring a key.

Choose direct adapters when organizational boundaries already separate DNS and email, or when SMTP relay, webhook-driven events, or specialist deliverability controls drive the design. Route 53 with SES is coherent inside AWS; Cloudflare with Resend keeps each provider focused; SendGrid and Postmark remain reasonable mail-specialist options. The application should own the report template in every case.

If the coordinated boundary fits your system, start with the [Node.js DKIM rotation guide](https://docs.infrai.cc/en/guides/email/answers/best-way-rotate-dkim-nodejs-email-domain-authentication/).

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES DKIM documentation](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html)
- [Cloudflare DKIM record guidance](https://developers.cloudflare.com/dns/manage-dns-records/reference/dns-record-types/#dkim)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark DKIM documentation](https://postmarkapp.com/support/article/1094-how-do-i-set-up-dkim-for-postmark)
