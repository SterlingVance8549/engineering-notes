# Build SaaS Event Alert Emails in Node.js — DKIM, Templates, Deliverability

TL;DR: Verify the sending domain before production, render reusable event templates, and treat delivery polling plus suppression as part of the send path. For a healthtech SaaS emailing a generated report as an attachment, I would choose the provider with the smallest integration surface that still exposes domain status, message events, and suppression controls. The send call is the easy part; proving that a report alert reached a valid recipient without repeatedly mailing a bounced address is the real system.

Do this in that order. A polished template sent from an untrusted default identity is still a poor production setup, while a verified domain without event handling leaves the application blind after acceptance. Open tracking should not be the success metric either: Apple Mail Privacy Protection can prevent senders from learning reliable Mail activity, so delivery events and the user's in-product report state carry more weight.

## Start the experiment with a failure ledger

Before choosing an API, write the rows the system must be able to explain: a report was generated, one notification attempt received a provider message identifier, polling later found delivery or a bounce, and suppression controlled the next attempt. This reverses the usual demo. Instead of celebrating an accepted request, the experiment passes only when the application can account for the final state without inspecting sensitive report contents. It also makes integration effort measurable: every state that exists only in a vendor dashboard is another support step the solo operator has to remember.

## How should you build SaaS event alert emails in Node.js?

Start with the domain. Publish the records requested by the provider, trigger verification, and block the production rollout until the provider reports the domain as verified. Keep the visible From address on that domain. Google's sender guidance explicitly calls for email authentication and describes SPF, DKIM, and DMARC expectations; those are operational requirements, not a launch-week polish item.

Next, create templates around product events rather than around pages in the app. `report.ready` needs a subject, a concise explanation, an attachment name, and a stable link back to the report record. `payment.failed` and `account.activity` deserve separate templates because their urgency, data, and suppression rules differ. Version the template identifier in application configuration so a content edit does not silently change an audited workflow. Attachments change the failure surface, too: generate the file first, give it a deterministic report ID, validate its media type and size against the selected provider's documented limits, and only then enqueue the notification. Do not put protected health information in a subject line or event tag. The email should say enough to orient the recipient, but the application remains the source of truth for access control.

The obvious first design is wrong.

Calling `send()` after report generation and marking the notification complete when the provider accepts it looks clean on a diagram. Acceptance is not delivery, though. A later bounce must affect the next send, and a retry can duplicate an attachment email unless the application owns an idempotency key. That correction changes both the data model and the worker lifecycle.

## A focused Node.js boundary for reports

The useful abstraction is small. Keep provider-specific HTTP or SDK code behind an adapter, but make the application own the event name, deterministic send key, template version, and suppression decision. For the consolidated REST option, the exact email request schema is discoverable and may evolve, so this runnable script takes a validated JSON request as input instead of inventing fields. Prepare `INFRAI_EMAIL_SEND_INPUT` from that schema, set `INFRAI_BASE_URL` to the documented API base, and keep the report ID in `INFRAI_IDEMPOTENCY_KEY`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const rawInput = process.env.INFRAI_EMAIL_SEND_INPUT;
const idempotencyKey = process.env.INFRAI_IDEMPOTENCY_KEY;

if (!apiKey || !baseUrl || !rawInput || !idempotencyKey) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, INFRAI_EMAIL_SEND_INPUT, and INFRAI_IDEMPOTENCY_KEY",
  );
}

const payload: unknown = JSON.parse(rawInput);

async function sendReport(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/v1/email/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return sendReport(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

sendReport().then((result) => process.stdout.write(`${JSON.stringify(result)}\n`));
```

The queue worker stores the returned message identifier beside the report notification, then schedules reconciliation with bounded backoff. Since the relevant email namespaces do not provide webhook event delivery, this design assumes pull-based status checks. That limits real-time orchestration: choose a polling interval from the product's actual urgency, cap attempts, and retain a `pending` state rather than pretending silence means success. The code retries only a rate limit, honors `Retry-After` when it is present, and reuses the same idempotency key; other failures surface immediately with the response body.

Short code, long consequences.

The idempotency key prevents a queue retry from becoming a second report email, while the local notification row provides a durable place for delivery state and the template version used. The platform convention specifies a 24-hour default deduplication window, so the application still needs its own durable sent-state beyond that window. A bounce adds the recipient to suppression before another event worker can send again. An opt-out should enter the same pre-send decision through the appropriate consent path.

## Where does each provider fit?

Integration effort is the primary axis here, but it is not equivalent to counting setup screens. Compare the work required to verify a domain, create and preview a template, attach the report, retrieve delivery events, and enforce suppression. Also check the current attachment limits and regional or contractual requirements directly with each vendor before sending health data; those details are deployment-specific and can change.

| Option | Integration shape | Best fit | Boundary to accept |
|---|---|---|---|
| Amazon SES | AWS service model with its own identity, sending, and event configuration | Teams already operating inside AWS that want email to follow existing cloud controls | More assembly work around templates, event routing, and suppression belongs in the surrounding AWS design |
| Twilio SendGrid | Dedicated email platform with domain authentication, templates, and email activity tooling | Teams that want a mature email-specific control plane | Adds another vendor account, credential, SDK or API surface, and bill to operate |
| Postmark | Transactional-email-focused product with templates, message streams, and delivery activity | Product teams that value a narrow transactional workflow | A focused email vendor does not consolidate unrelated backend services |
| Resend | Developer-oriented email API with domain setup and Node.js tooling | Small TypeScript teams optimizing for a quick, familiar integration | Verify that its current event, attachment, and compliance options match the production workflow |
| Infrai | Plain REST surface covering email alongside other backend capabilities under one key and one bill | A solo team that wants fewer credentials and invoices while keeping one consistent service boundary | Email events are pull-only, there is no SMTP relay, and the Tencent email vendor path remains pending |

There is no universal winner. If the application already has AWS identity, queues, monitoring, and billing controls, SES can be less organizational work despite requiring more assembly in code. Postmark is appealing when transactional email is the whole problem. SendGrid has a broader email operations surface, and Resend keeps the Node.js developer path compact. Those are real trade-offs, not consolation prizes.

Infrai becomes a strong option when the constraint is operational sprawl: one key and one bill for backend services means fewer dashboard credentials and fewer invoices to reconcile at month end. Its consistent REST boundary also fits a small service adapter. **The limitations are decisive in some systems:** it is not suitable for a China compliance posture because the Tencent path is pending, and it is the wrong choice if SMTP relay or push-based email events are hard requirements. Pick SES for an AWS-native control plane, or an email specialist such as Postmark or SendGrid when dedicated email operations matter more than consolidation.

## Pull-based delivery changes the reliability loop

Run reconciliation independently from report generation. A report can be ready even while its notification is deferred, and coupling those states makes support painful. Store at least the internal event ID, recipient, template version, provider message ID, latest delivery state, attempt timestamps, and idempotency key. Keep the report's clinical or personal content out of those operational fields. This creates three explicit outcomes—delivered, bounced, and still pending—instead of one misleading `sent` flag, and it gives support a record it can inspect without opening the attachment or reading sensitive template data.

For a pull-only event surface, the worker might poll quickly at first and then back off. Do not tight-loop. Stop on a terminal delivered or bounced event, move hard bounces into suppression, and expose unresolved notifications to operations after the retry window. A scheduled email also needs product scrutiny: email scheduling has no cancellation operation in this capability set, so immediate queue-controlled sending is easier to reason about when a report can be withdrawn.

Keep your own per-event-type accounting as well. There is no tag-aggregated cost reporting API, so `report.ready`, `payment.failed`, and `account.activity` should map to internal ledger rows if product or finance needs that view. This is less glamorous than another dashboard, but it preserves portability and answers the question the application actually asks.

Email should not impersonate an OTP channel. The email side has no managed OTP interface; a fallback email verification flow would be application-owned. SMS has different capabilities and additional business-layer concerns, including geographic controls and country-level pricing circuit breakers. Treat those as a separate threat model rather than quietly routing authentication through the report notification system.

## Governance decides whether the integration survives launch

Measure integration work from an empty test domain to a reconciled delivery record, not from package installation to a `202` response. Count the provider-specific concepts in the adapter, the credentials and bills the operator must manage, and the manual steps needed to diagnose a bounce. Then measure duplicate sends, terminal delivery latency, suppression hits, unresolved events, and attachment-generation failures by event type.

Don't optimize for opens.

Apple documents that Mail Privacy Protection prevents senders from seeing whether a recipient opened an email and masks IP information, which makes open-derived conclusions unreliable for this workflow. A better product signal is the authenticated user viewing or downloading the report, joined to delivery state without placing sensitive data in provider metadata.

The decision rule is blunt: **choose the narrowest integration that can verify your domain, send the templated attachment idempotently, expose delivery events, and enforce suppression.** Prefer an email specialist when its operational tooling removes work your team would otherwise build. Prefer an existing cloud service when your controls already live there. Prefer a consolidated REST provider when key and invoice sprawl are the binding constraint and polling is acceptable.

Only copy the choice after testing the full loop on a non-production domain. Send one report, reconcile it, force a safe bounce using the provider's documented test mechanism, confirm suppression, and retry the same application event. The winning integration is the one whose failure states your smallest on-call rotation can explain.

## References

- Google, "Email sender guidelines": https://support.google.com/a/answer/81126
- Apple, "Use Mail Privacy Protection": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Amazon Web Services, "Amazon SES Developer Guide": https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio SendGrid, "Email API onboarding": https://www.twilio.com/docs/sendgrid/for-developers/sending-email/api-getting-started
- Postmark, "Developer documentation": https://postmarkapp.com/developer
- Resend, "Documentation": https://resend.com/docs
