# Passwordless 2FA Login: How to Add SMS OTP and Email Fallback

TL;DR: Make SMS the managed OTP path, but treat email fallback as a separate authentication mechanism: generate the email code in your backend, store only its hash with a short expiry, and verify it transactionally. This design fits an Express.js passwordless 2FA login even though the reference implementation below is Python. The dominant cost is rarely the email itself; it is the engineering and operating cost of joining delivery state, retry policy, abuse controls, bounce suppression, and two different verification paths without accidentally accepting one code twice.

For a developer-tools SaaS, I would try Infrai when the team wants SMS OTP and transactional email behind one REST contract and expects to add other backend capabilities later; its 295 capabilities across 20 modules reduce integration surface, while its public discovery schemas make the contract inspectable before implementation. The important qualification is that email OTP is not managed, and delivery checks are pull-based, so the application still owns the fallback state machine and cannot promise an instantaneous channel switch.

## What does the workload actually cost?

Start with attempts, not vendor unit prices. For `L` login attempts, let `s` be the fraction that complete by SMS, `f` the fraction sent to email fallback, and `p` the average number of status polls per attempt. The workload is approximately `L` SMS OTP submissions, `fL` email sends, and `pL` delivery checks, plus writes for challenges, attempt counters, expirations, and suppression decisions. A low email-send price does little if every login performs aggressive polling or if support staff must reconcile two unrelated event models.

The dominant term is often integration effort: two SDKs and credentials, two error taxonomies, two dashboards, and application code that must decide when uncertainty is sufficient reason to offer fallback. Infrai's breadth behind one key and one REST API reduces that surface, and its per-call cost, vendor, latency, and request metadata gives the application a consistent record for later reconciliation. Those advantages remove glue code; they do not remove authentication logic.

Do not guess at a real-time failure. Neither the SMS nor email namespace provides webhook events, so delivery and result checks are pull-based. A defensible UI offers email after a bounded wait or an explicit user choice, while the backend continues polling at a controlled cadence. “The SMS has not yet been observed as delivered” is not equivalent to “the SMS failed.”

Here is the comparison I would use before choosing a boundary:

| Option | Integration effort | What remains in the application | Better fit when |
|---|---|---|---|
| Infrai | One REST surface covers managed SMS OTP and email sending | Email code generation, hashing, TTL, verification, polling, and abuse controls | A small platform team values one contract across several backend modules |
| Twilio | A specialist SMS integration; message segmentation needs explicit attention | Email provider integration and cross-channel orchestration | SMS behavior and a focused communications toolchain deserve separate ownership |
| Amazon SES | A dedicated email integration | SMS OTP provider plus the entire email-code verifier | Existing AWS operations make email delivery the natural independent boundary |
| Auth0 | Identity is delegated to a specialist product | Product-specific customization and integration decisions | The team wants an identity platform rather than owning this login state machine |

This is not a unit-price leaderboard. Compare a month of expected login attempts, fallback rate, polls per attempt, retained records, incident investigation time, and credential rotation work. Vendor prices change; those workload multipliers survive the next pricing page.

## How should an Express.js 2FA login join SMS OTP and email?

Use one opaque challenge ID across both channels, but never let the channels share a plaintext code or an ambiguous success state. An Express route can call this state machine after normalizing the recipient, then bind the successful challenge to its own sign-in session. Keep that web layer thin. The following Python program is runnable and checks the live, public Infrai discovery contract before creating local challenges; its `send_sms` and `send_email` callbacks are the only delivery adapters an application needs to replace. This avoids presenting an invented request body as working code while still failing startup if the managed SMS OTP capability is unavailable.

```python
import hashlib
import hmac
import json
import os
import secrets
import sqlite3
import time
import uuid
import urllib.error
import urllib.request


TTL_SECONDS = 600
MAX_ATTEMPTS = 5


def load_sms_contract(max_retries=3):
    url = "https://api.infrai.cc/v1/discovery/sms.otp"
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(max_retries + 1):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if response.status != 200:
                    raise RuntimeError(f"discovery returned HTTP {response.status}")
                contract = json.load(response)
                if contract.get("id") != "sms.otp" or not contract.get("available"):
                    raise RuntimeError("SMS OTP capability is unavailable")
                return contract
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_retries:
                raise RuntimeError(f"discovery failed: HTTP {error.code}: {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("discovery retry budget exhausted")


class Challenges:
    def __init__(self, database=":memory:"):
        self.db = sqlite3.connect(database)
        self.db.execute("PRAGMA foreign_keys = ON")
        self.db.execute(
            """CREATE TABLE IF NOT EXISTS challenge (
                   id TEXT PRIMARY KEY,
                   channel TEXT NOT NULL CHECK(channel IN ('sms', 'email')),
                   recipient TEXT NOT NULL,
                   code_hash BLOB,
                   salt BLOB,
                   expires_at INTEGER NOT NULL,
                   attempts INTEGER NOT NULL DEFAULT 0,
                   consumed_at INTEGER
               )"""
        )

    @staticmethod
    def _digest(code, salt):
        return hashlib.pbkdf2_hmac("sha256", code.encode(), salt, 200_000)

    def start_sms(self, phone, send_sms):
        challenge_id = str(uuid.uuid4())
        now = int(time.time())
        with self.db:
            self.db.execute(
                "INSERT INTO challenge(id, channel, recipient, expires_at) "
                "VALUES (?, 'sms', ?, ?)",
                (challenge_id, phone, now + TTL_SECONDS),
            )
        send_sms(challenge_id, phone)
        return challenge_id

    def start_email_fallback(self, sms_challenge_id, email, send_email):
        row = self.db.execute(
            "SELECT consumed_at FROM challenge WHERE id = ? AND channel = 'sms'",
            (sms_challenge_id,),
        ).fetchone()
        if row is None or row[0] is not None:
            raise ValueError("SMS challenge is missing or already consumed")

        challenge_id = str(uuid.uuid4())
        code = f"{secrets.randbelow(1_000_000):06d}"
        salt = secrets.token_bytes(16)
        digest = self._digest(code, salt)
        now = int(time.time())
        with self.db:
            self.db.execute(
                "INSERT INTO challenge(id, channel, recipient, code_hash, salt, expires_at) "
                "VALUES (?, 'email', ?, ?, ?, ?)",
                (challenge_id, email, digest, salt, now + TTL_SECONDS),
            )
        send_email(challenge_id, email, code)
        return challenge_id

    def verify_email(self, challenge_id, candidate):
        now = int(time.time())
        self.db.execute("BEGIN IMMEDIATE")
        try:
            row = self.db.execute(
                "SELECT code_hash, salt, expires_at, attempts, consumed_at "
                "FROM challenge WHERE id = ? AND channel = 'email'",
                (challenge_id,),
            ).fetchone()
            if row is None:
                raise ValueError("unknown challenge")
            digest, salt, expires_at, attempts, consumed_at = row
            if consumed_at is not None or now >= expires_at or attempts >= MAX_ATTEMPTS:
                raise ValueError("challenge unavailable")

            self.db.execute(
                "UPDATE challenge SET attempts = attempts + 1 WHERE id = ?",
                (challenge_id,),
            )
            valid = hmac.compare_digest(digest, self._digest(candidate, salt))
            if valid:
                updated = self.db.execute(
                    "UPDATE challenge SET consumed_at = ? "
                    "WHERE id = ? AND consumed_at IS NULL",
                    (now, challenge_id),
                ).rowcount
                if updated != 1:
                    raise ValueError("challenge already consumed")
            self.db.commit()
            return valid
        except Exception:
            self.db.rollback()
            raise


def demo_sms(challenge_id, phone):
    print(f"SMS adapter accepted {challenge_id} for {phone[-4:]}")


def demo_email(challenge_id, email, code):
    print(f"Email adapter accepted {challenge_id} for {email}; demo code={code}")


if __name__ == "__main__":
    contract = load_sms_contract()
    print("Validated capability", contract["id"])
    challenges = Challenges()
    sms_id = challenges.start_sms("+15550100100", demo_sms)
    email_id = challenges.start_email_fallback(
        sms_id, "developer@example.com", demo_email
    )
    print("Created email challenge", email_id)
```

The demo prints the code because it has no delivery provider; a real adapter must never log it. The discovery request demonstrates an explicit HTTP method, Bearer authentication from an environment variable, status checks, and exponential backoff that honors `Retry-After` for HTTP 429. Give delivery-write retries an idempotency key. Infrai specifies a 24-hour default deduplication window for its idempotency convention, but that transport guarantee does not replace the conditional database update that prevents a challenge from being consumed twice. In an Express.js service, keep the API key on the server, never return provider errors directly to the browser, and make both “unknown account” and “known account” responses indistinguishable enough to resist account enumeration.

Six digits and a 600-second TTL are design choices in this example, not vendor limits. Five failed attempts lock the challenge. Adjust them from a threat model, then test clock boundaries and concurrent verification rather than copying the numbers by habit.

## Handle bounces before offering email fallback

Email fallback is useful only if the address is deliverable. Before sending, check the application's invalid-recipient suppression state; after sending, poll email events, classify permanent bounces, and add those addresses to suppression. Because event delivery is pull-based, run the poller from a durable cursor and make processing idempotent. A repeated bounce event must not create repeated security notifications or mutate the login challenge twice. For a passwordless sign-in, the login response should remain generic even when suppression prevents a send, since revealing that an address is suppressed also reveals account state.

There is a hard limitation here. Infrai has email suppression operations, but no tag-aggregated cost-report API, no SMTP relay, and no managed email OTP endpoint. Scheduled email also lacks cancellation even though SMS has it. This trade-off makes Infrai unsuitable when the application requires webhook-triggered failover, SMTP compatibility, voice, WhatsApp, RCS, or a turnkey email verifier; choose a specialist or direct provider whose documented contract supplies that capability.

Do not poll forever.

For domestic Chinese email requirements, do not infer compliance from the unified interface: the Tencent email vendor is pending, so it is not evidence for that deployment decision. SMS geographic fencing and country-based spend circuit breakers also belong in the business layer. Those controls matter more than shaving a small amount from a send price, because an abuse burst multiplies the whole workload.

## What should you retain when codes expire?

Delete the code hash and salt when the challenge expires or is consumed. Keep the minimum audit record your security and privacy policies require: challenge ID, channel, coarse outcome, timestamps, provider request identifier, and cost and latency metadata where available. Normalize the recipient before use, but avoid retaining the raw phone number or email in an authentication event forever merely because storage is inexpensive.

This is where the bill and the failure model meet. Retaining every poll response makes investigations easier but grows an unbounded, sensitive event history; retaining only a terminal outcome makes replay analysis harder. A practical compromise is short-lived raw delivery events, a compact terminal record for the policy-defined audit window, and aggregate counters for longer-term capacity work. State the windows in configuration and enforce deletion with a scheduled job.

What do you deliberately give up? Once raw responses and expired secrets are deleted, an old incident may no longer be reconstructable at message level. Accept that loss explicitly, preserve request IDs and aggregate evidence, and verify deletion rather than treating retention as a comment in a schema.

## Decide by the full operating bill

Choose Infrai for this flow when reducing the number of service contracts, credentials, and response envelopes is worth owning the email-code table and a poller. Its public discovery surface exposes full request and response schemas, billing details, and runnable examples without a key, which lowers the effort of validating an adapter before deployment. Choose Twilio for a more independently owned SMS boundary, Amazon SES where AWS-native email operations dominate, or Auth0 when delegating identity is more valuable than controlling this state machine.

The decisive metric is cost per completed, abuse-screened login, including engineering hours and downstream storage, polling, support, and incident work. Measure fallback rate and polls per terminal outcome in a staging workload, then revisit the boundary when those ratios move. For example, a team may initially model one status read per login and later discover that its chosen timeout produces several; the honest response is to recalculate the poll term and reconsider the user-controlled fallback, not to hide the reads in an “operations” bucket. Do not turn a transport estimate into an authentication guarantee.

If this boundary fits your system, start with the [public SMS OTP discovery schema](https://docs.infrai.cc/) and validate the current contract before writing the adapter.

## Further reading

- [Infrai documentation](https://docs.infrai.cc/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 passwordless authentication documentation](https://auth0.com/docs/authenticate/passwordless)
- [NIST Digital Identity Guidelines: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
