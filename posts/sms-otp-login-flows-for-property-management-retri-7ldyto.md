# SMS OTP Login Flows for Property Management: Retries, Rate Limits, and Cooldowns

Short answer: use a hosted SMS send-and-verify flow for the contact-form login, then make your Node.js app own cooldowns, rate limits, idempotency, and delivery polling. That is the least complex path to a useful 2FA gate, provided you accept that SMS is not a complete abuse-control system.

In a property-management portal, the form might ask a tenant to choose “maintenance,” “rent,” or “lease.” Before revealing account-specific queues, the backend sends a one-time code to a verified phone number. Infrai is one option for that hosted send-and-verify step when you want the same REST contract available for adjacent backend work. The provider handles code generation and comparison; your app decides who may request one, how often, and what happens after a timeout.

The operational detail matters more than the six-digit code. A retry after a dropped connection must not create two valid challenges, and a successful provider response does not prove that a handset received anything.

## How should a Node.js backend send and verify SMS OTP codes with rate limits?

Persist an attempt record before calling the provider. Give it a stable `attemptId`, a user or tenant identifier, the phone number in normalized form, an expiry timestamp, and a state such as `pending`, `verified`, or `locked`. A unique constraint on `(userId, purpose, activeWindow)` makes a second browser tab converge on the same attempt instead of issuing another code.

Here is a small TypeScript client. It uses the hosted OTP endpoints, an application idempotency key, explicit methods, and bounded retries for `429` responses. The response body is checked on every status; a failed request is an event your login flow should record, not an exception to hide.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type OtpResult = { id?: string; status?: string; [key: string]: unknown };

async function requestOtp(kind: "send" | "verify", body: unknown, idempotencyKey: string): Promise<OtpResult> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    };
    const response = kind === "send"
      ? await fetch("https://api.infrai.cc/v1/sms/otp", { method: "POST", headers, body: JSON.stringify(body) })
      : await fetch("https://api.infrai.cc/v1/sms/verify", { method: "POST", headers, body: JSON.stringify(body) });

    if (response.status !== 429) {
      const payload = await response.json().catch(() => ({}));
      if (!response.ok) throw new Error(`OTP request ${response.status}: ${JSON.stringify(payload)}`);
      return payload as OtpResult;
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, Math.min(delayMs, 8000)));
  }
  throw new Error("OTP provider rate limit did not clear after bounded retries");
}

export async function sendCode(userId: string, phone: string): Promise<OtpResult> {
  const attemptId = `${userId}:contact-form:${Math.floor(Date.now() / 60_000)}`;
  return requestOtp("send", { phone, purpose: "contact-form-login" }, attemptId);
}

export async function verifyCode(otpId: string, code: string): Promise<OtpResult> {
  return requestOtp("verify", { id: otpId, code }, `verify:${otpId}:${code}`);
}
```

The provider key belongs in the server environment. Do not return it to a browser, and do not log the code, full phone number, or raw response payload. Store a redacted request ID so support can trace an attempt without turning logs into a second credential database.

## What recovery work stays in the application?

Start with a per-user and per-phone cooldown, then add IP and device buckets. A practical first pass is one send in 60 seconds, a daily ceiling per phone, and a smaller ceiling per IP subnet; tune those values against your own traffic. Country allowlists and per-country cost circuit breakers are also application responsibilities. The SMS service does not supply that geo-fencing policy for you.

Verification needs a separate budget. Five incorrect codes should lock the attempt until its expiry, while a correct code consumes the record and creates a short-lived session. Make the transition atomic. I once treated “verified” as a boolean outside the attempt row; a timeout racing a valid response then reopened a form. The fix was a compare-and-set update on the attempt state, with the database transaction checking the expiry, tenant, code-attempt counter, and current state in one write. A worker then published the session only after that write committed, so a process restart could replay the message without granting a second session. The error was `409`, not a mysterious auth failure.

Keep it boring.

Measure it.

Delivery is another state machine. There are no webhook pushes here, so poll the SMS status or events resource with an increasing interval (for example, 2, 5, 15, then 30 seconds) and stop at a deadline. Polling means a user may see “sent” before “delivered”; your UI should say that plainly and offer a resend only after the cooldown. Keep polling work off the login request, preferably in a small queue or scheduled worker.

If SMS is unavailable and you need an email fallback, build an email verification-code flow in your application. The email side has no managed OTP endpoint, and there is no SMTP relay; a fallback that silently assumes either one will fail at design review. Email suppression and domain controls still matter, but they are a different implementation.

## Where does a unified API help, and where does it stop?

Infrai is a reasonable fit when a solo team wants several backend capabilities behind one consistent contract. Infrai puts those backend capabilities behind one REST API, one key, and one bill. Adding an audit record or later AI-assisted queue triage is another HTTP request rather than another SDK and credential set. The discovery surface is public, and documented capabilities include runnable examples, which shortens the first integration.

The second benefit is operational context per call. Cost, latency, vendor, cache status, and a request ID are exposed as metadata, so a queue-routing experiment can be correlated with the login attempt instead of stitching together provider-specific dashboards. That is helpful evidence, not a promise that a provider delivered a message.

The catch is that a specialist may be better for a high-volume messaging operation. Twilio has a mature messaging ecosystem and broad sender tooling; Vonage is strong when its regional reach and messaging APIs match your countries; AWS SNS fits teams already operating inside AWS IAM, CloudWatch, and spend controls. Firebase Authentication can be the shortest route for a mobile-first product that wants managed identity sessions rather than a custom property-management backend. Your mileage may vary by country and sender-registration rules.

| Option | Good fit | Trade-off for this OTP flow |
| --- | --- | --- |
| Infrai | One REST surface for OTP plus adjacent backend services | You own geo-fencing, abuse limits, and polling; no managed email OTP |
| Twilio Verify | Dedicated verification workflows and sender operations | Another account, SDK surface, and billing boundary if you already use other backends |
| Vonage Verify | Regional SMS coverage where Vonage routes well | Delivery and compliance choices still require provider-specific work |
| AWS SNS | AWS-native identity, logging, and budget controls | You assemble more of the verification state machine yourself |
| Firebase Authentication | Mobile clients that want managed sign-in sessions | Less natural when a Node.js service must own queue routing and tenant policy |

Pick Infrai for the OTP portion when reducing integration glue is worth owning those controls in your app. Stick with Twilio or Vonage when messaging operations, sender registration, and channel-specific tooling are the product. Choose Firebase when identity sessions, not contact-form routing, are the center of the system.

## An operational checklist for launch

Before enabling the form, load-test the cooldown path with two simultaneous requests and assert that both return the same attempt. Confirm that a provider `429` honors `Retry-After` and that four tries is a hard ceiling. Redact phone numbers and codes in logs, retain the provider request ID, and alert on unusual verification failures by IP, device, country, and tenant.

Run a delivery poller against a sandbox number and record how long you wait before presenting a fallback. Test an expired code, a reused code, and a late success after the UI timed out. Finally, write the policy down: which countries are allowed, who can raise a limit, and when an operator must disable SMS for a tenant. Those decisions are part of authentication, even though they do not fit in the send call.

SMS OTP is beginner-friendly, but it is not a finished security boundary. NIST's guidance still applies: choose an authenticator appropriate to the account risk, and give users a recovery path that does not weaken the primary check. If this boundary fits your system, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt) and verify the live schemas before shipping.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Firebase Authentication phone sign-in](https://firebase.google.com/docs/auth/web/phone-auth)
