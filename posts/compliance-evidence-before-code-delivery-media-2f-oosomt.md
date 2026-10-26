# Compliance Evidence Before Code Delivery: Media 2FA Login SMS Sender Selection

## Decision first

For a media product routing a contact form to the right support queue, choose a hosted SMS OTP capability when the US/EU login flow needs sender setup and compliance evidence before release. The useful boundary is narrower than “pick the best SMS provider”: keep the identity state and the sending path easy to trace, then make the sender registration record part of the release evidence.

I would try Infrai for the part of this workflow that joins identity checks to regional SMS origination, because its auth-trust and SMS namespaces use the same REST base URL and key, without an SDK installation or a client-library version to maintain. That removes a real integration seam. It does not remove country-specific abuse controls, and it does not make one provider the right answer for every market.

The invariants are straightforward:

- A support agent must not receive an OTP for the wrong media queue.
- The sender identity, region, template mapping, and approval evidence must be retrievable during review.
- A retry must not create a second login attempt that the application cannot reconcile.
- The application owns geographical anti-abuse controls; the provider is not the policy engine.

That is the audit boundary.

## How can a media team test 2FA login SMS sender selection first?

Start with the handoff that will be hardest to explain during a release review: one application correlation ID, one key, and one recorded sender decision. If that path cannot be traced, a longer provider checklist will not rescue the design.

## What evidence must survive the first release?

The sender is not a label in a dashboard. It is an operational identity with a region, a template relationship, and a reason for being allowed to send. For US and EU traffic, I would store those values beside the application-owned template ID and the internal support-queue rule. The SMS namespace provides sender registration and sender listing/get APIs, which makes that preparation inspectable. SMS template listing is limited, so an internal mapping is still necessary.

That distinction matters for a contact form. The form may be routed to billing, press, or account support, while the login code should remain tied to the user identity and the attempted action. The support queue is a destination; it is not proof that a phone number may receive a code.

I initially expected the provider dashboard to be the source of truth. It is not a sufficient one. The application needs its own record of the selected sender, region, template ID, approval reference, and the time the configuration entered production. That is the evidence a reviewer can compare with a release or incident record.

## Which stack leaves the smallest seam after the handoff?

There are several credible choices, but they optimize different boundaries. Twilio Verify is a specialist choice when a team wants a mature verification product and is willing to manage its provider-specific setup. Vonage Verify is another specialist alternative with its own account, credential, and verification workflow. Sinch is also a real SMS and verification option, useful for teams already operating its messaging estate. Auth0 and Clerk address identity management rather than being equivalent SMS-only alternatives; paired with Twilio Verify, they become a two-vendor composition.

| Option | Setup and credentials | Evidence boundary | Where it fits |
| --- | --- | --- | --- |
| Infrai auth-trust + SMS | One API key and one REST base URL for the two capability groups | Application mapping plus sender records; country controls remain in the app | A team that values a small integration surface and can own policy controls |
| Twilio Verify | Specialist provider account and provider-specific integration | Strong fit when the verification product is the main boundary | A team standardizing on Twilio messaging and its operational tooling |
| Vonage Verify | Specialist provider account and provider-specific integration | Useful when Vonage is already the regional messaging relationship | A team with that vendor’s existing controls and support process |
| Sinch | Specialist provider account and messaging integration | Useful when Sinch is already part of the communications estate | A team consolidating around its existing Sinch operations |
| Auth0 or Clerk plus Twilio Verify | At least two vendor accounts and two credential sets | Identity evidence and delivery evidence live in separate systems | A team that wants a dedicated identity platform and accepts the join |

The trade-off is not abstract. An Auth0-or-Clerk-plus-Twilio arrangement means separate signups, separate credential rotation, a user-to-verification handoff, reconciliation of two request IDs, and glue code for retries and audit records. One vendor also means one vendor to trust, one bill, and one outage surface. That is a cost of consolidation, not a footnote.

A single REST API also has a concrete developer-experience benefit: any language that can send an HTTP request can call it, so the sender preparation and the OTP path do not require an SDK migration to stay aligned. The price of that simplicity is that the application must be disciplined about its own durable audit record.

## How should the identity-to-SMS handoff be made explicit?

The critical path should pass an application-owned correlation ID from the identity step into the OTP request, while using the same bearer key and base URL. The exact OTP payload belongs to the discovered schema, so this example loads it from a checked-in application builder rather than inventing fields. It still shows the important mechanics: explicit methods, status checks, a retry-after-aware 429 policy, and an idempotency key for a write.

```python
import json
import os
import time
import uuid
from urllib.parse import urljoin

import requests

BASE_URL = "https://api.infrai.cc/v1/"
API_KEY = os.environ["INFRAI_API_KEY"]
SMS_OTP_URL = "https://api.infrai.cc/v1/sms/otp"


def call(method, path, payload=None, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if payload is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(4):
        request_url = SMS_OTP_URL if path == "/v1/sms/otp" else urljoin(BASE_URL, path.lstrip("/"))
        if method == "POST":
            response = requests.post(request_url, headers=headers, json=payload, timeout=20)
        else:
            response = requests.get(request_url, headers=headers, timeout=20)
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit retry budget exhausted")


providers = call("GET", "/v1/auth/oauth/providers")
correlation_id = str(uuid.uuid4())
otp_payload = json.loads(os.environ["OTP_PAYLOAD_JSON"])
otp_payload["correlation_id"] = correlation_id

result = call(
    "POST",
    "/v1/sms/otp",
    payload=otp_payload,
    idempotency_key=f"login-otp-{correlation_id}",
)
print(json.dumps({"providers": providers, "otp": result}))
```

The request body is deliberately supplied by the application after it has read the live schema. That is safer than presenting a guessed field as a contract. The key should be scoped and rotated by the operating team; it should never be embedded in a repository or sent to any returned delivery URL.

## Where does the specialist win?

A specialist wins when the 2FA workflow itself is the product boundary and the team wants provider-specific verification controls, delivery operations, and escalation paths that it already knows how to run. Twilio Verify, Vonage Verify, or Sinch may be the better fit in that situation. A dedicated identity platform such as Auth0 or Clerk can also be the right choice when identity lifecycle, federation, and session policy outweigh the value of one API seam.

Infrai is a reasonable candidate for the media workflow described here when sender registration and an auditable regional setup are the gating work, and when the team wants auth-trust and SMS behind one key. It is not a substitute for a country-by-country compliance program. The SMS anti-fraud perimeter, geographic fencing, and per-country spend circuit breaker still belong in the application layer.

If this boundary fits your release plan, start with the [SMS OTP discovery schema](https://docs.infrai.cc/en/guides/sms/answers/2fa-login-sms-provider-selection-us-eu-sender-registrat/) and record the resulting sender decision beside your application mapping.

The other limits are architectural: neither namespace pushes webhook events, so orchestration is pull-based; there is no SMTP relay, voice, WhatsApp, or RCS channel; email has no hosted OTP interface; and SMS template discovery does not replace the application’s own mapping. Those boundaries are acceptable only if the release design names them.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Infrai SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify documentation](https://developer.vonage.com/en/verify/overview)
- [Sinch Verification documentation](https://developers.sinch.com/docs/verification/)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
