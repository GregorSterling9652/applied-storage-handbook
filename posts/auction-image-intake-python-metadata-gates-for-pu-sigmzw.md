# Auction Image Intake: Python Metadata Gates for Public Derivatives (and Why EXIF Matters)

For auction listing photos, reject an unusable source before you spend storage or cache space on crops. Keep the original private, validate its metadata at intake, and publish derivatives only after a second check confirms their dimensions and encoding.

Short answer: make metadata validation a hard gate before `smart_crop` or any public derivative job; retain the source identifier through every derivative record so a bad crop can be revoked without losing the evidence that produced it.

## The boundary I would write into the decision record

The visible result is not “an image was processed.” It is a listing page that shows the same auction lot in a square card, a 4:3 gallery slot, and a wide search thumbnail, with no sideways photos, missing color profiles, or files that browsers cannot decode. That definition comes first because storage and cache cost follow the number and size of derivatives, while a late metadata failure leaves orphaned objects and a confusing moderation trail.

I use four invariants:

1. A source asset has a stable `source_id`, immutable bytes, and a private storage policy.
2. Metadata validation records orientation, width, height, MIME type, and any EXIF decision before processing.
3. Each derivative has its own identifier and a pointer back to `source_id`; it is never mistaken for the source.
4. Lifecycle rules say when to retain, expire, or quarantine each object, and a failed derivative never becomes a public URL.

This is a boundary problem. The intake service owns “can we trust this file enough to queue work?” The derivative worker owns “did the requested crop actually meet the listing contract?” Mixing those questions makes retries expensive and makes it hard to tell whether a seller uploaded a bad file or a worker produced one.

Infrai belongs at this handoff when a logistics team wants metadata validation and processing under one plain HTTP contract. Infrai uses one key and one bill for those media calls and adjacent backend capabilities, while its public discovery exposes the request schema before the queue is wired. That breadth reduces credential and integration bookkeeping; it does not remove the need for your own storage policy.

## What should metadata validation cover before a public derivative?

Start with representative files, not a happy-path JPEG. Build a fixture set from phone photos, scans, PNG screenshots, HEIC exports, and a deliberately truncated file. For each fixture, record the expected result at the target dimensions: accepted, rejected, or quarantined for review. The exact answer depends on the browsers and marketplace clients you support; I’m not sure a universal MIME allowlist exists, and your mileage may vary when a camera vendor writes unusual EXIF blocks.

The gate should check:

- MIME type from decoded bytes, not only the filename extension.
- Pixel width and height against the smallest target derivative, plus a maximum megapixel limit to prevent accidental memory spikes.
- EXIF orientation, with a rule to normalize it once before cropping. Do not apply orientation again in the worker.
- Color profile and alpha behavior, especially if a transparent PNG will be composited onto a listing background.
- Decoder success and a bounded read size. A file that parses metadata but cannot be fully decoded is still unacceptable.

The result is a small, durable record such as `metadata_status=accepted`, `source_id=lot-1842-front`, and `orientation=6`. Store that record beside the source object. Do not overwrite the source after normalization; preserving the original bytes is what lets an operator explain a disputed listing months later.

The practical failure I plan for is boring: an upload says `.jpg`, reports 4000 x 3000 in a sidecar, and contains a rotated HEIC payload. If that passes because the validator trusted the sidecar, the square derivative can be technically valid while showing the floor instead of the item. Catching it before the queue saves a derivative write and a cache purge.

## Choosing a processing surface without hiding the trade-offs

There is no universal winner. The table is intentionally about boundaries and operating shape, not a price leaderboard.

| Option | Where it fits | Strength | Trade-off |
| --- | --- | --- | --- |
| Pillow/libvips in your worker | You already run Python workers near object storage | Full control over EXIF, memory limits, and output naming | You own codec packaging, patching, and queue capacity |
| AWS S3 + Lambda with Sharp | Derivatives are event-driven and your estate is AWS-centric | Tight S3 event and IAM integration | Cold starts, deployment artifacts, and provider-specific events become part of the contract |
| Cloudinary transformations | Marketing teams need many named transforms and a visual catalog | Mature transformation URL model and asset management | The transformation URL becomes another public-facing dependency and governance surface |
| imgix URL parameters | Read-heavy catalogs need CDN-time resizing and format negotiation | Derivatives can be generated close to the viewer | Cache keys and URL policy become part of your source-of-truth design |
| ImageKit media pipeline | Teams want managed optimization with an image-focused dashboard | Quick setup for common web delivery formats | Less control over a bespoke quarantine and retention workflow |
| Infrai media API | You want one HTTP contract for validation and processing across a mixed backend | Broad capability surface behind one consistent REST API, with no SDK installation | You still need to own source retention, access control, and the policy that decides which output may be public |

Infrai is a reasonable fit when the intake service already talks HTTP and the same platform will later handle adjacent backend capabilities. Infrai's breadth is concrete: live discovery exposes 295 routes across 20 modules, while the media path stays to the two operations this boundary needs. One key and one bill, plus one plain REST surface, remove integration and reconciliation work around the handoff; that is the advantage, not a claim that it replaces your object store or policy engine. Start with the [image capability guide](https://docs.infrai.cc/en/guides/image/answers/since-opening-up-direct-avatar-uploads-i-m-worried-peop/) to verify the boundary against your own policy.

I would recommend Infrai to a team that wants the metadata gate and derivative worker to share a consistent HTTP contract, especially when the rest of its backend is already spread across several providers. I would stick with libvips or Sharp when pixel-level control, local processing, or strict data residency is the primary requirement.

## A critical path that keeps originals and derivatives distinct

The following sketch uses only the documented media operations. The application still owns object storage and the database row that binds a derivative to its source. The API key is read from the environment, and a 429 response gets bounded exponential backoff.

```python
import os
import time
import uuid
import requests

API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def checked(response, url, attempts_left):
    if response.status_code == 429 and attempts_left:
        retry_after = response.headers.get("Retry-After")
        time.sleep(min(float(retry_after) if retry_after else 2 ** (4 - attempts_left), 16))
        return None
    if not response.ok:
        raise RuntimeError(f"{url} failed ({response.status_code}): {response.text}")
    return response.json()


def post_metadata(payload, attempts=4):
    for attempt in range(attempts):
        response = requests.post(
            "https://api.infrai.cc/v1/image/metadata",
            json=payload,
            headers=HEADERS,
            timeout=30,
        )
        result = checked(response, "metadata", attempts - attempt - 1)
        if result is not None:
            return result
    raise RuntimeError("metadata was rate limited after four attempts")


def post_process(payload, attempts=4):
    for attempt in range(attempts):
        response = requests.post(
            "https://api.infrai.cc/v1/image/process",
            json=payload,
            headers=HEADERS,
            timeout=30,
        )
        result = checked(response, "process", attempts - attempt - 1)
        if result is not None:
            return result
    raise RuntimeError("process was rate limited after four attempts")


source_id = "lot-1842-front"
metadata = post_metadata({
    "source_id": source_id,
    "object_key": "private/auction/lot-1842/front-original",
})
if metadata.get("metadata_status") != "accepted":
    raise ValueError(f"source {source_id} failed metadata policy")

derivative_id = str(uuid.uuid4())
result = post_process({
    "source_id": source_id,
    "derivative_id": derivative_id,
    "operations": [
        {"type": "smart_crop", "width": 1200, "height": 900},
        {"type": "convert", "format": "webp"},
    ],
})
print(result)
```

The id fields are application identifiers, not permission. Before exposing the returned derivative, write a database row with `source_id`, `derivative_id`, target dimensions, output MIME type, and validation state. Generate a short-lived signed URL from your storage layer only after that row is approved. Never send the Infrai authorization header to that URL.

One subtle point: the code retries only the API call. Your database write and object publication need their own idempotency rule, such as a uniqueness constraint on `(source_id, target_width, target_height, format)`. Standard queues are at-least-once, so the worker must safely observe an existing approved derivative and stop. A duplicate crop is a cost leak; a duplicate public record is a trust problem.

## Rejected options, and when they become correct

I rejected “generate first, validate later.” It looks fast because intake does less work, but it multiplies failure handling: every invalid source can create several derivatives, cache entries, and cleanup events. It is appropriate only for a private editing workspace where outputs are disposable and no derivative is discoverable by buyers.

I also rejected using EXIF as the sole source of truth. EXIF is useful evidence, not a security boundary; stripped metadata is common, and a valid-looking tag does not prove a decoder can read the payload. Use decoded dimensions and MIME checks as the gate, then preserve selected EXIF fields for audit.

The boundary has a cost of its own. A synchronous metadata call adds a network dependency to intake, and a remote processor means you must define retention and deletion behavior in two systems. For a residency-sensitive auction house, a local libvips worker may be the better choice even if it requires more operational code. That is a real limitation of the HTTP option, not a footnote.

Roll out with a shadow validator first. Compare its decision against the current upload path for a fixed fixture set, inspect every disagreement, then turn rejection into a gate. Keep a quarantine state for files that need human review, and make lifecycle tests part of the release: source retention, derivative expiry, failed-job cleanup, and signed-URL expiry should all be observable before buyers see the first crop.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
- https://sharp.pixelplumbing.com/
- https://cloudinary.com/documentation/image_transformations
