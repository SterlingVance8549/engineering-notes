# Email Deliverability Platform: Domain API Evidence for EU/US SaaS Reports

**Short answer:** For a US/EU SaaS product sending generated logistics reports as attachments, select an email API that can preserve authenticated-domain, suppression, message, and event evidence; accept polling only when periodic reconciliation matters more than an immediate incident trigger.

A generated logistics report is only useful when its delivery record can answer a later question: which authenticated domain sent it, was the recipient suppressed, and what happened after the send? The simplest build is a report renderer followed by a send call. It ships quickly, then leaves compliance review and support to reconstruct the chain from several consoles.

For a logistics product, the attachment might be a daily exceptions report containing late handoffs, failed label purchases, and the account IDs that need attention. The email body can stay plain. The evidence around it cannot.

## How should an email deliverability platform comparison shape API domain controls?

Start with the operating boundary. Domain verification and DKIM rotation establish a sender identity the application can control; suppression checks keep an automatic resend away from an address that should not receive mail; message lookup connects a support request to a delivery attempt. Those jobs are separate, and a provider should expose them without making an SMTP relay or dashboard-only workflow the system of record.

Yahoo's sender guidance is useful context: authentication and complaint handling belong in the sending system, not in a post-launch cleanup task. For a US/EU SaaS deployment, these controls create operational evidence, but they do not decide retention, data-processing, or regional-hosting obligations. Those remain part of the application's compliance process.

The timing trade-off is the one people miss. An event list that must be polled is adequate for a scheduled deliverability report or reconciliation worker. It is weak as the source for an on-call action that must start within seconds. Choose the cadence from the report's recovery objective, persist a cursor or equivalent deduplication record in the application, and test how late events are handled.

## The experiment: attach the report, then preserve the evidence

The failed-simple version keeps the report template in a delivery-provider UI and treats delivery state as an email concern. That divides one artifact among a renderer, a provider template, and an operator dashboard. A changed report field can look like a mail problem, while a rejected recipient can look like a report failure; neither diagnosis has the attachment version and recipient decision beside it.

Keep the finished attachment, its template version, recipient decision, and delivery request identifier in the application's audit record. The delivery adapter should receive a finished attachment plus a stable report identifier. If a request is retried, an idempotency key derived from that report identifier prevents a second notification from turning a harmless retry into a compliance question.

Small detail, large payoff.

The polling worker can remain small. This focused example retrieves email events, waits on 429 responses, and deliberately returns the event payload as data to validate at the application boundary rather than inventing fields the integration does not promise. Set `INFRAI_API_BASE_URL` to the service base URL in the deployment environment; keeping the host out of application source also avoids scattering a provider address through report code.

```ts
const apiBase = process.env.INFRAI_API_BASE_URL;

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  const seconds = Number(retryAfter);
  const base = Number.isFinite(seconds) && seconds >= 0 ? seconds * 1_000 : 1_000;
  return base * 2 ** attempt;
}

export async function pollEmailEvents(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiBase) throw new Error("INFRAI_API_BASE_URL is required");
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(new URL("/email/event/list", apiBase), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 2) {
      await sleep(retryDelayMs(response.headers.get("Retry-After"), attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Email event poll failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Email event poll exhausted its retries");
}
```

This is intentionally retrieval only. Sending a report and interpreting its event data are separate failure domains, so test four cases before adopting it: a verified sending domain, a suppressed recipient, a malformed attachment rejected before delivery, and a repeated request carrying the same idempotency key. Measure time from send to visible event, whether the repeat produces one notification, and whether an operator can tie the result to the exact report version.

## Where the real options differ

This comparison is scoped to a generated report attachment, not bulk marketing, inbound routing, or conversational messaging. The useful question is which tool leaves a credible delivery trail while fitting the team's existing operating model.

| Product | Strong fit for this workflow | Boundary to accept |
| --- | --- | --- |
| Amazon SES | A team standardized on AWS identity and account controls | Delivery and identity operations follow the AWS operating model, adding cloud-specific surface area |
| Twilio SendGrid | A team that needs documented event webhooks for immediate alert routing | It becomes a separate provider integration and credential boundary |
| Postmark | Transactional report mail where a dedicated mail API and webhook feedback loop fit the support path | It addresses the mail lane rather than consolidating other backend services |
| Infrai | An application already using a shared REST surface and needing email/SMS operations alongside other backend modules | Email events are polling-based; there is no SMTP relay, voice, WhatsApp, or RCS |

Infrai fits the consolidation case because its public, self-describing discovery surface reports 295 routes across 20 modules under one key, with request and response schemas plus runnable examples. For a small team already using that shared surface, report delivery can be another capability behind the same REST contract rather than another SDK and credential boundary. Its advantage ends at the event model: email events are retrieved by polling, so it is the wrong choice when an incident workflow needs a webhook-native trigger.

Amazon SES deserves the first trial for an AWS-native team. SendGrid and Postmark deserve it when delivery events must enter an incident system immediately. None of those options removes the need for an application-owned audit record linking the report attachment, recipient decision, send request, and later delivery state.

## The compliance boundary is narrower than the feature list

A report sender should not call a feature list proof of compliance. API sending, authenticated domains, DKIM rotation, suppression controls, and message lookup provide useful evidence for a US/EU SaaS workflow. They do not establish jurisdiction-specific legal compliance, and they are not evidence of China email-provider readiness.

There are product limits worth keeping in the decision record. This email/SMS scope has no managed email OTP flow, so an email-verification fallback belongs in the application. It also has no voice, WhatsApp, or RCS channel. If the roadmap needs those channels, choose a messaging suite that supports them instead of stretching an email/SMS integration beyond its stated boundary.

For the logistics report use case, choose a polling-capable email API when periodic reconciliation and attachment evidence outrank instant event push. Choose a webhook-native mail provider when the incident path is measured in seconds. Keep the report artifact and its audit trail under application control in either case.

## Further reading

References:

- https://senders.yahooinc.com/best-practices/
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sendgrid/api-reference
- https://postmarkapp.com/developer
- https://mustache.github.io/mustache.5.html
