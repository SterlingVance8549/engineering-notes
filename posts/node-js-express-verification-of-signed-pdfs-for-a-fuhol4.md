# Node.js Express Verification of Signed PDFs for Auditable Health Data Redaction

**Short answer:** In Node.js, verify an incoming signed PDF against the expected certificate before redaction, preserve the original bytes, and make the redacted derivative a new, separately audited artifact.

That ordering is the result of a small experiment: a simple “open, redact, then verify” path looked convenient, but any rewrite can invalidate the original byte range and hide who approved the source. The deciding constraint in a healthtech handoff is the signature and audit trail, not request latency.

The practical target is a deterministic decision record: which certificate was expected, which certificate actually signed the file, what bytes were covered, and what happened to the derivative. I would measure false accepts, false rejects, and time spent in certificate-chain checks before copying this design into another service.

## How can Node.js verify an incoming signed PDF against the expected certificate?

A PDF signature covers a byte range in the file. ISO 32000-2 defines the PDF signature structures, but it does not turn an arbitrary certificate into a trusted identity. Your service still needs a trust policy: an expected certificate fingerprint or public key, an approved issuer set, validity-time rules, and a decision about revocation evidence. A mathematically valid signature from an unapproved certificate is still a rejection.

This is a policy choice, not a parser feature.

Keep that distinction visible in code review.

The first trap is normalizing the upload before verification. Re-serializing, fixing line endings, or running a PDF optimizer changes bytes. Keep the original upload immutable in controlled storage and calculate a SHA-256 digest over exactly those bytes. The digest is an audit pointer; it is not a substitute for checking the PDF's signed byte range and CMS signature.

The second trap is treating a successful parser call as approval. Parsing tells you that a library understood the file. Approval requires comparing the signer identity to policy and recording the reason for every branch, including an expired certificate, an unsupported algorithm, or an incomplete chain.

## How should an Express endpoint separate verification from redaction?

Make admission small and explicit. The endpoint authenticates the caller, applies a byte limit, stores the untouched stream, and sends a reference to a worker. It should not redact in memory and then attempt to prove provenance afterward. A synchronous path is acceptable for a tightly bounded internal tool; a queue is easier to reason about when rendering and chain validation have unpredictable cost.

Here is a deliberately generic TypeScript boundary. `verifyPdf` must be backed by a PDF/CMS implementation that returns the covered digest and signer certificate; the policy code remains yours.

```ts
import express from "express";
import { createHash } from "node:crypto";

type Verification = {
  coveredDigest: string;
  signerFingerprint: string;
  chainValidAt: string;
};

type PdfVerifier = (bytes: Buffer, at: Date) => Promise<Verification>;

const app = express();
const expectedFingerprint = process.env.EXPECTED_CERT_SHA256 ?? "";

export function buildVerifyHandler(verifyPdf: PdfVerifier) {
  return async (req: express.Request, res: express.Response) => {
    const chunks: Buffer[] = [];
    for await (const chunk of req) chunks.push(Buffer.from(chunk));
    const original = Buffer.concat(chunks);
    const originalDigest = createHash("sha256").update(original).digest("hex");

    let result: Verification;
    try {
      result = await verifyPdf(original, new Date());
    } catch {
      return res.status(422).json({ accepted: false, reason: "invalid_signature", originalDigest });
    }

    const accepted = result.signerFingerprint === expectedFingerprint &&
      result.chainValidAt.length > 0;
    return res.status(accepted ? 200 : 422).json({
      accepted,
      reason: accepted ? "trusted_signer" : "untrusted_signer",
      originalDigest,
      coveredDigest: result.coveredDigest,
    });
  };
}

app.post("/pdf/verify", buildVerifyHandler(async (bytes, at) => {
  // Connect this interface to a standards-compliant PDF/CMS verifier.
  throw new Error(`verification implementation required for ${bytes.length} bytes at ${at.toISOString()}`);
}));
```

The sample intentionally returns a decision, not a redacted file. A worker can consume the accepted reference, render a derivative, and write a second digest. The audit event should bind the original digest, signer fingerprint, policy version, redaction rule-set version, operator or service identity, and derivative digest. Store these fields append-only; do not overwrite a failed attempt with a later success.

## Which failure modes deserve a hard stop?

Hard-stop on a missing signature, a digest mismatch, an unapproved signer, an unsupported digest or public-key algorithm, and a certificate chain that cannot be evaluated under the policy's time. Do not silently fall back to “the PDF opened.” Route ambiguous cases to review with a stable reason code. That gives compliance staff something more useful than a 500 and lets operations distinguish bad input from a verifier outage.

There is a cost to strictness. Revocation checks can depend on network evidence and can make a previously accepted document impossible to replay. Record the evidence source and evaluation time, cache only within a stated policy window, and make offline behavior explicit. For high-risk sharing, fail closed when required evidence is unavailable; for lower-risk internal transfer, a documented pending state may be safer than pretending the check completed.

The main limitation is operational weight. This approach is a poor fit for a tiny, trusted utility that never shares patient data; the extra immutable storage and audit events are overhead there. In a regulated exchange, that overhead is the point. The trade-off should be recorded in the service decision log, because changing the sharing boundary changes the risk calculation.

I also keep redaction tests separate from signature tests. Fixtures include a valid signed file, a one-byte mutation, a certificate with the right subject but the wrong fingerprint, and a file whose signature covers only an earlier revision. The last case catches a subtle bug: appending content can leave an old signature cryptographically valid while the visible document has changed. The policy must decide whether that revision is acceptable, and the audit record must say so.

## What should be measured before shipping?

Track acceptance by policy reason, chain-validation duration, queue age, derivative-generation duration, and the count of review states. Sample the original and derivative digests in an audit export, but never log patient names, page text, or certificate private material. Alert on sudden shifts in signer fingerprints and on derivatives that lack a parent digest.

The smallest useful release has one immutable upload, one verification decision, one redaction job, and one append-only event linking both artifacts. Add concurrency and retries only after replay tests show the idempotency behavior you expect. A duplicate worker must not create a second “approved” event for the same original digest and policy version.

Ship the audit link first.

## References

- https://www.iso.org/standard/75839.html
- https://www.rfc-editor.org/rfc/rfc5652
- https://nodejs.org/api/crypto.html
- https://expressjs.com/en/api.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/422
