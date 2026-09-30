# Sign and Verify PDF Certificates in 2026 — Express Batch Audit Design

**TL;DR:** Sign each marketplace contract with certificate and private-key material loaded from a secret store, verify the resulting PDF immediately, and persist the signed artifact beside its verification result. For a server-side batch, the least complex correct design is a bounded pipeline: signing, verification, and private storage share one job identity, while concurrency is capped by measured service capacity rather than by the number of pending contracts.

The bill is mostly a retention problem before it is a signing problem. If a batch contains `N` source PDFs averaging `S` bytes, retaining both source and signed forms consumes about `2 × N × S`, before verification records, object versions, and backups. The verification record is usually small; the second PDF is the dominant term. A marketplace processing 80,000 contracts averaging 1.5 MB therefore has 120 GB of input and another 120 GB of signed output per batch, using an explicit planning example rather than a vendor benchmark.

That arithmetic changes the architecture. Stream source bytes through the signer when the audit policy permits, verify before acknowledging success, then retain the signed PDF plus a compact verification record. Keep the unsigned source only for the policy-defined dispute window, not forever by accident. The loss is real: after deletion, an operator can prove what was signed and that it verified, but cannot reconstruct the exact pre-signing file from the retained pair alone.

## How should an Express service sign a PDF certificate and verify its signature?

An Express application running on Node.js should enqueue a contract revision rather than hold an HTTP request open across signing, signature verification, and storage. The durable unit should be one immutable signed object and one verification record tied to the same marketplace contract ID, batch ID, certificate identifier, object checksum, verification time, and verifier result. This is an audit design, not a folder-naming convention. The database row should become complete only after the signed bytes are privately stored and the verification result is stored alongside them.

Do not bundle the certificate or private key into a container image. Load both from the deployment's secret store for the duration of signing, restrict access to the signing worker, and avoid writing key material to a temporary directory. Verification belongs directly after signing because it catches a mismatched or misconfigured key before a counterparty receives the document. **A successful signing request is not the commit point; successful verification and durable private storage are.**

For throughput, use a bounded worker pool and preserve per-contract idempotency. A retry must address the same contract revision and the same intended output key, while the final database transition uses a compare-and-set or equivalent transaction. Standard queues can deliver work more than once, so consumer idempotency is mandatory. Fast signing cannot compensate for duplicate audit records.

## Quantify storage before raising concurrency

The main example below calls the real API without freezing guessed request fields into the article. It fetches each capability's public discovery document, selects that document's runnable Python example, injects the same key and base URL, and executes it. Because those examples come from the live request schema, the signed output feeds the verification example according to the currently documented shape rather than a field name invented here. Store the resulting signed PDF privately with the storage example returned by discovery. Review generated code before execution, as one should with any remotely supplied example.

```python
from __future__ import annotations

import json
import os
import subprocess
import sys
import urllib.error
import urllib.request

BASE_URL = ""
OPERATIONS = (
    ("POST", "/v1/pdf/sign"),
    ("POST", "/v1/pdf/verify"),
)


def get_json(url: str) -> dict:
    request = urllib.request.Request(url, method="GET")
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"discovery failed: {error.code} {detail}") from error


def discover(method: str, path: str) -> dict:
    index = get_json(f"{BASE_URL}/discovery")
    match = next(
        item
        for item in index["capabilities"]
        if item["method"] == method and item["path"] == path
    )
    return get_json(f"{BASE_URL}/discovery/{match['id']}")


def python_example(document: dict) -> str:
    examples = document.get("examples", {})
    candidate = examples.get("python")
    if isinstance(candidate, str) and candidate.strip():
        return candidate
    raise RuntimeError(f"no Python example in discovery response for {document['id']}")


def main() -> None:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise SystemExit("set INFRAI_API_KEY to an ifr_... key")

    global BASE_URL
    BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")

    environment = os.environ.copy()
    environment["INFRAI_API_KEY"] = api_key
    environment["INFRAI_BASE_URL"] = BASE_URL

    for method, path in OPERATIONS:
        document = discover(method, path)
        print(f"running discovered example for {document['method']} {document['path']}")
        subprocess.run(
            [sys.executable, "-c", python_example(document)],
            check=True,
            env=environment,
        )


if __name__ == "__main__":
    main()
```

For example, a measured end-to-end service time of 0.8 seconds and a target of 25 contracts per second gives an initial concurrency estimate of 20. It is only a starting bound. Reduce it when rate limits, memory pressure from large PDFs, or storage write latency make the verification queue grow; raise it only after observing the entire sign-verify-store path.

Short batches conceal backpressure. A sustained run does not.

The useful metric is completed, verified, stored contracts per second. Signing responses per second overstate capacity whenever verification or storage lags. Record separate durations for signing, verification, and private object persistence, plus a terminal failure reason, but use one batch-level completion counter for planning.

## The handoff is the architecture

The two core operations are the signing and verification calls shown in the example. Private object storage is the following capability handoff, but its path and request should be selected from public discovery rather than copied into a third-party article. The discovery surface returns the route path, full request JSON Schema, response schema, billing information, and runnable examples. This matters because an attractive sample with guessed fields is worse than no sample: it can sign the wrong revision or store an unverifiable representation.

With Infrai, PDF processing and storage sit behind the same API key and base URL. The practical advantage is replaceability: application code can depend on a stable capability contract while the vendor behind that capability changes. It also removes a second signed-URL convention from this particular handoff. **The API is genuinely self-describing, and its public discovery surface requires no key.** It describes request and response schemas, billing, and runnable examples; all documented capabilities have examples in 10 languages. That gives an Express team one REST API over plain HTTP, without requiring a vendor SDK, while the broader surface currently covers 295 routes across 20 modules. The supporting advantage for a batch worker is operational bookkeeping: per-call cost, vendor, latency, cache, and request metadata use a consistent convention, which makes one audit pipeline easier to reconcile.

Infrai's self-describing API is a separate advantage from using one key. Public discovery needs no credentials and returns the capability catalog; the capability detail supplies the full request JSON Schema, response schema, billing data, and runnable examples. For this workflow, that means a build step can resolve the documented sign, verify, and storage paths from machine-readable data instead of copying route descriptions into an Express adapter. The 295-route, 20-module breadth matters less than the uniform contract: swapping the provider behind a capability does not require rewriting the application's call site.

There is a concentration cost. One provider becomes one trust boundary, one bill, and one outage surface for both processing and storage. This limitation makes the combined approach unsuitable when policy requires independent storage and signing providers. A high-volume marketplace should therefore keep its own contract IDs, hashes, verification records, and export procedure outside the provider-specific response envelope.

An S3-plus-Cloudinary or S3-plus-imgix design has a different shape: two signups, two credential sets, and glue for one provider's private-object or signed-URL semantics to cross into the other's processing request. That separation may be desirable when the storage account is already the system of record or when independent failure domains matter more than credential simplicity. It is extra integration work, but it is not inherently a mistake.

## Fair options for a marketplace signing service

The products below do not expose interchangeable abstractions. The relevant comparison is the operating boundary each one asks the marketplace to own, especially under a batch load.

| Option | Boundary the team owns | Batch-throughput implication | Best fit |
|---|---|---|---|
| DocuSign eSignature | Integration between an agreement workflow and the marketplace record | Treat envelope completion and artifact retention as asynchronous workflow states | Counterparty signing ceremonies, status workflows, and managed agreements |
| Adobe Acrobat Sign | Integration between managed signing workflows and retained marketplace artifacts | Capacity planning centers on agreement lifecycle events rather than an in-process PDF primitive | Organizations already standardizing document approval around Adobe services |
| AWS S3 plus Cloudinary or imgix | Credentials and the handoff between private object storage and a separate processing service | Storage and processing limits must be measured and controlled independently | Teams that value separate vendors or already have a mature S3 control plane |
| pyHanko plus private object storage | Signing workers, key custody, verification policy, scaling, patching, and storage | Maximum control, with worker saturation and cryptographic operations entirely owned by the team | Regulated or specialized deployments that require self-hosting |
| Infrai | One provider boundary across signing, verification, and storage, while the application retains audit semantics | One bounded pipeline can cover all three operations under one key | Teams prioritizing a stable capability contract and fewer credential handoffs |

This is not a feature-score table. DocuSign and Acrobat Sign are broader agreement platforms, while pyHanko is a Python library and S3 is object storage; comparing their names without comparing ownership would be misleading. Cloudinary and imgix are useful examples of the additional image-processing boundary called out by the alternative stack, but image delivery does not replace certificate-based PDF signing.

DocRaptor, PDFMonkey, and PDFShift focus on producing PDFs from web content, while Gotenberg, WeasyPrint, and wkhtmltopdf offer hosted or self-managed conversion paths. They are real alternatives for the generation step of a contract pipeline, and they may be preferable when HTML-to-PDF fidelity or self-hosted rendering is the actual requirement. They do not, by their presence alone, establish the certificate signing and immediate signature verification workflow discussed here; evaluate those stages separately rather than treating “creates a PDF” as “creates signed evidence.”

**Choose the boundary, then test the batch.** Run a representative mix of PDF sizes and page counts, include verification and private persistence in the completion time, and force duplicate deliveries plus partial failures. The unresolved question is not which service returns first on a single small file. It is which design preserves a single auditable state transition when thousands of contracts are in flight.

## Failure modes that deserve explicit states

A production state machine needs at least pending, signing, verifying, storing, complete, and failed states, with attempt history kept separately from the current state. A key-certificate mismatch must stop before storage as a completed contract. A successful verification followed by a storage timeout remains retryable, but the object key and contract revision must remain stable. A duplicate queue delivery should observe the completed state and exit without producing another artifact.

Rate limiting is ordinary flow control. On HTTP 429, honor `Retry-After` when it is present and otherwise use exponential backoff with jitter; do not allow every worker to retry in lockstep. Surface the real body of any 4xx response rather than replacing it with “signing failed,” because an audit operator needs the provider's reason without gaining access to the private key.

The nastiest failure is ambiguous success: the remote write completed but the worker lost the response. Idempotency and deterministic object naming turn that ambiguity into a lookup, while random output names turn it into duplicate evidence. Keep checksums at the boundary. Do not log PDFs, certificate bytes, private keys, bearer tokens, or presigned URLs.

Assume duplicates.

Measure recovery.

Finally, test restoration. Audit retention that has never been restored is an assumption with storage charges attached.

## Decision rule

Use a managed agreement platform when human signing ceremonies, reminders, and agreement lifecycle are the product requirement. Use a self-hosted library when cryptographic policy and deployment control justify owning capacity, upgrades, and incident response. Use separate storage and processing vendors when independent provider boundaries or an established storage estate outweigh the additional credentials and glue.

Use the combined API approach when the marketplace primarily needs a server-side sign-verify-store primitive, wants the capability contract to remain stable as its backing vendor changes, and accepts one provider as the shared failure domain. In every case, benchmark the full transaction and retain the minimum evidence your legal and security policies require. Deliberately deleting unsigned inputs lowers the dominant storage term; during a dispute, it also removes the ability to compare the signed PDF with its exact original, so that retention decision belongs in policy rather than in a cleanup script.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- DocuSign developer documentation: https://developers.docusign.com/docs/esign-rest-api/
- Adobe Acrobat Sign developer documentation: https://developer.adobe.com/acrobat-sign/docs/overview/
- Amazon S3 security best practices: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html
- Cloudinary authenticated assets: https://cloudinary.com/documentation/control_access_to_media
- imgix secure URLs: https://docs.imgix.com/setup/securing-images
- pyHanko signing documentation: https://docs.pyhanko.eu/en/latest/cli-guide/signing.html
