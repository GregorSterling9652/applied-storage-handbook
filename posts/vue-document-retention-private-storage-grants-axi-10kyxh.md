# Vue Document Retention: Private Storage Grants, Axios Progress, and Node Authorization

**Short answer:** For a Vue frontend that must send large signed documents to private object storage, let a Node backend authorize a single upload grant, let the browser transfer the bytes directly, and treat Axios progress as transport telemetry rather than proof of retention. The backend should own the object key, the deletion deadline, and the final state transition.

That last distinction is the design. A progress bar is useful feedback, but it cannot tell a B2B SaaS application whether a document is attached to the right tenant, whether its retention deadline was recorded, or whether a cleanup worker can find it later. Large-file throughput makes direct browser-to-storage transfer attractive; it does not make the browser trustworthy.

## Start with the deletion deadline, not the upload widget

The application scenario is signed documents with an explicit deletion deadline. That changes the storage record from “a file the browser sent” into a small lifecycle object: tenant, owner, storage key, expected size, content type, upload state, created time, deletion deadline, and eventually a deletion result. The bucket holds bytes. The database holds the policy.

The backend should derive tenant and user identity from the authenticated session, generate an opaque upload identifier, and construct the object key from server-controlled values. A filename from a file input can be retained as display metadata, but it should not become authority. A key assembled by the browser invites cross-tenant overwrites, confusing retries, and cleanup jobs that cannot distinguish an abandoned upload from an attached document.

For throughput, the browser sends the file bytes to storage instead of making the Node process receive and re-send them. The Node service still does the important work: validate size and media type, bind the intent to an application record, request a short-lived signed upload, and accept a completion signal that it verifies against the known key. Direct transfer moves the data path; it does not remove the control plane.

The deletion deadline needs the same discipline. Save it before issuing the upload grant, enforce that it is within the product policy, and have a worker query application records whose deadlines have passed. Do not infer expiration from a browser timer. A closed tab is not a retention system.

Keep it explicit.

## How should a Vue frontend upload a file to private object storage with Axios?

Give the frontend a deliberately small state machine: `idle`, `authorizing`, `uploading`, `verifying`, `complete`, and `failed`. The authorization response should contain the upload identifier, the signed target, the required method, the exact headers to send, and an expiry. The permanent storage key can remain a backend concern.

Axios's upload callback reports bytes transferred by the browser. It does not establish that the object has passed backend verification. Keep the interface in `verifying` after the callback reaches its apparent maximum, then show success only after the backend binds the known key to the document record. This prevents a familiar failure mode: the bar reaches 100 percent, the user navigates away, and the document never becomes an attachment.

The callback should also handle the unpleasant cases. A missing total means the UI cannot calculate a trustworthy percentage, so show an indeterminate state or a byte count. A cancellation should be retryable. A retry should reuse the same application upload identifier and obtain a fresh grant when the earlier one has expired; silently creating a second document makes later deletion ambiguous.

Here is the calculation and state transition logic in Python, kept separate from any provider SDK. The same invariants belong in the Vue component: never display verified ownership from a progress event, never send application credentials to the signed target, and never let a client-supplied key select the tenant prefix.

```python
from dataclasses import dataclass
from enum import Enum


class UploadState(str, Enum):
    IDLE = "idle"
    AUTHORIZING = "authorizing"
    UPLOADING = "uploading"
    VERIFYING = "verifying"
    COMPLETE = "complete"
    FAILED = "failed"


@dataclass(frozen=True)
class Progress:
    state: UploadState
    percent: int | None
    uploaded_bytes: int


def progress_from_event(uploaded: int, total: int | None) -> Progress:
    if total is None or total <= 0:
        return Progress(UploadState.UPLOADING, None, max(0, uploaded))

    bounded = min(max(uploaded, 0), total)
    percent = min(99, int((bounded / total) * 100))
    return Progress(UploadState.UPLOADING, percent, bounded)


def after_transfer_success() -> Progress:
    # Transport ended; backend verification still has to finish.
    return Progress(UploadState.VERIFYING, 100, 0)
```

The `99` cap is intentional. A browser handing over the last byte is not the same event as a server confirming the object and its deadline. Once the completion response is verified, the UI can replace the pending state with `complete` and a separate server-provided record summary.

## What should the Node backend verify before private storage accepts the document?

The authorization operation should be idempotent for the application record. Validate the authenticated tenant, allowed content type, declared size, and deletion deadline before asking storage for a signed request. Generate the key on the server, and persist the upload intent before returning it to the browser. That ordering gives a reconciliation worker something stable to inspect if the tab disappears after authorization.

The completion operation should accept an opaque upload identifier, load the server-side key, and inspect the stored object using the storage integration's metadata operation. Compare the observed size and relevant content metadata to the intent. Then perform one guarded database transition from `pending` to `complete`. A second completion request should return the already-complete result, not create another record.

Do not confuse a signed URL with a permanent permission. It is a time-bounded capability for the operation it was signed to perform. The browser must use the returned method and signed headers exactly. Application bearer credentials belong at the Node boundary, not in a request to a different storage origin. CORS must also allow the application origin, method, and headers; the frontend cannot repair a storage policy that was never configured.

A practical intent record might look like this:

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class DocumentIntent:
    upload_id: str
    tenant_id: str
    storage_key: str
    expected_bytes: int
    content_type: str
    delete_after: datetime
    state: str = "pending"


def can_complete(intent: DocumentIntent, observed_bytes: int) -> bool:
    return intent.state == "pending" and observed_bytes == intent.expected_bytes
```

This is policy scaffolding, not a storage client. The provider adapter should be the only place that knows how signed requests and metadata checks are constructed. That boundary keeps a change in storage protocol from leaking into the Vue state machine or the retention worker.

## Where do large-file uploads actually fail?

Throughput is usually discussed as bandwidth, but the failure modes are more mundane. A single request can be interrupted by a laptop sleeping, a mobile network changing, a browser tab being discarded, or a signed grant expiring while a large transfer is still in flight. A successful HTTP response can also be recorded without the application having committed its database state.

For each failure, define ownership and recovery before production:

| Failure mode | What the user sees | Correct recovery |
|---|---|---|
| Missing progress total | A bar cannot be calculated honestly | Show indeterminate progress and uploaded bytes |
| Browser cancellation | Transfer stops partway through | Mark the same intent retryable and issue a new grant |
| Grant expiry | The transfer cannot finish with the old capability | Re-authorize the same intent; do not create a second document |
| Refresh after transfer | The browser loses local state | Reload the intent and ask the backend to verify its known key |
| Database write after storage transfer | Bytes exist but the attachment is pending | Reconcile pending intents and apply the recorded deadline |
| Cleanup race | A worker sees a late completion near the deadline | Use a guarded state transition and a clear retention rule |

Multipart transfer can improve recovery for genuinely large objects because completed parts need not be sent again, but it creates more state: part numbers, completion, abort handling, and unfinished-part cleanup. It is suitable when measurements show that restarting a whole request is costly. It is not a substitute for an idempotent intent record, and it increases the test matrix.

I would test with the actual origin, headers, file sizes, sleep and resume behavior, cancellation, expired grants, duplicate completion, and deletion-worker timing. Your mileage may vary with browser, network, and provider; a benchmark on a fast office connection will not settle the behavior of a customer uploading a signed contract from a constrained connection.

## What belongs in the first upload rollout for a SaaS document system?

Ship the smallest complete lifecycle: one private storage target, one document class, one server-generated key scheme, one upload intent, and one completion transition. Instrument intent creation, transfer outcome, verification latency, pending age, abandoned bytes, and deletion outcomes. Never log the full signed query string. It is a credential with an expiry, not ordinary diagnostic text.

Consider a document intent that expects 2 GB and carries a deletion deadline three days after creation. The browser can transfer 1.4 GB, lose its network, receive a new grant, and finish the same intent later; the database must still contain one key, one deadline, and one audit trail. If the first transfer left bytes behind but no completion transition, reconciliation should inspect that known key and either attach it after verification or remove it according to the pending-upload policy. Creating a fresh record for every retry turns a temporary network event into duplicate retention obligations, and a cleanup worker cannot safely guess which copy is authoritative. This is why the identifier, rather than the progress bar, is the useful unit of recovery.

The catch is that this direct-transfer design is not suitable when the product needs the application to inspect or transform every byte before storage, when the browser cannot be granted the required cross-origin policy, or when the storage system lacks the retention controls required by the document policy. In those cases, proxy through a controlled ingestion service or choose storage with explicit immutability and lifecycle features. Stick with a simpler single-request flow when files are small and retrying the whole body is cheap; adopt multipart only when the operational cost is demonstrated.

Roll out behind a feature flag. Start with noncritical documents, compare intended and observed sizes, and run a cleanup drill before enabling deletion deadlines for regulated records. The decision rule is straightforward: direct browser transfer for large-file throughput, a server-owned intent for authorization, and a database-backed lifecycle for retention. The progress bar is the least authoritative part of the system.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://cloud.google.com/storage/docs
