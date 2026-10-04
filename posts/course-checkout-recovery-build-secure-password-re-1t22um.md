# Course Checkout Recovery: Build Secure Password Reset Flow with Hashed Tokens

TL;DR: In a Node.js and Express password-reset flow, reliability must never weaken security: generate a high-entropy opaque token, store only its digest with an expiry, consume it in the same database transaction that changes the password, and enqueue email through a transactional outbox. Return the same response for every reset request, apply limits by account and network, and treat delivery as an observable attempt rather than proof that a person received the message. For an edtech checkout, the order receipt still originates from the settled-payment event; account recovery merely restores access to that receipt and must not replay payment or receipt creation.

The hard constraint is ownership. Token state, expiry, single use, login revocation, and the learner-to-receipt mapping live in the application data model. Mail transport records an attempt to move a message. Confusing those boundaries creates the dangerous cases: a sent link with no committed token, two valid links racing, an expired link that becomes valid after a retry, or a password change that leaves old login grants alive.

## How should Node.js build a secure password reset flow?

Issuing a reset spans a database and a mail system, so a single ACID transaction cannot honestly cover the whole operation. Commit the reset row and an outbox row together, then let a worker deliver the message. If the worker crashes after sending but before acknowledging the outbox record, it may send the same link twice. That is acceptable only because both messages refer to one server-side record and successful consumption makes every copy useless. Exactly-once email is the wrong promise; single-use state is the enforceable invariant.

Duplicates happen.

Use a cryptographically secure random token. A practical policy is 32 random bytes, encoded for URLs, with a 20-minute lifetime; those numbers are design choices, not a standard mandate, and the threat model should determine them. Put the raw token only in the outbound URL. Store a SHA-256 digest, the user identifier, creation and expiry timestamps, a consumed timestamp, and enough outbox metadata to retry safely. Never log the raw query string.

The reset request handler should perform a constant-shaped lookup and always return a generic message such as "If the account exists, an email will be sent." OWASP recommends consistent responses and timing to reduce account enumeration. Perfectly identical timing is difficult once queues and databases are involved, so measure the distributions rather than adding a fixed sleep and assuming the problem is solved.

## Model the ledger before the route

The storage rule matters more than Express syntax. The following Python model makes the state transition explicit; a Node implementation should map the same transaction to its database driver and keep the Express controller thin. The compare-and-set update is the critical line.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
import hashlib
import secrets


@dataclass(frozen=True)
class IssuedReset:
    raw_token: str
    token_digest: bytes
    expires_at: datetime


def issue_token(expires_at: datetime) -> IssuedReset:
    if expires_at.tzinfo is None:
        raise ValueError("expires_at must be timezone-aware")
    raw = secrets.token_urlsafe(32)
    digest = hashlib.sha256(raw.encode("ascii")).digest()
    return IssuedReset(raw, digest, expires_at)


def consume_reset(db, raw_token: str, new_password_hash: str) -> bool:
    digest = hashlib.sha256(raw_token.encode("ascii")).digest()
    now = datetime.now(timezone.utc)
    with db.transaction() as tx:
        reset = tx.find_reset_for_update(digest)
        if reset is None or reset.consumed_at is not None or reset.expires_at <= now:
            return False
        changed = tx.mark_consumed_if_unused(reset.id, consumed_at=now)
        if changed != 1:
            return False
        tx.replace_password_hash(reset.user_id, new_password_hash)
        tx.revoke_login_grants(reset.user_id, revoked_at=now)
    return True
```

A unique index on the digest catches accidental duplication, but it does not establish single use by itself. The conditional update, password replacement, and login revocation belong in one transaction. Under concurrent submissions, row locking or a predicate such as `consumed_at IS NULL` must allow exactly one update to affect one row. Test that race with two independent database connections; an in-memory mock cannot demonstrate isolation behavior.

Delete or invalidate older outstanding resets when issuing a new one if the product rule is "latest link wins." Alternatively, allow several outstanding rows and consume all of them after one succeeds. Both can be secure. The second approach is kinder to delayed mail, while the first reduces the live-token set; document the choice because users will otherwise experience it as random failure.

This design has real limitations. A transactional outbox adds a table, a worker, cleanup, monitoring, and a small delivery delay; it is a poor fit when the datastore cannot provide the required transaction or conditional update. A low-volume internal system may accept synchronous dispatch and its narrower operational footprint, provided it also accepts the failure window between database commit and mail submission. A managed queue can remove worker maintenance, while a database-backed worker keeps the reset and outbox commit together. Neither choice repairs weak token state, and neither guarantees inbox arrival.

| Decision | Safer default | Failure mode to test |
|---|---|---|
| Token persistence | Digest only | Database disclosure turns live rows into usable links |
| Expiry check | Server-side UTC timestamp | Clock boundary accepts an old token |
| Consumption | Conditional update inside password transaction | Two requests both succeed |
| Dispatch | Transactional outbox | Commit succeeds but no message is queued |
| Request response | Same body and status | Account existence leaks through content or timing |
| Login handling | Revoke existing login grants | A stolen login grant survives recovery |

## How does email delivery fail?

A queue acknowledgment means the worker accepted work. An SMTP acceptance means a receiving system accepted responsibility for the message. Neither means the learner opened it. Keep those states separate: `queued`, `attempted`, `accepted`, `temporarily_failed`, and `permanently_failed` are operational evidence, not token states. The token ledger remains authoritative even if a delayed message arrives after expiry.

Acceptance is not delivery.

For the payment path, use a different idempotency key and event type. A `payment_settled` event should create one immutable receipt record and enqueue its receipt email; a password-reset request must never invoke that handler. Once the learner regains access, the application reads the existing receipt. This separation prevents a recovery retry from mutating financial history.

Authentication of the sending domain also affects delivery. SPF publishes which hosts may send for a domain, DKIM signs selected message content, and DMARC defines domain alignment and reporting policy. They do not make a reset token secure, and a passing result does not prove inbox placement. Use a dedicated return path, process bounces, and expose queue age plus permanent-failure counts to operators. Do not place email addresses or tokens in metric labels.

Delivery retries need limits. Retry transient transport failures with bounded exponential backoff and jitter, stop on permanent failures, and stop when the token is too close to expiry to be useful. A dead-letter state must retain the outbox identifier and sanitized reason while excluding the raw link.

One short alert is better than a dashboard that quietly turns red.

## Abuse controls without account leaks

Rate limiting belongs at several scopes because each key has a blind spot. An account limit slows repeated mail to one learner but can reveal account state if responses differ. An IP or network limit slows broad scans but can punish a school NAT. A global circuit breaker protects the mail queue during an attack but affects everyone. Apply all three conservatively, return the same external response, and keep the internal reason in restricted telemetry.

Start with explicit policy values, then tune from observed legitimate traffic and abuse. For example, a team might permit three accepted requests per account in 30 minutes while using a larger per-network budget; this is an example configuration, not a universal security constant. Count requests even when the address is unknown so the observable behavior stays aligned. CAPTCHA or step-up checks can follow suspicious volume, but they should not become the sole control.

Measure first.

URLs deserve equal suspicion. Construct the origin from trusted configuration, never from the incoming `Host` header. Permit HTTPS, place the token in a query parameter consumed by the reset page, set a restrictive referrer policy, and avoid third-party assets on that page because referrer and analytics collection can disclose URLs. After validation, accept the new password through a POST and do not change state on the GET that opens the email link.

## Roll out with evidence

Ship the ledger and outbox behind a narrow flag, then exercise expiry boundaries, duplicate worker delivery, concurrent consumption, unknown accounts, mail delay, bounced addresses, and login revocation. The acceptance test for concurrency is precise: two simultaneous valid submissions produce one password change and one generic failure. The receipt test is equally precise: recovery activity cannot create a second receipt or alter the settled order.

Watch request volume by coarse network bucket, outbox age, retry count, acceptance and permanent-failure rates, time from request to accepted handoff, successful consumption, and expired-token attempts. Keep identifiers out of logs, restrict access to reset records, and define retention for consumed and expired rows. The useful rollout decision is not "did email send?" It is whether the complete chain preserves its invariants under delay, duplication, and concurrency.

**A reliable recovery flow is a state machine with an email side effect.** Once that distinction is enforced in storage, Express becomes an adapter rather than the security boundary, and the edtech receipt remains tied to payment settlement instead of to whichever recovery message arrived last.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://nodejs.org/api/crypto.html
- https://expressjs.com/en/guide/behind-proxies.html
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc9110
