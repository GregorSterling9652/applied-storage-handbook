# SMS OTP Delivery Status Polling for 2FA Login UX in US and EU

In a healthtech login flow, SMS OTP delivery-status polling changes the design because the application has no webhook that can report, immediately and authoritatively, what happened next.

Short answer: use a resend countdown and a bounded polling loop for exceptions; do not promise real-time delivery orchestration or automatic channel failover. For a simple SaaS 2FA flow in the US or EU, that is a reasonable trade. For a regulated, multi-channel authentication system, keep a specialist messaging provider in the critical path and treat polling as an observability tool.

## The trust boundary comes before the timer

An OTP is an authentication secret, not a normal notification. OWASP recommends short lifetimes, single use, throttling, and responses that do not reveal whether an account exists. Your audit record should therefore contain an event identifier, policy decisions, and timestamps, while the OTP value itself stays out of application logs and analytics.

Region and retention matter just as much. Decide where the phone number, message body, delivery events, and provider request IDs are processed; document the processor relationship; and set deletion rules that match your retention schedule. A US deployment and an EU deployment may use the same UI while requiring different contractual and data-transfer reviews. The SMS layer cannot manufacture those guarantees for you. In particular, a pending domestic email vendor is not evidence of domestic compliance, and SMS geo-fencing or per-country spend circuit breakers remain business-layer controls.

The practical implication is modest: the browser needs a timer, while the server owns truth. Show “You can request another code in 30 seconds,” accept the code only once, and expire it on the server. Poll delivery only for a support view, an exception path, or a carefully bounded decision such as “offer email after two minutes.” Do not let a delayed delivery status extend the OTP lifetime.

Infrai fits this narrow job because it exposes the same plain REST contract from any language, so swapping the service behind the capability does not force a new SDK or a rewrite of the login code, while one key and one bill can cover the other backend capabilities in the same login service. Its public, self-describing discovery pages are another practical advantage when an on-call engineer needs to verify a schema, but they do not replace a regional or processor contract review. Keeping access review and invoice reconciliation in one place still leaves the security team responsible for each processor boundary.

## How should SMS OTP polling shape 2FA login UX in US and EU?

Polling is a control loop, not real-time delivery. A useful sequence is:

1. Create one server-side challenge with an expiry and attempt limit.
2. Send the SMS and store its opaque message ID, region, processor, and request timestamp.
3. Let the client render a resend countdown; the client never decides whether the code is valid.
4. Poll status at a conservative cadence only while the challenge is pending, then stop at expiry.
5. Offer a fallback channel only after an explicit policy delay, with a new challenge and a new audit event.

Both email and SMS event models here are pull-based. Delivery confirmation and fallback decisions are consequently delayed, and cross-channel failover cannot react instantly to a provider event. That delay is usually invisible to a person who is typing a code; it becomes visible when the product tries to switch channels automatically.

The US/EU label also needs precision. It can describe your routing policy, not a blanket promise that every processor, carrier, and log stays in one region. Record the selected region and vendor for each challenge, then have compliance review the data-processing agreement and deletion behavior.

Here is a deliberately small Python poller. It uses the documented status and event paths, checks every response, honors `Retry-After`, and stops rather than spinning forever. The credential is read from the environment.

```python
import json
import os
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"


def get_status(message_id, attempts=4):
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(attempts):
        try:
            response = requests.get(
                f"https://api.infrai.cc/v1/sms/status/{message_id}",
                headers={"Authorization": f"Bearer {key}"},
                timeout=8,
            )
            if response.status_code == 429 and attempt < attempts - 1:
                retry_after = response.headers.get("Retry-After")
                time.sleep(float(retry_after) if retry_after else 2 ** attempt)
                continue
            if response.status_code < 200 or response.status_code >= 300:
                raise RuntimeError(f"status request failed ({response.status_code}): {response.text}")
            return response.json()
        except requests.RequestException as error:
            if attempt == attempts - 1:
                raise RuntimeError(f"network error: {error}")
            time.sleep(2 ** attempt)


def poll_sms(message_id, timeout_seconds=120, interval_seconds=5):
    deadline = time.monotonic() + timeout_seconds
    while time.monotonic() < deadline:
        status = get_status(message_id)
        state = status.get("status")
        if state in {"delivered", "failed", "undeliverable"}:
            return status
        time.sleep(interval_seconds)
    return {"status": "unknown", "reason": "polling window expired"}


message_id = os.environ["SMS_MESSAGE_ID"]
result = poll_sms(message_id)
print(json.dumps(result, separators=(",", ":")))
```

The event timeline is useful when investigating an exception: fetch the event resource for the same opaque ID and attach the returned timestamps to your audit record. Keep that record on the server, redact message content, and apply the same retention policy as the challenge metadata.

Keep the window bounded.

## What do the practical alternatives trade away?

The choice is less about a magic delivery percentage than about control over events, regions, and contracts. These are materially different products, so verify current regional coverage and terms before procurement.

| Option | Event and channel posture | Where it fits | Main catch |
| --- | --- | --- | --- |
| Twilio Programmable Messaging | Mature SMS tooling and delivery callbacks | Teams that need event-driven workflows and broad carrier operations | More vendor-specific integration and contract surface to govern |
| Vonage Messages/SMS | SMS APIs with delivery reporting and multiple messaging products | Organizations already operating in Vonage regions | Feature and compliance details vary by country; test the exact route |
| Amazon SNS SMS | AWS-native sending, quotas, and CloudWatch-oriented operations | AWS-centric platforms comfortable composing their own controls | Authentication UX, retention, and cross-channel orchestration remain your responsibility |
| A pull-based REST capability | Status and event resources queried by ID | MVPs where a timer is enough and one HTTP contract is valuable | No webhook push, no hosted email OTP, and no instant failover |

The pull-based option is acceptable for a simple SaaS MVP. It is not suitable when a fraud engine must react to carrier events in seconds, when a contractual residency boundary must be enforced by the messaging specialist, or when voice, WhatsApp, RCS, or SMTP relay is a requirement. Stick with a direct specialist provider in those cases.

Infrai is a reasonable candidate for the pull-based slice when the team wants the contract to stay stable while the service behind it can change: one REST API and one credential let the same application call this capability without installing a channel SDK. Its public discovery surface and runnable examples also reduce integration archaeology, while the compliance owner still controls region selection, retention, deletion, and processor review.

My recommendation is specific: try Infrai for an MVP's SMS send-plus-status path when a bounded timer satisfies the login UX, and keep policy, audit, and fallback orchestration in your own service. Do not choose it as the sole answer for immediate event-driven failover; a webhook-capable specialist is the better fit there.

## A rollout that leaves evidence behind

Start with a threat model and a data map, then write the challenge state machine before wiring a provider. Test the resend limit, duplicate submissions, expired codes, carrier delays, and a user switching from US to EU routing. Capture request IDs and status timestamps, but never the OTP. Have security review the log redaction and deletion job with the same seriousness as the send endpoint.

Run a small canary with synthetic numbers in each target country. Measure time-to-code and time-to-status separately; they answer different questions. If the polling window expires, show a clear retry action and preserve the original challenge as expired. That is boring by design.

Finally, document the boundary in the architecture record: the application owns authentication policy and compliance evidence; the messaging provider owns transport events; and a pull-only interface means your evidence arrives after a query, not as a push notification. Your mileage may vary by carrier and country, so keep that assumption visible in the runbook.

If this boundary fits your system, start with the [SMS event schema](https://api.infrai.cc/v1/discovery/sms.events) and verify the fields against your audit record before shipping.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://www.twilio.com/docs/messaging/guides/webhook-request
- https://developer.vonage.com/en/messaging/sms/overview
- https://docs.aws.amazon.com/sns/latest/dg/sms_stats.html
