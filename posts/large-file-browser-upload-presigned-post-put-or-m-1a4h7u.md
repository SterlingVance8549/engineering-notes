# Large-File Browser Upload: Presigned POST, PUT, or Multipart for Compatible Storage

Short answer: for a developer tool that sends large media straight from a browser to compatible object storage, the recovery boundary matters more than the HTTP verb. A single presigned PUT or POST fits small, infrequent objects; multipart fits a large object when retransmitting one failed request would consume a meaningful part of the user's time or bandwidth.

This is an experiment note, not a storage shopping list. The simple design sends the whole file in one request and is easy to reason about. The more involved design records independently completed parts and finishes the object explicitly. The latter spends more control-plane effort to reduce the number of bytes that must be sent again after a disconnect.

For a browser-based media tool, that trade is visible. A user uploads a 1.8 GB screen recording, changes networks, and comes back to the tab. A single request has one blunt recovery action: start the transfer again. A multipart session can retain completed parts and retry only the missing work, provided the application owns the session state and can finish or abandon it deliberately. That last condition is where otherwise tidy demos become operational work: the product needs a durable record, a reconciliation path for a tab that closes at the wrong moment, a bounded retry policy, and an explicit answer for abandoned sessions; without those pieces, a progress indicator can report an optimistic local state while the storage object is still incomplete.

Measure first.

Record completion rate by object-size band, retransmitted bytes, time to completion, and abandoned sessions before choosing a cutoff.

That's the whole test.

## What should a browser upload use for large media and object storage?

Start with the failure mode. A presigned PUT gives the browser a scoped URL and a file body. A presigned POST gives it a policy-backed form submission. Both can be sensible for a single transfer. Neither, by itself, makes a whole-file retry cheap.

Multipart moves the retry boundary inside the object. The application creates an upload session, the browser sends parts, the client retains the successful-part results, and a completion request assembles the final object. A dropped connection then invalidates a part rather than the complete file. That is the useful property.

There is no honest universal byte threshold. It depends on file sizes, network churn, browser memory constraints, concurrency, storage limits, and how much implementation state the team is willing to operate. Your mileage may vary. Treat the threshold as a measured product policy, not a folklore constant copied from another uploader.

The catch is that multipart adds state and cleanup. It is not suitable when the product handles only tiny objects, when a simple one-request flow already meets its completion target, or when the team cannot observe and expire unfinished sessions. Stick with the single-request path in those cases.

## The data path should leave the application server

The application server should authorize an upload and return narrowly scoped upload information. The browser should then send the bytes directly to storage. That keeps media payloads out of the application server's bandwidth and memory path, while leaving ownership, authorization, and lookup metadata in the application database.

The browser-side contract can stay small. The server decides the object key and the permitted request shape; the client reports a successful response and does not invent storage policy. Signed request headers must match the headers the browser actually sends. A mismatch is an integration error, so make the contract explicit in the upload response and test it with the real browser origin.

Here is the single-request boundary as TypeScript. It deliberately does not hide the response body: a status that is not successful should be observable by the caller, and retry policy belongs to the product rather than to a generic helper.

```ts
export async function uploadOnce(
  presignedUrl: string,
  file: File,
  contentType = file.type || "application/octet-stream",
): Promise<void> {
  const response = await fetch(presignedUrl, {
    method: "PUT",
    headers: { "Content-Type": contentType },
    body: file,
  });

  if (!response.ok) {
    throw new Error(`Upload failed (${response.status}): ${await response.text()}`);
  }
}
```

The function is intentionally boring. Boring is good at this boundary. Retrying the entire file after an uncertain result can duplicate work, so the caller needs an idempotency decision, a status check, or a user-visible retry action appropriate to its storage contract. Do not call a request successful merely because `fetch` resolved.

## Multipart is a state machine, not a faster PUT

The useful multipart record is more than an upload URL. It should identify the object, the upload session, the expected parts, the parts already confirmed by storage, and a terminal state such as completed, cancelled, or expired. Persist enough information to resume without asking the browser to resend confirmed parts.

One practical client shape is:

```ts
type PartReceipt = {
  partNumber: number;
  etag: string;
  size: number;
};

type UploadSession = {
  objectKey: string;
  uploadId: string;
  expectedParts: number;
  receipts: PartReceipt[];
  status: "active" | "completed" | "cancelled" | "expired";
};

export function missingParts(session: UploadSession): number[] {
  const done = new Set(session.receipts.map((part) => part.partNumber));
  return Array.from({ length: session.expectedParts }, (_, index) => index + 1)
    .filter((partNumber) => !done.has(partNumber));
}
```

The example is a model, not a provider-specific SDK. A production flow still needs a create operation, a way to obtain signed part requests, a bounded concurrency policy, a completion operation, and an explicit abort path. Persist each confirmed receipt before moving on. If the tab closes after storage accepted a part but before the application saved its receipt, reconcile that uncertainty rather than blindly creating a second upload.

Cleanup belongs in the design review. A cancelled session should be abortable; an abandoned session needs an expiry policy and an operator-visible count. A lifecycle rule can be useful, but it does not replace application observability or a clear terminal transition. The exact part-size and retention settings are deployment choices; choose them from observed file distributions and storage constraints.

## What does a defensible upload experiment measure?

Run the comparison against the same media cohort and the same browser class. For the single-request path, measure total bytes sent, retries, and the time between a failed response and a completed retry. For multipart, measure part retries, active sessions, completion failures, aborts, and the storage cost of unfinished work.

The most useful metric is wasted transfer per completed object. A flow that finishes quickly in the happy path but resends gigabytes after a brief network change may be the wrong fit for large recordings. A multipart flow that rarely retries but leaves a growing inventory of unfinished sessions has a different failure, one that belongs in operations rather than in the upload button.

Watch browser behavior too. Keep UI progress tied to confirmed bytes, make cancellation explicit, and preserve enough state to tell a user whether the file is still uploading or merely has an open session. A green progress bar is not proof that the final object exists.

## Decision table for a browser-to-storage uploader

| Situation | Default path | Reason to revisit |
| --- | --- | --- |
| Small objects and stable sessions | Presigned PUT or POST | Whole-file retry becomes expensive |
| Large media with interrupted connections | Multipart | State and cleanup exceed the product's operational budget |
| Need to resume after a tab or network change | Multipart with persisted receipts | Resume semantics are not worth the added implementation |
| Public distribution or searchable metadata | Separate delivery and database design | Upload authorization alone does not solve discovery or delivery |

The decision is not a vendor ranking. It is a failure-budget choice: decide how much rework a user can tolerate, then select the smallest protocol that keeps that budget. The recommendation above is unsuitable for a team that cannot own multipart cleanup, and the simple path is unsuitable when a failed whole-file retry is already a product incident.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://www.backblaze.com/cloud-storage/pricing
