# Direct Browser Upload for User Documents: 4 Barriers (Presigned Storage or Server)

Short answer: let an authenticated browser upload directly with a narrowly scoped presigned request when the application can bind one object key, content type, size ceiling, and short expiry to one tenant; send the bytes through the application server when the server must inspect or transform them before storage accepts them.

For generated fintech reports, tenant isolation is the decision axis. Bandwidth and setup effort matter, but neither compensates for a design in which customer A can overwrite, discover, or receive customer B's report. A React client and a Node.js control plane don't change that rule. The control plane should authorize the report operation and choose the object identity; the browser should never invent a tenant prefix.

Direct upload is usually the cleaner data path for already-authorized, bounded files. It keeps bulk bytes away from the application tier while leaving policy decisions there. A server upload is the deliberate choice when byte-level custody is part of the policy, not a default response to anxiety about CORS.

## Eliminate four cross-tenant failure modes first

Start by separating authorization from transport. A presigned upload does not mean an unauthenticated upload: the browser first authenticates to the application, the application verifies the tenant and report record, and only then does it issue a temporary capability for one storage operation. The browser transports the file directly to object storage. With a server upload, the same authorization happens, but the application also receives and forwards every byte.

That distinction creates four barriers that must agree: the application identity, the tenant-owned report record, the exact object key, and the later download authorization. If any one of them is derived from browser input without a server-side ownership check, the architecture has a cross-tenant path regardless of how private the bucket is.

Use an opaque report identifier mapped to a tenant in the database, then derive a key such as `tenants/<internal-tenant-id>/reports/<report-id>/<version>`. Do not accept a complete key from the client. Do not use an email address, company name, or other mutable display field as the isolation boundary. An object key is an internal locator; the database remains the authority for who owns the report.

The catch is that a presigned request delegates exactly what was signed for its lifetime. Revoking the user's application session does not retroactively rewrite a capability already handed to the browser. Keep expiry short, issue it only after an authorization check, and make each request specific to one intended object. If policy requires immediate revocation during transfer, synchronous malware inspection, document normalization, or a compliance-controlled byte stream, direct upload is not suitable; use the server path or a quarantined ingestion service.

## Make report ownership a control-plane invariant

Treat upload creation as a state transition, not as a request for an arbitrary URL. The control plane creates a pending report version, records its tenant owner, decides the storage key, and returns the authorized upload instructions. Completion is a second transition: storage presence alone must not make a report visible to customers.

The contract should include the expected media type, maximum byte count, expiry, and an application-level idempotency key. If integrity matters, include the checksum mechanism supported by the selected storage API and verify the stored result before publication. I'm not sure a single checksum policy fits every report pipeline: encrypted payloads, multipart transfers, and post-upload transformations can change what the useful digest represents. Resolve that during design by naming the byte sequence being attested, then test it against the actual storage implementation.

Here is the policy boundary in Python. `signer` is a generic adapter around the chosen object-storage signing implementation; it receives a server-derived key and constraints, never a client-supplied tenant path.

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Protocol


ALLOWED_MEDIA_TYPES = {
    "application/pdf",
    "text/csv",
}
MAX_REPORT_BYTES = 25 * 1024 * 1024


@dataclass(frozen=True)
class PendingReport:
    report_id: str
    tenant_id: str
    version: int
    status: str


class ObjectSigner(Protocol):
    def create_upload(
        self,
        *,
        key: str,
        media_type: str,
        max_bytes: int,
        expires_in: timedelta,
    ) -> dict[str, object]: ...


def authorize_report_upload(
    *,
    authenticated_tenant_id: str,
    report: PendingReport,
    media_type: str,
    content_length: int,
    signer: ObjectSigner,
) -> dict[str, object]:
    if report.tenant_id != authenticated_tenant_id:
        raise PermissionError("report does not belong to the authenticated tenant")
    if report.status != "pending":
        raise ValueError("report version is not pending")
    if media_type not in ALLOWED_MEDIA_TYPES:
        raise ValueError("unsupported report media type")
    if not 0 < content_length <= MAX_REPORT_BYTES:
        raise ValueError("report size is outside the accepted range")

    object_key = (
        f"tenants/{report.tenant_id}/reports/"
        f"{report.report_id}/versions/{report.version}"
    )
    return signer.create_upload(
        key=object_key,
        media_type=media_type,
        max_bytes=MAX_REPORT_BYTES,
        expires_in=timedelta(minutes=5),
    )
```

This code is intentionally unimpressed by a browser's claims. It checks ownership and state first, constrains type and length, derives the key, and gives the signing layer a five-minute window. The exact signed fields and multipart behavior vary by object-storage API, so integration tests must assert what the chosen implementation actually enforces rather than assuming that a field in application JSON became a storage-side condition.

One subtle failure is publishing on the client's “upload complete” callback. That callback says the client observed a response; it does not prove that the pending database row, stored object metadata, checksum, and tenant ownership all agree. Finalization should re-read trusted storage metadata, compare it with the pending record, and atomically move the report version to an available state. No match, no publish.

## CORS is a browser boundary, not an authorization model

CORS decides whether browser JavaScript may make and read a cross-origin request. It does not decide which tenant owns an object. For direct browser uploads, allow only the deployed application origins, required methods, and required request headers. Keep development origins separate from production policy, and test the preflight request as part of deployment.

A common diagnostic trap is a generic browser CORS message masking a storage rejection. Consider the whole sequence: the control plane signs an upload for `application/pdf`, the React client omits that header or a request library substitutes another value, the browser sends a preflight because the request is cross-origin, and the eventual upload no longer matches the signed inputs. Developer tools may foreground the CORS symptom even though changing allowed origins cannot make the signature valid. Capture the network request, compare its method, URL, content type, and signed headers with the server's signing inputs, then inspect storage-side request logs where available. Repeat the test from the deployed origin because `localhost` proves very little about production policy. Also test an intentionally expired capability and a foreign origin, both of which must fail without making the pending report visible. Don't widen the origin rule to `*` as a reflex — that may change what JavaScript can read, but it cannot repair a mismatched signature, and broad origin access is a poor fit for authenticated financial reports. Your mileage may vary on the exact diagnostic detail exposed by each storage implementation, which is why this sequence needs an integration test rather than a runbook that assumes every browser error is literal.

Be strict.

Limits also belong at more than one layer. Reject an excessive declared size before signing, enforce the strongest corresponding condition the storage API supports, and verify actual stored size during finalization. A client-side check improves feedback but has no authority. For large reports, multipart upload introduces abandoned parts, per-part retries, completion ordering, and cleanup; adopt it only after measuring a real need, then record the upload identifier against the pending tenant-owned report rather than leaving it solely in browser state.

Keep the object namespace private. Customer downloads should repeat the ownership check and return a short-lived read capability or stream through the application when policy requires that custody. Upload isolation without download isolation is half a design.

## Should a browser upload user documents directly or use a server?

Once the contract is explicit, the transport comparison becomes less subjective.

| Concern | Presigned direct upload | Application-server upload |
|---|---|---|
| Tenant decision | Server authorizes and derives one object key before signing | Server authorizes and derives the key before forwarding |
| Byte path | Browser to object storage | Browser through application to object storage |
| CORS | Required between browser origin and storage endpoint | Usually confined to the application's own browser API policy |
| Inspection | Best with quarantine plus asynchronous validation | Can inspect synchronously before forwarding, subject to buffering or streaming design |
| Capacity pressure | Bulk transfer bypasses application workers | Worker, connection, memory, timeout, and egress budgets own the transfer |
| Revocation | Existing capability remains usable until its short expiry | Server can stop an in-progress transfer according to application policy |
| Operational burden | Signing correctness, CORS, lifecycle cleanup, and finalization | Backpressure, retries, request limits, scaling, and finalization |

The easiest setup for a small internal tool may be a server relay because it avoids a second browser origin and centralizes logs. Stick with that path when files are small, traffic is bounded, and mandatory inline inspection already lives in the application. It becomes a liability when slow customer connections occupy application capacity or platform request limits are close to report sizes.

Direct upload is a better default for bounded generated reports when the team can operate the two-step pending/finalized state machine. It is not suitable when the object service cannot express the constraints the threat model requires, when policy demands synchronous inspection before any byte enters durable storage, or when clients cannot implement the required retry semantics. There isn't a universal winner — tenant isolation survives only if the chosen path makes ownership checks and failure ownership explicit.

Watch failure modes by stage. Count authorization denials, signing attempts, expired capabilities, preflight failures, upload rejections, pending records that never finalize, checksum mismatches, and orphan cleanup. Correlate them with a report ID and tenant-safe internal identifier, but don't place presigned query strings in logs; they are temporary credentials. Alert on age and rate, not merely raw totals, because one abandoned upload is normal while a growing backlog indicates a broken state transition.

## Capacity cost follows the byte path

Price lists do not answer the architectural cost question. A relayed 20 MB report consumes inbound and outbound transfer through the application, holds a connection for the customer's upload duration, and adds retry traffic when either leg fails; a direct upload moves those bytes away from application workers but adds signing, finalization, and orphan-cleanup operations. Model both paths with report-size percentiles, concurrent slow uploads, retry rates, and retention volume. Average file size alone hides the capacity event that hurts.

The team cost shifts too. A relay keeps browser behavior familiar but makes the backend team own streaming limits, backpressure, and horizontal capacity. Direct upload makes signing and CORS part of the deployment contract, so frontend and platform engineers must test them together. Choose the burden the team can observe and rehearse. Don't claim savings until production measurements include failed transfers and cleanup work.

## Migrate the byte path without moving the trust boundary

Begin with one private bucket, one server-owned key scheme, and one report type. Exercise cross-tenant negative tests before happy paths: tenant A requests tenant B's report ID, a client changes the object key, the content type differs after signing, the declared length exceeds policy, a capability expires, and completion is repeated. The expected result is denial or an idempotent final state, never a second visible version.

Then shadow finalization metadata checks in production without publishing from the shadow result. After their behavior is understood, enable direct upload for a small cohort while retaining the same authorization record and download gate used by the server path. Rollback should switch transport, not object identity or ownership rules. This matters.

Finally, add cleanup for expired pending records and incomplete multipart sessions, rehearse credential rotation, and verify lifecycle rules against retention requirements. The architecture is ready when changing between direct and relayed transport does not change who may name a key, who may finalize a report, or who may download it.

## References

- https://developers.cloudflare.com/r2/
- https://cloud.google.com/storage/docs
