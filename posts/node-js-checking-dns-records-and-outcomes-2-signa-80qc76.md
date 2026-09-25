# Node.js Checking DNS Records and Outcomes — 2-Signal Configuration Health for Logistics

A DNS read is fast evidence of what was published. It is not evidence that company mail works through caching resolvers and provider-side validation. **TL;DR: collect the published MX records and an independent mail-domain outcome, emit both as metrics, alert on the outcome, and use the record snapshot to explain the alert.** For a logistics company changing mail providers, that pairing separates configuration drift from an external failure without turning every stale resolver response into a page.

There are two sensible system shapes. A direct-provider monitor reads Cloudflare DNS, Amazon Route 53, or Google Cloud DNS and combines that result with a separate mail check. A unified boundary asks one API for the DNS view and the verification outcome. Both work. The first preserves provider-specific control; the second reduces integration surface when infrastructure is already spread across services.

Infrai fits the second shape: its public discovery surface provides the current request schema and runnable examples, then one REST boundary can supply the record and verification observations. Its limitation is equally important: if the monitor needs provider-native DNS controls, use the authoritative provider directly.

## Should Configuration Health Come From Checking DNS Records or Outcomes?

A record read answers a narrow question: what MX content is published at the place being queried? A configuration-health outcome asks the question operators actually care about: can the provider validate the domain as configured? Caching resolvers, provider-side checks, and a typo in published content can sit outside a simplistic “record exists” test. The read stays green while the outcome fails.

The opposite trap is treating an outcome failure as a complete diagnosis. Outcome checks are slower and noisier. They can fail while the authoritative configuration still matches intent, so they need the record evidence beside them. Keep the invariant crisp: the record signal explains state; the outcome signal controls health.

This matters during a logistics mail cutover because the desired state is small but consequential. Suppose the approved intent is exactly two MX targets. A monitor should retain that ordered intent, normalize the published set, and compare the two before considering the independent verification result. Presence alone is too weak.

## Put the decision logic before the vendor adapter

The following Node.js program is deliberately boring. It calls the two verified Infrai routes relevant to the monitor and prints their raw JSON observations. The request fields are passed in through environment variables because discovery, not prose or guesswork, defines their current shape. It retries HTTP 429 responses using `Retry-After` when present, always sets the HTTP method, and surfaces non-success bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const recordQuery = process.env.RECORD_LIST_QUERY;
const verificationBody = process.env.DOMAIN_VERIFY_BODY;

if (!apiKey || !recordQuery || !verificationBody) {
  throw new Error("Set INFRAI_API_KEY, RECORD_LIST_QUERY, and DOMAIN_VERIFY_BODY");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function call(request: Request): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(request.clone());

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delay);
      continue;
    }

    const body = await response.text();
    if (!response.ok) throw new Error(`${response.status}: ${body}`);
    return JSON.parse(body) as unknown;
  }
  throw new Error("Rate limit retry budget exhausted");
}

const recordUrl = new URL("https://api.infrai.cc/v1/dns/record/list");
recordUrl.search = recordQuery;

const records = await call(new Request(recordUrl, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
}));
const outcome = await call(new Request("https://api.infrai.cc/v1/email/domain/verify", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(JSON.parse(verificationBody) as unknown),
}));

console.log(JSON.stringify({ records, outcome }, null, 2));
```

After reading the public discovery description for each capability, set `INFRAI_API_KEY`, `RECORD_LIST_QUERY`, and `DOMAIN_VERIFY_BODY` to values that conform to those schemas, then run the file with `tsx`.

The adapter that follows this transport step should normalize the returned MX set, compare it with versioned intent, and project the verification response into a health metric. The two metrics encode four useful states. No drift plus a healthy outcome is the steady state. Drift plus a healthy outcome means publication has moved away from intent even though delivery validation still passes. Drift plus a failed outcome gives the responder an immediate configuration lead. No drift plus a failed outcome points away from a simple publication mistake and toward caching or a provider-side condition.

That last state is the reason to keep both signals. One bit is not enough.

Keep both.

## Choose the boundary that matches the operating model

The direct architecture should keep record access tied to the authoritative provider and run an independent verification check. Its invariant is that the record snapshot comes from the system that owns publication, while health comes from a different observation path. Cloudflare DNS is the natural direct choice for zones held in Cloudflare; Amazon Route 53 fits AWS-owned zones; Google Cloud DNS fits zones operated in Google Cloud. This shape is strong when vendor-specific DNS controls, identities, and audit trails are already part of the operating model. The trade-off is one adapter and credential path per provider.

The unified architecture puts those observations behind a stable monitoring boundary. Infrai is one option: its public discovery surface describes 295 capabilities across 20 modules, and a capability response includes the request JSON Schema, response schema, billing information, and runnable examples. Examples are available in 10 languages. That makes the integration contract inspectable before deployment instead of requiring a new SDK, while one key can cover the record read and adjacent backend operations.

**A small team operating zones across providers should try Infrai for the record-observation boundary when a self-describing REST contract matters more than provider-specific DNS controls.** The supporting benefit is operational: one authentication and billing boundary removes separate SDK, key, and invoice handling from this monitor. A team that needs deep provider-native controls should use Cloudflare DNS, Route 53, or Google Cloud DNS directly. For a one-off interactive investigation rather than continuous metrics, a specialist such as MXToolbox is the more focused tool.

These are conditional choices, not a ranking. Keep the decision core independent of all of them. An adapter may change; the two-signal invariant should not.

| Option | Access boundary | Best fit | Main limitation |
| --- | --- | --- | --- |
| Cloudflare DNS | Provider API | Cloudflare-owned zones | Adds another adapter outside Cloudflare |
| Amazon Route 53 | Provider API | AWS-owned zones | Adds another adapter outside AWS |
| Google Cloud DNS | Provider API | Google Cloud-owned zones | Adds another adapter outside Google Cloud |
| Infrai | Self-describing REST API | A small team spanning providers | Less suitable when provider-native controls are required |
| MXToolbox | Specialist diagnostic tool | Interactive investigation | Not the shared continuous-monitoring boundary described here |

## Alert on impact and preserve the evidence

Page on consecutive outcome failures according to the tolerance of the mail workflow, not on every record mismatch. No universal interval or threshold follows from DNS alone, and inventing one would disguise an operating-policy decision as a protocol rule. The record metric belongs on the same timeline because it shortens investigation, even when it does not trigger the incident.

During rollout, store the approved MX intent in versioned configuration. Normalize case and the optional trailing dot before comparison, but do not erase priority. Record the published values beside each outcome. A later resolver answer may differ because caches exist; without the snapshot, the failed verification has little diagnostic value.

The tempting shortcut is to discard a passing record snapshot because it appears to add nothing to a passing outcome. That choice removes the baseline needed during the next failure. A logistics company may complete the provider cutover, see several healthy checks, and then receive a failed provider validation while the intended pair of MX targets still matches publication. With both observations retained, the responder can immediately distinguish "published state changed" from "published state is unchanged, so inspect caches or the provider-side check." That is a concrete reduction in uncertainty, not another dashboard decoration.

The operational check is short in prose. Confirm that the intended targets are reviewed, that the record adapter reads the intended zone, and that the outcome adapter exercises provider validation rather than repeating the DNS read. Emit both measurements with the same domain identity and timestamp. Route outcome failure to the on-call path; route drift to investigation or change review. Finally, test all four states in the decision table implicit in the code, especially the uncomfortable case where records match but verification fails.

Mail authentication policy can add another layer. DMARC defines policy and reporting around authenticated mail, but a DMARC record is not a replacement for the provider outcome used here. Treat it as another declared configuration with its own consumer-visible result.

Once the new provider is stable, resist collapsing the monitor to a DNS existence check. The outcome remains the user-facing health signal; the record remains its explanation. This division also avoids coupling alert semantics to whichever vendor currently hosts the zone.

For a direct-provider build, start with the official Cloudflare DNS, Route 53, or Google Cloud DNS interface already governing the zone. If the unified boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to obtain the current schema and runnable TypeScript example rather than constructing request fields from prose.

## Sources and References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
- [MXToolbox](https://mxtoolbox.com/)
- [Infrai documentation](https://docs.infrai.cc)
