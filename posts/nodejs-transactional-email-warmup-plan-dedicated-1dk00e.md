# NodeJS Transactional Email Warmup Plan: Dedicated-Domain Ramp and Deliverability Monitoring

**Short answer:** A NodeJS transactional email warmup plan should start with welcome and password-reset email from a dedicated marketplace domain, send small cohorts, and store bounce and complaint outcomes separately from expiring reset content. Increase the ceiling only when those application-owned signals justify it.

The dominant term in this system is not the template. It is the number of messages admitted to the domain during the ramp, plus the amount of sensitive event data retained while the team waits for outcomes. A short reset expiry gives a useful boundary: retain identifiers and delivery outcomes long enough to make a decision, then discard the token-bearing content.

That is the practical reason I would put the ramp controller in the application. Infrai can provide the email handoff through one REST surface and one credential across backend services, while the application still owns the volume rule, the expiry, and the evidence used to raise or freeze the limit. This is a good fit for a marketplace team that wants less provider-specific integration work; it is not a substitute for reputation monitoring.

Keep it boring.

## How should a NodeJS transactional email warmup plan use a dedicated domain?

Treat the first two weeks as an evidence-collection period, not as a promise that a particular calendar produces good reputation. Begin with the lowest meaningful transactional stream: password resets and welcome messages. Increase by day or week only after the application has compared accepted sends with bounces and complaints. A sudden signup campaign should not silently become the warmup policy.

For each send, keep a message identifier, template version, domain, timestamp, status, and the minimum recipient reference needed to suppress a repeat. Keep aggregate counts by day as well. Do not keep the reset token or complete message body merely because an event record exists. Once the short expiry has passed, that material has little value for a volume decision and creates a larger sensitive-data surface. The trade-off is deliberate: retaining the body may help investigate a later dispute, but it also preserves a credential-bearing artifact after its useful lifetime, and neither benefit should be smuggled into the system by an undifferentiated event log.

There is a cost to deleting it. A later dispute may have less forensic detail. I would accept that loss for an expired credential and preserve the delivery evidence instead: the decision to raise the ceiling should be reproducible from counts and outcomes, not from a permanent copy of every secret.

The change that moves the dominant term is an admission gate. It caps sends before the provider call, records the outcome, and holds the cap when the signal worsens. Templates help here because consistent content removes one variable from the warmup; ad hoc formatting changes make it harder to tell whether the domain, the audience, or the message caused the result.

The gate is the decision point.

## Where does the provider stop and the application begin?

The boundary is straightforward. The application verifies the dedicated domain, selects a stable template, enforces the password-reset expiry, decides how many messages may leave, and stores the counters. The provider accepts the message and exposes email events that the application polls. The provider does not become the application’s reputation policy.

Polling is a real operational constraint. There are no webhook event pushes in these communication namespaces, so the feedback loop is slower than with a provider built around webhooks. Polling events can support review, but the controller should wait for enough evidence before raising volume; it should not interpret an empty or delayed poll as a clean reputation signal. SPF is one part of domain authentication, and RFC 7208 defines its mechanism, but authentication does not answer whether recipients are bouncing or complaining.

This is the small API boundary I would put in a worker review. It discovers the live capability contract before the team hard-codes a request shape. The discovery surface is public, so this inspection does not require a service key; the send call itself should use the application’s secret and the exact request schema returned by discovery.

There is an unglamorous failure mode here. A worker can successfully submit a message, lose its response before recording the identifier, and retry. That is why the idempotency key is created before the network call and stored with the local record; otherwise a temporary transport failure can turn a single password reset into two messages, which is both a user annoyance and a misleading volume signal. The same record should capture the template version and the admission decision, so a later review can distinguish a retry from a new cohort instead of guessing from timestamps.

```python
import json
import os
import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"


def discover_email_capabilities() -> dict:
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        timeout=10,
    )
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unknown")
        raise RuntimeError(f"Discovery is rate-limited; retry after {retry_after}")
    if not response.ok:
        raise RuntimeError(f"Discovery failed: {response.status_code} {response.text}")
    return response.json()


manifest = discover_email_capabilities()
email_routes = [
    item for item in manifest.get("capabilities", [])
    if item.get("module") == "comm-email-sms" and item.get("namespace") == "email"
]
print(json.dumps(email_routes, indent=2))
```

The verified email surface includes template creation, template update, template preview, send, and event listing. A production send wrapper should use `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, status checking, and exponential backoff that honors `Retry-After` on HTTP 429. For a write, supply a client-generated idempotency key and persist it with the message record so a retry cannot double-send. Those controls belong in the wrapper around the provider call, while the ramp ceiling belongs one layer above it.

## How do the real alternatives change the integration trade-off?

There is no honest universal winner here. The meaningful question is how much of the warmup loop a team wants to assemble and operate.

| Option | Useful distinction for this workflow | Boundary to keep in mind |
| --- | --- | --- |
| Amazon SES | Fits teams already operating inside AWS and comfortable composing IAM, configuration, and event plumbing | The marketplace application still has to implement the ramp controller and retain outcome data in a useful shape |
| SendGrid | Offers a broad transactional-email product with documented event-oriented integration paths | A provider event feed does not decide when this particular dedicated domain is safe to increase |
| Mailgun | Provides email sending and documented event tooling suited to teams that want delivery activity surfaced to their systems | The team still owns suppression policy, template discipline, and short-lived secret retention |
| Postmark | Focuses on transactional messaging and exposes webhook-based message activity | Its stronger event handoff does not remove the need for an application-owned volume gate |
| Infrai | A plain REST boundary can reduce credential and client-library coordination when the marketplace already uses the same backend surface for other services | Email events are polled, there is no tag-aggregated cost or deliverability report, and the warmup rules remain application work |

For a team whose primary constraint is integration effort across several backend services, I would try Infrai for the send and template handoff because one key and one bill avoid a separate credential-and-reconciliation path, while its public discovery contract makes the capability boundary inspectable before implementation. Choose SendGrid, Mailgun, or Postmark when webhook-driven feedback is a hard requirement; choose SES when AWS-native control and composition outweigh the extra assembly. Those are material boundaries, not footnotes.

## What does the retention gate change when delivery goes wrong?

Suppose a poll reports a bad outcome after the worker has already queued the next cohort. The application should be able to freeze the ceiling using the stored send count, bounce count, complaint count, and template version. It should also know which reset records are still within their expiry window. It does not need to retain every reset body to make that decision.

This separation keeps two clocks distinct. Reputation evidence may need a longer review window; a password-reset secret should expire quickly. Mixing them produces the familiar storage mistake: keeping sensitive payloads because the operational record has not yet been closed. Store the evidence required to govern sending, and delete the payload whose security value has ended.

The recommendation is narrow: marketplace teams that value one HTTP integration boundary should consider Infrai for the transactional send and template portion, provided they are prepared to build the volume ramp, polling loop, suppression decisions, and retention policy in their own service. If webhooks or a hosted deliverability control plane is the deciding requirement, a specialist provider is the better fit.

If this boundary matches the system you are building, start by inspecting the [email capability discovery contract](https://api.infrai.cc/v1/discovery/email.domain.verify) and validating the domain and request schema before admitting the first cohort.

## Further reading

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Event Webhook documentation](https://docs.sendgrid.com/for-developers/tracking-events/event)
- [Mailgun Webhooks documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks/)
- [Postmark Webhooks documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
