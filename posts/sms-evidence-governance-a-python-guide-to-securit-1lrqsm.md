# SMS Evidence Governance: A Python Guide to Security Notifications and OTP Fallback

A healthtech password-reset code should expire quickly; the explanation of why the application accepted or rejected it may need to survive much longer. That difference in lifetimes is the constraint that changes the choice of a simple SMS alert provider for US/EU security notifications.

TL;DR: choose transactional SMS transport only after defining the evidence that will remain when the reset secret is gone. Keep authorization decisions in the authoritative application state, assess OTP verification services separately from delivery services, and require a reviewed processing and retention contract before calling any candidate suitable for healthtech. Basic SMS alerts and code flows are a narrower requirement than a complete multi-channel authentication system.

Deletion matters here.

The selection axis is compliance evidence: what can a reviewer reconstruct without recovering a patient's reset code, phone number, or message body? A delivery receipt answers a transport question. It doesn't establish who authorized a password change, and a vendor's channel list doesn't establish the contractual basis for processing that message.

## What should an SMS provider retain for security notifications and OTP verification?

Start with a proposed 300-second validity window, explicitly an application policy rather than a provider default. Record when the application issued the challenge, the deadline it assigned, the policy version it applied, and the decision it eventually committed. Give the reset attempt an opaque identifier that can correlate transport observations without copying patient details into every downstream record. These are proposed application fields; they are not parameters to any vendor API.

Keep the secret separate.

For a storage architect, the difficult question is where these records acquire authority. If the account mutation commits but the evidence export fails, an exported object is an incomplete account of the transaction; if an export succeeds before the account mutation rolls back, it may describe a change that never happened. Commit the authorization decision and its evidence together where the application supports that transaction, then export a projection for review. Where that cannot be done, define a reconciliation procedure and mark the export's completeness explicitly. Object retention alone cannot repair an ambiguous source record.

There are three different lifetimes to approve: the challenge's usable lifetime, the correlation record's retention period, and the lifetime of the link from an opaque account reference back to a person. Don't silently make them equal. A pseudonymous identifier can remain sensitive when another system can resolve it, so a secret-free export still needs access controls and a justified retention policy. The retention duration belongs to the deployment's review; this design supplies no universal legal deadline.

The trade-off is deliberate: retaining less message content reduces what a later review can inspect, but retaining complete bodies can preserve reset secrets and unnecessary patient context. Keep a template or policy revision reference when that meets the review requirement, and obtain approval for any additional content. Before comparing products, request evidence of processing locations, subprocessors, contractual scope, deletion behavior, and available event exports for the actual US/EU deployment. These are acceptance questions, not claims that a candidate has already passed them.

## Make the export boundary executable before adding a sender

Inspect the transport contract before connecting it to patient data. This Python 3 script reads public discovery and counts SMS capability declarations without sending a message or printing a route inventory. Set INFRAI_API_BASE to the documented HTTPS API base including its version prefix, and INFRAI_API_KEY to the deployment key. Discovery is public; the example includes environment-based Bearer authentication to make credential handling explicit. It uses an explicit GET, surfaces HTTP errors, and retries HTTP 429 with exponential backoff while honoring Retry-After. Because it is read-only, retrying cannot duplicate a reset message.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base = os.environ["INFRAI_API_BASE"].rstrip("/")
if not base.startswith("https://"):
    raise ValueError("Use the documented HTTPS API base")
request = Request(
    base + "/discovery",
    headers={
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
        "Accept": "application/json",
    },
    method="GET",
)
for attempt in range(5):
    try:
        with urlopen(request, timeout=15) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            manifest = json.load(response)
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)

capabilities = manifest["capabilities"]
if isinstance(capabilities, str):
    capabilities = json.loads(capabilities)
sms_count = sum(item["namespace"] == "sms" for item in capabilities)
print(json.dumps({"declared_sms_capabilities": sms_count}))
```

The 15-second HTTP timeout and five-attempt limit are local choices, not delivery guarantees. A capability declaration is an inspection starting point, not proof of contractual suitability. Review the selected capability's full request and response schema before implementing a write adapter, including its declared idempotency behavior.

The next boundary is a projection of the evidence, because a transport adapter should not decide which patient data enters a long-lived record. This separate, local example uses only the standard library. Its synthetic input represents a delayed delivery observation: the message was observed after the application's 300-second deadline. The program checks the proposed deadline policy, exports an explicit field allowlist, and preserves the distinction between transport observation and authorization. It neither sends a message nor verifies a code.

```python
import json

POLICY_SECONDS = 300
EXPORT_FIELDS = (
    "attempt_ref",
    "policy_version",
    "issued_at_s",
    "expires_at_s",
    "decision",
    "delivery_observed_at_s",
)


def export_evidence(record):
    issued = record["issued_at_s"]
    expires = record["expires_at_s"]
    if type(issued) is not int or type(expires) is not int:
        raise ValueError("Issue and expiry times must be integer seconds")
    if expires - issued != POLICY_SECONDS:
        raise ValueError("Deadline does not match this policy version")
    if record["policy_version"] != "reset-short-v1":
        raise ValueError("Unknown policy version")
    evidence = {name: record[name] for name in EXPORT_FIELDS}
    observed = evidence["delivery_observed_at_s"]
    if type(observed) is not int:
        raise ValueError("Observation time must be integer seconds")
    evidence["observation_after_expiry"] = observed >= expires
    return evidence


sample = {
    "attempt_ref": "synthetic-reset-a",
    "policy_version": "reset-short-v1",
    "issued_at_s": 0,
    "expires_at_s": 300,
    "decision": "expired_without_redemption",
    "delivery_observed_at_s": 312,
    "internal_note": "Synthetic fixture; omit from the export",
}
print(json.dumps(export_evidence(sample), indent=2, sort_keys=True))
```

All timestamps here are synthetic seconds relative to issuance. The 312-second observation is a fixture, not a latency measurement, and the allowlist excludes the internal note instead of serializing the entire input record. Production input validation, authenticated access, durable storage, and the actual redemption transaction remain separate implementation responsibilities. This small example makes one boundary inspectable; it isn't an authentication service or a compliance certification.

Observation isn't delivery time.

With polling, a delivery event can occur before a worker sees it. Preserve the distinction between a provider event timestamp, where available, and the application's observation timestamp; do not interpret the fixture's late observation as proof of late delivery. For a proposed 30-second polling interval, observation can lag an available update by nearly one interval before scheduling and network delays are considered. That is a design consequence, not measured provider performance. Webhook-based messaging stacks are worth evaluating when that gap changes the fallback decision, although authenticated events, duplicate handling, and reconciliation still need owners.

Resends must preserve the approved expiry policy. A transport retry should not quietly restart authorization; an intentional replacement challenge needs a recorded relationship to the superseded attempt. Email fallback likewise needs an explicit challenge and redemption policy, including the rule for a late SMS arriving after an email has already been used. A useful failure-mode name is evidence drift: the transport trail reports a retry while the verifier treats it as a new grant. Prevent that disagreement before optimizing delivery speed.

## Compare the processing chain, then the product surface

The candidates below occupy different places in the reset workflow. Treat the table as a procurement map, with questions to resolve through the linked product documentation and deployment review, rather than a certification ranking. Comparing an email sender with a verification service as interchangeable SMS APIs would hide the ownership decision.

| Candidate | Role in the decision | Trade-off and evidence to examine |
| --- | --- | --- |
| Twilio Verify | Verification-oriented candidate for OTP workflows | Evaluate delegated challenge management against the required expiry and redemption policy; obtain the event and processing evidence needed to explain its decisions. |
| Amazon SNS | Messaging-oriented candidate for transactional SMS | Evaluate transport records while keeping challenge state and password mutation under explicit application ownership; sending is a smaller responsibility than verification. |
| Resend | Email candidate for a fallback branch | Evaluate email evidence separately from SMS observations; introducing an email sender still requires a shared authorization and retention policy. |
| Amazon SES | Alternative email candidate for an AWS-oriented deployment | Compare responsibility for the email evidence pipeline with the SNS branch; a shared infrastructure ecosystem does not settle contractual or retention requirements. |

Infrai offers 295 routes across 20 modules through one REST API, one key, and one bill, so adding a backend capability can reuse a consistent integration contract instead of introducing another SDK and credential inventory. Its public discovery exposes request and response schemas without requiring a key, giving reviewers a concrete contract to inspect before access is granted; consistent per-call metadata, including request_id and vendor, can help correlate a transport attempt with the application evidence. SMS sending, OTP verification, and resend support fit alerts and basic code flows. Evaluate that scope with polling-based event observation, application-owned email-code verification, and application-level geographic restrictions and country-based spending cutoffs. This comparison does not establish healthcare contractual suitability, processing jurisdiction, or retention guarantees for that option.

The narrower scope is useful when the application already owns challenge state and wants a consistent backend surface. For delegated verification, evaluate the verification-oriented candidate first; for email fallback, compare the email candidates against the evidence needed for that branch. The simplest API is an advantage only after its processing chain is acceptable.

Several requirements should be separated from this basic SMS-and-email decision: SMTP relay, voice, WhatsApp, and RCS need their own channel assessment. This reset flow makes no assumption that a scheduled email can be canceled; the verifier must reject expired authorization regardless of what arrives later. Likewise, retain application-owned template revisions and any tag-based cost aggregation required for review, rather than making the evidence design depend on a particular template-list or reporting API. No unverified regional provider availability should count as compliance evidence.

## Roll out one evidence policy before migrating traffic

Begin with synthetic reset attempts and review the exported records with the people responsible for retention and account security. Exercise a duplicate transport observation, an observation after expiry, a replaced challenge, and concurrent redemption attempts; require one authoritative authorization outcome and an explanation of any unresolved transport state. Synthetic fixtures establish expected behavior, not a delivery benchmark.

Then enable a limited cohort under the approved processing contract, retaining the previous adapter for a controlled rollback. During migration, keep the policy version, expiry semantics, and export fields stable across adapters so a reviewer can compare records without reverse-engineering two definitions of success. Expand only after reconciliation and deletion have both been demonstrated. A reset message that arrives is useful; an authorization decision that remains explainable after its secret is deleted is the acceptance condition.

## Sources

References for candidate roles and further deployment review; inclusion here does not assert a healthcare certification or contractual approval:

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
