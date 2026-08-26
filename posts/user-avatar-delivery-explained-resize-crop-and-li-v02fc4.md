# User Avatar Delivery Explained: Resize, Crop, and Lifecycle Validation with Python

Short answer: for social marketplace avatars, keep one validated master for a deliberately limited period, generate a small fixed set of crops from that master, and publish a new immutable version only after decoding, moderation, and output validation all pass.

Start with the bill, because it exposes the architecture. Avatar cost is the sum of uploaded-byte retention, derived-image retention, transformation work, moderation work, delivery, and operational records. In a deliberately simple capacity model, 1,000,000 accepted uploads averaging 3 MiB produce about 2.86 TiB of masters before replicas or metadata; keeping each master for 30 days instead of indefinitely places a hard bound on that term. This is arithmetic, not a workload benchmark, and each team should replace both inputs with observed distributions. The important move is to identify which term grows with every upload and every day, then bound it before arguing about codecs or vendors.

The least complex design is usually enough: accept an upload into a private staging area, decode it, normalize orientation, produce a canonical square master, run image and OCR-based moderation against that stable representation, and promote versioned derivatives only when the decision is final. Don't let a public URL point at staging data. At the end of the retention window, deliberately delete the original upload while keeping the approved canonical master and delivery variants. The catch is real: after deletion, a newly improved decoder or moderation model cannot reprocess the source pixels, and an appeal may have less evidence. A marketplace with long dispute windows should retain originals longer; an app with strict data-minimization requirements may choose a shorter window and accept that loss of reversibility.

## How should social apps resize, crop, and validate user avatar delivery?

Treat resize, crop, and lifecycle validation as separate decisions. Resizing chooses an output pixel budget. Cropping chooses what content survives. Lifecycle validation determines whether any output is eligible to become public. Combining them in one opaque upload callback makes failure analysis harder: a rejected face crop can look like a decode failure, while a moderation rejection can leave a derivative cached under a URL that appears valid. For the crop, define a deterministic policy. A centered square crop is predictable and easy to reproduce, but it can remove a face or text near an edge. A subject-aware crop preserves the detected subject more often, yet its result can change when the detector changes. For a social marketplace, that change matters because OCR moderation coverage may depend on which lettering remains inside the avatar. Moderate the canonical representation and every materially different public crop; otherwise a clean center crop can approve an image whose alternate wide crop still exposes prohibited text.

Formats deserve the same skepticism. A filename extension is a hint, not validation. Decode the bytes with a maintained image decoder, reject content the decoder cannot identify, and encode outputs into a format the delivery clients are meant to consume. MDN's media format guide documents that browser support and format characteristics differ, so a single fashionable output format isn't a universal compatibility policy. A conservative fallback is valuable when the supported-client matrix is uncertain.

I'm not sure one crop can ever represent every marketplace category well; the evidence needed is a review set sampled by category, language, device, and rejection reason. Until that exists, prefer a small, explicit derivative set over arbitrary client-selected dimensions. Fewer variants reduce retention and invalidation surface area, while a review set tells you whether the simplification damages moderation coverage.

## The cost model should decide what survives

Use a per-upload ledger rather than a single monthly total. The ledger should record source bytes, canonical bytes, derivative bytes, transformation count, moderation count, and delivered bytes by avatar version. That separates a retention problem from a popularity problem: stored bytes grow with accepted versions and time, while delivery grows with views. It also prevents a noisy account from hiding a policy that creates too many derivatives.

| Decision | Cost term it changes | What can go wrong | Suitable default |
| --- | --- | --- | --- |
| Keep every original forever | Master retention | Unbounded accumulation and a larger sensitive-data footprint | Use only when disputes or reprocessing justify it |
| Keep a time-bounded original | Master retention | Reprocessing and appeals lose source pixels after expiry | Set the window from policy, not convenience |
| Generate arbitrary dimensions | Transform and derivative retention | Variant explosion and inconsistent moderation coverage | Publish a fixed derivative set |
| Regenerate on every request | Transform work and latency | Repeated work during traffic spikes | Cache immutable, versioned outputs |
| Replace a mutable URL in place | Invalidation and delivery | Old content may remain visible in caches | Change the version in the object key |

The dominant term will vary. If avatars are viewed heavily, delivered bytes may outweigh retained derivatives; if users upload repeatedly but receive few views, masters and moderation work may dominate. Your mileage may vary — measure p50, p95, and maximum source size separately, because an average conceals the files most likely to exhaust decoder memory or stretch processing time. No public source can supply those workload-specific distributions.

Publication comes last.

Promotion is a state transition, not a successful resize. The durable states can be `staged`, `validated`, `approved`, `published`, `rejected`, and `expired`. Only `published` has a public delivery key. Retrying work must preserve that invariant, even when the same job is delivered twice.

## A small Python boundary keeps publication honest

The following Python sketch makes the ordering visible without binding the pipeline to a storage, moderation, or image-processing product. Implementations provide the decoder, moderator, and object repository; the coordinator owns the rule that public promotion happens last. It also validates the actual encoded output rather than trusting the requested width and height.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class ImageInfo:
    width: int
    height: int
    media_type: str


class ImageCodec(Protocol):
    def decode(self, content: bytes) -> ImageInfo: ...
    def square_cover(self, content: bytes, size: int) -> bytes: ...


class Moderator(Protocol):
    def approves_image_and_text(self, content: bytes) -> bool: ...


class AvatarRepository(Protocol):
    def stage(self, upload_id: str, content: bytes) -> None: ...
    def publish(self, object_key: str, content: bytes) -> None: ...
    def reject(self, upload_id: str) -> None: ...


def publish_avatar(
    *,
    user_id: str,
    upload_id: str,
    version: int,
    source: bytes,
    codec: ImageCodec,
    moderator: Moderator,
    repository: AvatarRepository,
) -> str | None:
    repository.stage(upload_id, source)
    source_info = codec.decode(source)
    if source_info.width < 256 or source_info.height < 256:
        repository.reject(upload_id)
        return None

    canonical = codec.square_cover(source, size=1024)
    if not moderator.approves_image_and_text(canonical):
        repository.reject(upload_id)
        return None

    delivery = codec.square_cover(canonical, size=256)
    delivery_info = codec.decode(delivery)
    if (delivery_info.width, delivery_info.height) != (256, 256):
        repository.reject(upload_id)
        return None

    object_key = f"avatars/{user_id}/v{version}/256"
    repository.publish(object_key, delivery)
    return object_key
```

The numeric dimensions are example policy values, not universal limits. A real coordinator should also cap input bytes and decoded pixels before expensive work, make `publish` idempotent for the versioned key, and emit a reason code at every rejection boundary. Keep the reason internal; a public response should not reveal enough moderation detail to help an uploader tune around the checks.

Versioned keys make rollback and cache behavior legible. The profile record should point to the active avatar version, so an approved replacement becomes one metadata update rather than an overwrite of a mutable image. Keep the previous approved version only if product policy needs rollback. Otherwise, expiry should remove superseded derivatives as well as the staged original.

## Failure modes and the decision rule

The nastiest failure is partial publication: one size becomes visible before another size fails moderation or validation. Prevent it by writing all outputs under a private version prefix, validating the complete set, and then switching the profile's active version. A second failure is moderation drift between crops; preserve the moderation policy identifier and crop policy identifier with each version so a later audit can distinguish changed rules from changed pixels. A third is retry duplication. Stable upload IDs and version-scoped keys make repeated work converge on the same result instead of creating new public objects.

Use the fixed pipeline when clients can accept a small derivative set, crop policy is centrally owned, and moderation must cover every public representation. It is not suitable when customers require arbitrary art direction for each placement, when originals must remain available for legal evidence, or when offline clients need formats outside the chosen compatibility matrix. In those cases, retain the source under stricter access controls, store crop metadata rather than only cropped pixels, or negotiate derivatives as part of the client contract.

Ship the policy only after a replay test can take the same source, version, crop policy, and encoder configuration and produce an equivalent moderation decision. Also test malformed inputs, unusually large decoded dimensions, duplicate jobs, rejected OCR text, expiry, rollback, and a cache still holding the previous version. Observe counts and latency at each state transition, plus bytes retained by lifecycle class. Pretty thumbnails are not evidence.

The decision rule is compact: minimize the number of public representations, moderate every representation that can differ materially, publish by immutable version, and retain source pixels only as long as the stated recovery or dispute policy requires. This favors explainability over speculative flexibility. Good.

## References

- [MDN: Media formats for HTML audio and video](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)

## Further reading

The MDN media formats guide above is the primary compatibility reference for choosing browser-facing encodings; pair it with measurements from the app's actual client and upload population before changing the derivative policy.
