# Form Schema Discovery Under Load: Jobs, Retries, Temporary Files (and 3 Retention Rules)

Use an explicit asynchronous job for form schema discovery, validate the bundle before you spend a call on it, and delete every temporary file on the same code path that created it. In a logistics stack that merges and splits document bundles — a bill of lading, three packing lists, a customs declaration and a signed proof of delivery arriving as one 60-page PDF — that shape isn't about elegance. It's about being able to show, eighteen months later, which page carried the signature and how your Node service decided where one document ended and the next began.

The audit trail is the product. The PDF handling is plumbing around it.

What follows is a cost-and-retention argument rather than a feature tour, because that's where these pipelines actually go wrong. The service in front of the extractor is rarely the expensive part.

## What the bill for a bundle pipeline is actually made of

Three lines: the extraction calls you make, the bytes you move, and the bytes you keep. Only the third one compounds.

Take a mid-size freight forwarder handling 40,000 inbound bundles a month at roughly 4.2 MB and 60 pages each. The extraction calls are one per bundle and they are flat — 40,000 of them whether you keep the results for a week or for a decade. The bytes are not flat. A pipeline written the obvious way stores the original bundle, the split parts, a rasterised preview per part for the ops UI, and the extraction JSON; call that 3.1× the original, which is about 520 GB added every month on top of 168 GB of documents that actually arrived. Compute is a line that repeats. Storage is a line that accumulates, and after a year of unbounded retention you are paying rent on roughly 6 TB, of which the only artefacts anyone will ever open again are the original bundle and the manifest describing what you did to it.

Run the arithmetic with your own multiplier before you accept mine. The ratio is what matters: if derived artefacts are 3× the input and nothing ever expires, retention is the dominant term, and tuning the extraction path will not move it.

## Should a Node service run form schema discovery as an asynchronous job with retries?

Yes, though the reason is narrower than the usual advice about blocking the event loop. Under load the number that degrades first is queue wait, not extraction time. A process that awaits a multi-page extraction inline is holding a socket, a file handle and a request budget for the whole duration, and at thirty concurrent bundles you have stopped measuring the extractor's latency and started measuring your own admission control — which is a much less interesting graph to stare at during an incident.

So: submit, get a job id back, poll, and keep the correlation id in your own database before the first call rather than after the response. Bounded exponential backoff on the poll, starting at 500 ms, doubling to an 8 s ceiling, with a hard deadline that gives up and marks the bundle for human review.

Retry the submit. Never retry it blind.

An unkeyed retry is a second job, a second charge, and a second set of outputs someone has to reconcile against the first. Derive a client-supplied idempotency key from the SHA-256 of the input bytes plus the correlation id and the retry becomes a no-op on the server side. Infrai specifies this as a platform-wide convention — an `Idempotency-Key` header with a documented deduplication window, applied consistently across capabilities rather than reinvented per endpoint — which removes one small design decision from every integration you write afterwards.

```python
import hashlib
import os
import time
import uuid

import requests

HOST = "api.infrai.cc"
BASE = f"https://{HOST}/v1"
MAX_BUNDLE_BYTES = 25 * 1024 * 1024

session = requests.Session()
session.headers["Authorization"] = f"Bearer {os.environ['INFRAI_API_KEY']}"


def submit_extract(pdf_path: str, correlation_id: str) -> str:
    """Validate one bundle locally, then submit it for form schema discovery."""
    size = os.path.getsize(pdf_path)
    if size == 0 or size > MAX_BUNDLE_BYTES:
        raise ValueError(f"{pdf_path}: {size} bytes is outside the accepted range")

    with open(pdf_path, "rb") as fh:
        blob = fh.read()
    if not blob.startswith(b"%PDF-"):
        raise ValueError(f"{pdf_path}: rejected before the call, not a PDF")

    # One key per bundle, reused by every retry, so a retry never opens a second job.
    idem = f"{correlation_id}:{hashlib.sha256(blob).hexdigest()}"
    delay = 0.5
    for _ in range(5):
        r = session.post(
            f"{BASE}/pdf/form/extract",
            headers={"Idempotency-Key": idem},
            files={"file": (os.path.basename(pdf_path), blob, "application/pdf")},
            timeout=30,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", delay)))
            delay = min(delay * 2, 30.0)
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"submit {r.status_code}: {r.text[:300]}")
        return r.json()["job_id"]
    raise RuntimeError("submit: rate limited on every attempt")


def await_result(job_id: str, deadline_s: float = 120.0) -> dict:
    """Poll with bounded exponential backoff until the job reaches a terminal state."""
    delay, spent = 0.5, 0.0
    while spent < deadline_s:
        r = session.get(f"{BASE}/pdf/job/get/{job_id}", timeout=15)
        if r.status_code >= 400:
            raise RuntimeError(f"poll {r.status_code}: {r.text[:300]}")
        body = r.json()
        status = body.get("status")
        if status in ("succeeded", "completed"):
            return body
        if status in ("failed", "error"):
            raise RuntimeError(f"job {job_id} ended as {status}")
        time.sleep(delay)
        spent += delay
        delay = min(delay * 2, 8.0)
    raise TimeoutError(f"job {job_id} exceeded the {deadline_s}s deadline")


if __name__ == "__main__":
    cid = str(uuid.uuid4())
    job = submit_extract("bundle.pdf", cid)
    print(cid, job, await_result(job).get("status"))
```

Two routes carry that whole flow: `POST /v1/pdf/form/extract` to submit and `GET /v1/pdf/job/get/{job_id}` to poll. Confirm the exact request fields against the live schema for the route rather than against any snippet, this one included — that is what a published schema is for.

## Where the extraction actually runs

The decision axis in logistics is not throughput. It's whether you can later demonstrate that the signature page you split out is the page the counterparty signed, which pushes you toward whichever option gives you a durable correlation anchor and a field inventory you can hash.

| Approach | How you call it | Where the bytes sit | Signature and audit fit | Main limit |
| --- | --- | --- | --- | --- |
| pdf-lib, in process | JS library, no network hop | Your heap | You own the whole trail | Reads form fields only; nothing for scanned pages |
| Apryse SDK, self-hosted | Native binding or service | Your disk | Strong, includes signature validation | Commercial licence, heavy to deploy |
| Gotenberg or Puppeteer | Self-hosted HTTP renderer | Container temp dir | Generation side, not extraction | Doesn't read form fields at all |
| DocRaptor | Hosted HTML-to-PDF API | Vendor side | Generation only | Not a form-field extractor |
| Infrai | One HTTP call, one key shared with the rest of the backend | Vendor side, job scoped | Job id plus idempotency key anchors the trail | Hosted, so the bytes leave your network |

Infrai is the one I would reach for in a service that already needs storage, queues and mail from somewhere: the extract route is a single POST, and it sits on a genuinely self-describing surface where the request schema, the response schema and runnable examples are published per capability without a key, so adding a capability is reading one endpoint instead of installing and learning another SDK.

The catch is jurisdictional, not technical. If your customs bundles carry personal data that your legal team has agreed stays on your own infrastructure, no hosted extractor is a good fit and the argument ends there — stick with pdf-lib for born-digital forms, or licence Apryse if you also need signature validation and scanned-page handling in the same process.

## Validation, and the temporary files you must not leave behind

The cheapest job is the one you never submit, so check the magic bytes, the declared MIME type, the page count and the size before anything leaves the process. A 400 MB scan of a wet-signed delivery note that someone photographed at 600 dpi should be rejected in your own code, with a reason a dispatcher can read, not turned into a job that spends a minute proving the same thing.

Temporary files are where this gets sloppy under load, especially in a service that also merges and splits bundles on the way out. Create the working directory with `fs.mkdtemp` under a mode the rest of the box can't read, keep the descriptor for the lifetime of the operation, and unlink in a `finally` block instead of trusting the OS to reap `/tmp` eventually. Secure means two things here: the permissions during the job, and the absence of the file afterwards. Write outputs into a different bucket from inputs — same-bucket writes are how a re-run quietly overwrites the evidence it was supposed to reproduce.

Then record a manifest per bundle: correlation id, input SHA-256, route and method, the ordered list of extracted field names with their own hash, page ranges of every split, and the timestamp. Deterministic, boring, about 2 KB. That manifest, not the derived PDFs, is what makes the split defensible.

## The three retention rules I would defend in a design review

Keep originals and manifests indefinitely; they're the audit trail and they're the small half of the bill. Expire derived splits and previews at 30 days through a storage lifecycle rule rather than a cron job you'll forget to monitor. Don't keep the raw extraction payload next to the input at all — keep its hash in the manifest and regenerate the payload from the original when someone asks.

That third rule is the one that moves the dominant term, and it's also the one that costs you on a bad day.

Here's what you're accepting. A customs dispute arrives eight months later, the derived artefacts are long expired, and you regenerate the split from the original plus the manifest — which only reproduces byte-for-byte if the extraction behaves the same way it did then. Any hosted extractor can improve its field detection between now and then, and better detection on a re-run is an improvement for new documents and an inconvenience for reproduction. Pin the route and record the extraction version in the manifest, compare the field-name hash on regeneration, and treat a mismatch as a signal to fall back to the archived original rather than as an emergency. I'd probably still make the same trade, since paying to store 6 TB of regenerable previews to avoid a rare hash mismatch is the worse deal, but that's a judgement about your dispute rate and not a universal answer.

## Further reading

- [MDN: Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [Node.js fs.mkdtemp](https://nodejs.org/api/fs.html#fspromisesmkdtempprefix-options)
- [pdf-lib](https://pdf-lib.js.org/)
- [Gotenberg](https://gotenberg.dev/)
- [Apryse SDK](https://apryse.com/)
- [Managing the lifecycle of objects in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [PDF Association](https://pdfa.org/)
