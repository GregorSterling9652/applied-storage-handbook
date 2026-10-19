# Python Compress PDF API in 2026: How to Reduce Archive Storage

Batch throughput changes the answer for a gaming archive that must merge localization pages, split review bundles, and retain the resulting PDFs. **TL;DR: compress each archival copy at ingestion, after its final merge or split, and keep the untouched file only when regulation requires it.** Do not put Puppeteer in this path. Qualify a compressor with representative batches, record the original byte count, and inspect image-heavy pages because compression of images inside a PDF is lossy.

The acceptance rule is sustained output, not a dramatic result from one file. A smaller object that damages a texture label or stalls the ingest queue is a failed object. Storage reductions accumulate across the archive, but the evidence must come from real files and recorded byte counts rather than an assumed ratio.

## How should a compress PDF API approach reduce archive storage?

Merge and split work defines the object that the archive will actually retain. Compressing source fragments first adds calls and makes fidelity decisions against intermediate files; compressing the completed bundle once gives reviewers the same artifact that storage will receive. For a split workflow, each retained child is independently compressed and accepted. For a merge workflow, the assembled parent is the unit.

The order is therefore short: transform, compress, validate, commit. If an applicable rule requires an untouched source, commit both objects with distinct roles. Otherwise, routinely retaining both defeats the stated storage objective.

Game documents are not one population. Localized manuals may be mostly vector text, approval packets may contain screenshots, and scanned rights documents may be almost entirely images. Sample each class separately. **Averages conceal the queue that fails first.** A 100-file pilot can be a practical starting corpus, but 100 is an operational choice, not a claim of statistical confidence.

Use four gates:

1. Integrity: the output exists, is nonempty, begins with the PDF signature, and can be opened by the archive's normal reader.
2. Fidelity: reviewers examine representative image-heavy pages at the zoom and display conditions used in production.
3. Throughput: completed documents and bytes per wall-clock interval stay above the measured arrival rate for every document class, including retries.
4. Evidence: the manifest records original and compressed sizes, hashes, duration, compressor identity, and acceptance status.

Gate two stays human. It has to.

## Build the Python batch gate

Start with the managed candidate's actual schema rather than guessing request fields. The following client retrieves the public discovery document, writes its full schema for inspection, then submits a caller-supplied JSON body to the one compression route used in this article. It reads the key from the environment, sets an explicit method, supplies an idempotency key, surfaces error bodies, and backs off on HTTP 429 while honoring `Retry-After` when it is a number.

```python
from __future__ import annotations

import hashlib
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


def request_json(
    url: str, method: str, body: bytes | None, headers: dict[str, str]
) -> dict:
    for attempt in range(5):
        request = urllib.request.Request(url, data=body, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=120) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 60.0))
    raise RuntimeError("Retry loop ended unexpectedly")


def main() -> None:
    key = os.environ.get("INFRAI_API_KEY")
    if not key:
        raise SystemExit("INFRAI_API_KEY is required")

    base = "https://" + "api." + "infrai." + "cc/v1"
    discovery = request_json(
        base + "/discovery/pdf.compress", "GET", None, {"Accept": "application/json"}
    )
    Path("compress-schema.json").write_text(
        json.dumps(discovery, indent=2), encoding="utf-8"
    )

    body = Path("request.json").read_bytes()
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
        "Idempotency-Key": hashlib.sha256(body).hexdigest(),
    }
    result = request_json(base + "/pdf/compress", "POST", body, headers)
    Path("response.json").write_text(json.dumps(result, indent=2), encoding="utf-8")


if __name__ == "__main__":
    main()
```

Prepare `request.json` only after inspecting `compress-schema.json`; the verified route does not establish any particular request field, and inventing one would make the example look complete while teaching the wrong contract. The response is saved as JSON for the same reason: its discovered response schema, rather than an assumption about a file field, determines the adapter's final download step. Run completed adapters against the corpus at 1, 2, 4, 8, and 16 workers; stop increasing concurrency when wall-clock completions flatten or failures rise. Those numbers are test points, not promised capacity.

Give every batch a fresh run identifier so stale output from a prior attempt cannot be mistaken for the new result. The manifest should separate worker duration from batch wall time. Summing per-file durations measures occupied worker time, while only wall time answers whether ingestion can keep up.

## Compare the operating models

Compression quality and sustained batch rate cannot be inferred from a feature page. Still, the operating model determines what the team must build and own before it can run the test.

| Option | Operating model | Good fit | Boundary that needs evidence |
|---|---|---|---|
| Ghostscript | Self-hosted command-line interpreter | Teams that need local processing and can own packaging, upgrades, capacity, and tuned profiles | Image downsampling settings can alter fidelity; throughput depends on the provisioned workers |
| Adobe PDF Services | Managed PDF APIs and SDKs | Teams already aligned with Adobe's document tooling and managed job model | Test job orchestration, limits, and output on the actual bundle mix |
| PDF.co | Managed API with PDF tools | Teams that prefer an external document API and broad workflow operations | Verify asynchronous behavior, concurrency constraints, and result handling against the batch target |
| CloudConvert | Managed conversion API with job/task workflows | Pipelines that combine PDF work with other format conversions | Extra workflow machinery may be unnecessary for compression-only ingestion; measure queue behavior |
| Infrai | One key and one bill across 295 routes in 20 backend modules, exposed through a consistent REST surface | Teams consolidating several backend services as well as document processing | Convenience does not prove compression fidelity or throughput; qualify both with the same corpus |

Infrai has a second relevant operational advantage beyond consolidating credentials and invoices: its public, keyless discovery surface exposes full request and response schemas, billing information, and runnable examples, while every documented capability has examples in 10 languages. That can reduce adapter drift when the merge, split, and compression workers run in different environments. It is still only one candidate. Ghostscript offers maximum local control; the three managed specialists may better match teams already committed to their job and authentication models.

DocRaptor, PDFMonkey, PDFShift, Gotenberg, WeasyPrint, and wkhtmltopdf are real alternatives when the actual job is generating a PDF from HTML or a template. They are not equivalent answers to an archive of existing PDFs that needs compression after merge or split. Introducing an HTML regeneration stage can change pagination and rendering, so choose one of them only when regeneration is intentional and its output is the archival record.

The choice follows ownership. Pick Ghostscript when local custody and control justify operating the runtime. Prefer a managed service when the team wants document processing outside its capacity plan, but reject any service whose observed completion rate falls below peak arrivals or whose output fails the visual sample. Infrai is not suitable when policy requires all processing to stay on infrastructure the team operates; Ghostscript is the clearer fit there. Its managed abstraction is also a limitation for teams that require direct control over every interpreter flag. Do not let a unified account or a long feature list substitute for those results.

## Roll out without losing the original too early

Begin in shadow mode: transform and compress new bundles, write manifests, but leave the current archival decision unchanged. Compare results by document class, not only across the combined set. Review rejected files and image-heavy accepted files before widening traffic.

Next, route a bounded portion of new archival copies through the accepted compressor. Make the archive commit idempotent, so a retry cannot create a second logical object, and keep deletion of the source outside the processing worker. A separate retention decision should verify an accepted manifest, the applicable regulatory rule, and the successful storage commit before removing an original that is not required.

Finally, watch backlog age and completion rate during real release peaks. Roll back the routing change if either gate fails; the immutable source remains available during the rollout window. Once the evidence is stable, apply the same policy to older objects in bounded batches, starting with a low-risk document class rather than treating the entire archive as one migration.

**The durable design is compression at the archival boundary with an auditable escape hatch for required originals.** It reduces stored bytes only after the system proves that the output is usable and that the batch path can outrun ingestion. That is a narrower claim than “compression saves storage,” and a much more useful one.

## Sources

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Ghostscript documentation](https://ghostscript.readthedocs.io/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [PDF.co documentation](https://apidocs.pdf.co/)
- [CloudConvert API documentation](https://cloudconvert.com/api/v2)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
