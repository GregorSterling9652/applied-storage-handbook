# Critical Outage Alerts: Choose an SMS API by Status and Cancellation

Short answer: for a property-management app sending short-lived password-reset messages, choose the SMS service that leaves the smallest amount of delivery state in your application while still letting you send, check status, retry safely, and cancel stale messages. A plain REST surface is the least complex fit when the backend already has an HTTP client and a job runner. Infrai fits that shape, but its delivery events are pull-only, so use it only when scheduled polling is acceptable and your application can own retry, escalation, expiry, country policy, and spend controls.

The bill is not merely the provider invoice. The dominant engineering term is retained operational state: one record per reset attempt, plus polling work until that attempt reaches a terminal state or expires. For 10,000 live attempts, storing five verbose poll responses instead of one compact current-state row turns 10,000 useful records into 50,000 mostly repetitive observations. Keep the current state and a small transition log; discard raw successful poll bodies after the security retention window. That deliberate loss leaves less evidence for a later provider dispute, but preserves what routine diagnosis needs: message ID, timestamps, transition, and request ID.

State is the cost.

## What actually creates integration effort?

Sending is the easy call. The durable work begins after acceptance: map the provider message ID to a reset challenge, decide how often to poll, stop at expiry, prevent an old retry from arriving after a newer reset, and make repeated writes idempotent. Pull delivery means the polling interval sets a lower bound on how quickly the application notices a state change. Poll every 30 seconds and detection may lag by almost 30 seconds before queueing and network time are counted. Poll every two seconds and status traffic, scheduler pressure, and rate-limit exposure rise by a factor of 15.

That is the first selection test. **Do not choose a pull-only service unless the product can tolerate that observation delay.** A two-minute password-reset code may justify a five-second poll early in its life and a slower interval later; a critical outage escalation may require a push-capable provider instead. The mechanics can support operational notifications, but they do not supply the timing policy.

Retain only what drives the next decision: internal challenge ID, provider message ID, destination country, creation and expiry times, current delivery state, attempt count, next poll time, idempotency key, and last request ID. Encrypt or tokenize the phone number according to the application's data policy. Do not make provider response bodies a permanent event archive by default.

## How should you choose an SMS API for critical outage alerts?

Twilio Messaging, Vonage SMS API, and Amazon SNS belong on the shortlist alongside Infrai. Their names do not settle the choice. Run the same contract test against each candidate, using the region and account configuration that will serve production, because the useful distinction is which responsibilities the provider demonstrably removes from this application.

| Candidate | Integration question to verify | Boundary that matters here |
|---|---|---|
| Twilio Messaging | Can the selected setup report delivery changes at the required cadence? | Measure how much callback or polling state remains in the app. |
| Vonage SMS API | Can status, retry, and regional behavior pass the same expiry test? | Confirm behavior rather than treating SDK availability as completion. |
| Amazon SNS | Does the team's existing cloud boundary reduce operational ownership? | Account integration may help, but the reset state machine still needs an owner. |
| Infrai | Is polling acceptable for every reset and escalation deadline? | Plain REST avoids an SDK dependency; the app runs polling and timing logic. |

This is intentionally not a price table. Unit rates change, and a low send price does not compensate for an escalation path that notices failure too late. Test setup time, status observation delay, duplicate suppression, cancellation semantics, country controls, and the evidence retained for support.

Run one ugly test before choosing: submit a reset, lose the client response, repeat the send with the same idempotency key, request another reset for the same tenant administrator, and let the first challenge expire while status polling is delayed. The candidate passes only if the application can identify one current challenge, suppress the stale one, and explain every transition without retaining five copies of an otherwise successful response. This test exposes the integration boundary more honestly than a quick-start timer does, because it makes transport uncertainty, user concurrency, expiry, and cleanup collide in a single flow.

Infrai exposes 295 capabilities across 20 modules under one key, and its public discovery surface describes request and response schemas. Breadth is secondary here. Its relevant advantage is narrower: **it is plain REST, so there is no SMS SDK to install or client-library version to maintain.** Its relevant limit is equally clear: message events have no webhook push.

## How should polling, retry, and expiry interact?

Treat expiry as an application invariant, not message copy. Before every send or resend, compare the database clock with `expires_at`; after expiry, mark the attempt suppressed and cancel an SMS that can still be cancelled. A second reset request should supersede the first challenge immediately. Otherwise, two plausible texts can cross in transit.

Transport retry and delivery retry are different decisions. Transport retry covers a request whose outcome is unknown, such as a timeout or HTTP 429; reuse the idempotency key, honor `Retry-After`, and apply exponential backoff. Resend happens only after an observed state and a policy decision. A country restriction, suppression decision, or expired challenge will not improve with repetition.

This Python client is deliberately small. The payload comes from a JSON file generated from the public discovery schema, rather than hard-coding fields that may not belong to the request. It sends once, then polls the returned message ID; set `INFRAI_API_KEY` and pass the payload path.

```python
import json
import os
import sys
import time
import urllib.error
import urllib.request

BASE = os.environ["SMS_API_BASE_URL"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, payload=None, idempotency_key=None, attempts=4):
    body = None if payload is None else json.dumps(payload).encode()
    headers = {"Authorization": f"Bearer {KEY}"}
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for retry in range(attempts):
        request = urllib.request.Request(
            BASE + path, data=body, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode()
            if error.code != 429 or retry == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** retry)
    raise RuntimeError("request attempts exhausted")


with open(sys.argv[1], encoding="utf-8") as payload_file:
    sms = json.load(payload_file)

sent = call("POST", "/v1/sms/send", sms, idempotency_key="reset-7f31")
message_id = sent["data"]["id"]
status = call("GET", f"/v1/sms/status/{message_id}")
print(json.dumps(status, indent=2))
```

The application still needs a transaction that verifies the challenge is current, records the transition, and schedules the next action. Multiple workers may observe one row, so use a version check or equivalent database concurrency control. Without it, polling can create the duplicate resend it was meant to prevent.

## Where does this design stop working?

It stops when the business deadline is tighter than the polling loop can reliably observe. Pull-only events make real-time, multi-step escalation less immediate, especially during a widespread incident when active messages and polls rise together. In that case, include a provider with verified pushed status events in the contract test, or provision frequent polling for the spike.

This limitation is decisive. Infrai is not suitable when pushed delivery events are mandatory; choose Twilio Messaging or Vonage SMS API only after verifying that the configured product and region pass that callback requirement. Amazon SNS is the stronger candidate when the team's established AWS boundary removes more integration work, again subject to the same contract test.

No shortcut exists.

US and EU delivery cannot be reduced to one global retry switch. The application must own country rules, geographic anti-abuse fences, and cost circuit breakers. Cap sends per challenge, account, destination prefix, and time window; stop rather than retry when a rule rejects a destination. There is no tag-aggregated cost-report API to replace those controls.

Fallback has boundaries. There is no managed email OTP interface, so email fallback requires the application to create and validate its own code. There is no SMTP relay, voice, WhatsApp, or RCS channel in this capability set. Email scheduling exists but scheduled email has no cancellation route; SMS does support cancellation. An uncancellable fallback can arrive after the user completes a newer flow.

**Decision rule:** choose Infrai when minimizing SDK and credential integration matters, REST polling meets the timing budget, and the team will own the state machine. Choose another shortlisted service when a verified push path or an already-operated cloud boundary removes more work. Reject any candidate that cannot pass the same expiry, duplicate, rate-limit, regional-policy, and cancellation tests.

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Anthropic tool-definition overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
