# Logistics Password Reset Email Deliverability Through Custom Domain Bounce Handling

Short answer: treat password-reset delivery as a small, security-sensitive data pipeline, not as a single API call. Authenticate a dedicated sending subdomain with DKIM, publish SPF and DMARC deliberately, suppress destinations after definitive failures, and retain only enough provider-neutral evidence to explain a reset attempt. For a logistics support portal that routes contact forms into regional queues, integration effort is lowest when queue routing and mail transport share an internal event contract but do not share provider-specific status codes.

Start with the bill. Assume an illustrative workload of 120,000 reset attempts a month, an average normalized delivery record of 1 KiB, and 90 days of hot retention. The payload footprint is roughly 360 MiB before indexes and replicas: `120,000 x 3 x 1 KiB`. Raw webhook bodies, duplicate events, full message bodies, and high-cardinality logs can outweigh that compact ledger. Transmission charges vary by provider and contract, so they belong in a measured worksheet; the controllable storage term is event multiplication and retention. The useful change is to retain one normalized state history plus a content hash, while expiring raw callbacks quickly.

That choice has a cost during an incident. After raw callbacks expire, operators can prove the accepted transition, timestamp, provider event identifier, and reason class, but cannot replay every original header or newly parse an old vendor field. I would accept that loss only after the parser has contract tests and the normalized record is immutable.

## What should a custom domain password reset email deliverability setup retain?

The durable record should answer a narrow set of questions: did the application accept the request, did the transport accept the message, did a later event report a terminal failure, and was a subsequent attempt suppressed? Store opaque internal identifiers rather than the reset token, message body, contact-form text, or email address in log fields. A keyed digest can correlate repeated attempts without making the address directly readable; key rotation and access policy still matter because pseudonymous identifiers remain operationally sensitive.

Keep less.

Keep authentication evidence separate from per-recipient events. DKIM signs mail on behalf of a domain, SPF authorizes sending infrastructure, and DMARC evaluates domain alignment and policy. Google's sender guidelines require authentication and describe additional requirements for bulk senders; they also call out TLS, DNS, formatting, alignment, and spam-rate expectations. Those controls are prerequisites, not proof that a particular mailbox accepted a particular reset.

Use a dedicated transactional subdomain so changes to support newsletters or route-notification campaigns do not silently alter the identity used for account recovery. The organizational choice is more important than the label: document who owns DNS, selector rotation, DMARC reports, and rollback. A long DKIM key rotation blocked on a separate infrastructure team is an integration defect even if the API itself takes five lines.

| Decision surface | Lower integration effort | Failure mode to test |
|---|---|---|
| Domain authentication | Automated DNS verification with observable status | Stale selector after key rotation |
| Event ingestion | Signed, replayable callbacks mapped at one adapter | Duplicate or out-of-order delivery events |
| Suppression | Internal policy with provider feedback as input | Permanent failure retried indefinitely |
| Regional handling | Explicit processing and retention configuration | EU event copied into an unapproved US log sink |
| Queue routing | Stable internal queue key in metadata | Contact-form category renamed upstream |

## How should bounce and suppression state behave?

Never let the last callback win. Delivery events can be duplicated or arrive out of order, and provider vocabularies differ, so the adapter should map external events into a monotonic internal model. A definitive recipient failure can move an address fingerprint to `suppressed`; a transient failure can schedule a bounded retry under local policy; an unknown event must be quarantined for inspection rather than guessed into a success state.

Consider a design exercise with three callbacks for one opaque message identifier. Callback A says the transport accepted the message, callback B reports a terminal recipient failure, and a delayed duplicate of A arrives last. An implementation that overwrites a row with each new callback ends at `accepted`, precisely the wrong answer; the monotonic transition keeps `terminal_failure`, while the unique event identifier makes the duplicate harmless. Now add a contact-form submission from the same address. That event can route to the correct logistics support queue, but it cannot mutate the suppression record because support intake is neither proof of mailbox recovery nor authorization to resume security mail. The trade-off is explicit: separate state machines add one correlation field and another policy check, but they prevent an operational workflow from weakening an account-recovery control. This isn't a benchmark or an incident report. It is the minimum sequence an acceptance test should reproduce before a transport adapter is trusted.

A compact Python transition function makes the invariant reviewable:

```python
from dataclasses import dataclass
from enum import IntEnum


class Stage(IntEnum):
    REQUESTED = 10
    ACCEPTED = 20
    DELIVERED = 30
    TERMINAL_FAILURE = 40


@dataclass(frozen=True)
class DeliveryEvent:
    message_id: str
    event_id: str
    stage: Stage
    reason_class: str | None


def apply_event(current: Stage, event: DeliveryEvent) -> Stage:
    if event.stage == Stage.TERMINAL_FAILURE:
        return Stage.TERMINAL_FAILURE
    if current == Stage.TERMINAL_FAILURE:
        return current
    return max(current, event.stage)
```

This is intentionally incomplete as a transport adapter. Production ingestion also needs signature verification, an idempotency constraint on `event_id`, an allowlist of recognized reason classes, and a dead-letter path. The point is the ownership boundary: the rest of the application consumes four states, while one adapter absorbs transport-specific churn.

Suppression needs provenance. Record the reason class, first and last event times, the policy version, and any authorized release action. Do not automatically clear a definitive failure merely because a user submits the logistics contact form again; route the support request, but keep account recovery and customer-service workflows distinct.

Stop there.

## Where should US and EU data cross?

"EU support" is not a region design. During evaluation, draw the actual path for the address, delivery event, authentication logs, backups, metrics labels, and support access, then obtain the provider's current contractual and technical details for each hop. The source material does not establish any vendor's residency or legal coverage, so those claims must be verified during procurement rather than inferred from an endpoint hostname.

For the logistics workflow, assign a regional queue key before dispatch. The contact form may route shipment problems to `eu-operations` or `us-operations`, while password resets stay in a security queue; both can use the same correlation identifier without copying message content between systems. Keep observability labels low-cardinality: region, template revision, normalized outcome, and queue key are useful; recipient addresses and free-form contact text are not.

There is a sharp trade-off here. Centralizing every event makes incident queries easier, but expands the data path and the number of operators who can see it. Regional ledgers with aggregated global counters require more deployment work, yet make retention and access boundaries easier to state. Integration effort should include that operational work, not stop at SDK installation.

## A provider trial should try to break the adapter

Run the same acceptance suite against every candidate. Verify custom-domain status and selector rotation, then send to controlled valid and invalid recipients. Replay a signed callback, deliver two events out of order, inject an unknown event type, and confirm that no reset token or message body reaches logs. Disconnect callback ingestion long enough to exercise reconciliation, because a webhook-only design otherwise turns a brief outage into permanent uncertainty.

Then inspect the human path. Can an operator explain one attempt from the normalized ledger without opening a provider console? Can a security reviewer find every retained copy? Can support route a contact-form request without being able to release a suppressed reset destination? These checks expose integration effort that a happy-path API demo hides.

Measure candidates with the same worksheet: engineering days for DNS and callback integration, number of provider-specific fields that escape the adapter, recovery procedure when callbacks are unavailable, supported retention controls, regional data path, and evidence export. Price is one row, based on the team's measured message and event volumes; it is not the architecture.

The deliberate retention policy is therefore small: keep the normalized transition history for the approved investigation window, keep suppression evidence for as long as the policy requires it, aggregate non-identifying service metrics, and delete raw callbacks and message content on short schedules. When a novel parser bug is discovered after raw data has expired, forensic depth is reduced. That is the honest price of collecting less, and it should be signed off before launch.

## Further reading

- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit
