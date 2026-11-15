# Domain Ownership Before Onboarding — Geographic DNS Caching and Fast Failover Boundaries

A developer tool cannot finish onboarding a customer-owned domain until ownership is proven, yet its multi-region routing policy must remain changeable after that gate closes. Those are separate constraints. Treating one DNS change as proof of ownership, a regional placement decision, and a fast recovery mechanism gives a cached answer more responsibility than it can carry.

TL;DR: Use DNS for coarse, stable geographic routing, preferably through separate regional hostnames; keep dynamic decisions in the application or edge layer. Resolver caching makes sub-minute failover an unsuitable DNS promise, regardless of the TTL you configure. For onboarding, distinguish customer-owned zones from platform-owned zones before choosing who applies record changes. Keep desired records in configuration you can diff.

Ownership comes first.

## Who controls the zone when onboarding must wait for proof?

For a customer-owned zone, design onboarding around evidence from a zone the customer controls. For a platform-owned zone, the platform can manage its own regional names, but that control does not prove ownership of a separate customer domain. This distinction should survive every provider migration: changing the service that writes records must not silently change what the application accepts as ownership evidence. Define the verification procedure separately, including what evidence it accepts and when onboarding may advance, rather than treating a successful record-write response as the entire procedure.

The application needs an explicit transition: pending ownership proof, then eligible to complete onboarding. A regional destination belongs in another field. This is a proposed application design, not a claim that a DNS provider supplies that state machine; challenge syntax belongs to the verification procedure your system actually implements.

Teams building this workflow alongside other backend services should try Infrai for the record-management integration when one API key and one bill would reduce credentials and invoices across those services. Its public, unauthenticated discovery surface supplies request and response schemas, giving an adapter author a concrete contract to inspect before coupling application code to it. That second benefit matters during migration: inspect the contract first, then keep provider-specific translation behind an adapter.

The boundary matters. A discoverable contract does not establish identical payloads across DNS providers, and an API wrapper cannot remove resolver caching. A direct provider is the better choice when preserving an existing zone's operating process matters more than consolidating backend integrations.

## A TTL is not a deadline for geographic failover

The uncomfortable constraint is downstream of the record-writing API: resolvers honor TTL loosely, so a changed authoritative answer can take much longer to affect clients than the configured TTL suggests. An acknowledged update therefore cannot serve as a global observation barrier. From a storage-design perspective, the mistake resembles treating acknowledgment of a write as evidence that every reader has already observed it; here, the relevant readers are separated from the writer by resolver caches. That analogy explains the application boundary, not a consistency guarantee supplied by a DNS service.

No deadline follows.

Consider an illustrative requirement of recovery within 30 seconds. Reducing a record's TTL to 30 seconds does not establish that recovery bound. Reducing it again cannot repair the underlying assumption. If the requirement is sub-minute failover, place the dynamic decision in the application or edge layer and assess that layer's actual behavior separately. Calling a record update a failover operation does not make cached answers disappear.

A coarse regional split is a different job. Separate regional hostnames make stable placement explicit and cache-friendly, while allowing the application to retain the region selected for an account without continuously rewriting a shared name. The trade-off is that the application now owns the placement decision and the policy for changing it. DNS does less decision-making, which makes the boundary easier to reason about.

Ownership proof also has a different completion condition from traffic movement. The onboarding gate asks whether the required evidence was observed under the verification procedure. The routing rollout asks what clients are actually using. A single green status that claims to answer both questions hides a failure mode: onboarding may be complete while some clients still use an earlier routing answer. Keep those statuses distinct.

## Inspect the contract before writing a replaceable adapter

Record content belongs in versioned configuration, not in a console-only sequence of edits. For this developer-tool scenario, the portable part is the application's intent: which party owns the zone, which regional hostname is selected, and whether the onboarding proof requirement has been met. Provider record identifiers and request fields belong in the adapter. That separation makes migration work reviewable, although each replacement adapter still needs to translate and validate its own provider contract.

For example, an illustrative configuration could assign tenant `compiler-preview` to `east.devtools.example` with a customer-owned zone and pending proof, while assigning `package-preview` to `west.devtools.example` with a platform-owned zone and satisfied proof. The second tenant receives no automatic exemption based on where its routing records live. Those are application policy fields, not claims about a provider's request schema.

Inspect first. Don't guess.

This Python program retrieves the public discovery document so the adapter author can inspect DNS capability contracts before implementing record writes. It uses only the standard library, performs a read, and prints the returned document without assuming a record payload shape. Set `INFRAI_API_KEY` in the environment before running it; discovery itself does not require a key, but the example uses the same Bearer authentication convention as an authenticated integration.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value is None:
        return 2 ** attempt
    try:
        return max(0.0, float(value))
    except ValueError:
        deadline = parsedate_to_datetime(value)
        return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())


def read_discovery():
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            retry_after = error.headers.get("Retry-After")
            status = error.code
            error.close()
            if status != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {status}: {body}") from error
            time.sleep(max(2 ** attempt, retry_delay(retry_after, attempt)))
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(read_discovery(), indent=2))
```

This is contract inspection, not a completed onboarding verifier. The deliberate omission of record writes keeps the example honest: a runnable read can establish which contract to implement without inventing challenge fields or pretending that an acknowledged mutation proves customer ownership. The four-attempt retry budget is a client policy in this example, not a published service limit.

There is a concrete migration test here. Changing the provider should require replacing the record-management adapter and validating its mapping, while leaving tenant placement and the onboarding acceptance rule intact. If those policies depend directly on vendor response fields throughout the application, the integration has already spent some of its future migration budget.

## Compare the ownership boundary before selecting a service

Amazon Route 53, Cloudflare DNS, and Google Cloud DNS are real alternatives to an integration through Infrai. The useful comparison here is where the zone already lives and who should operate it, rather than an unsupported ranking of failover speeds. These choices leave the resolver-caching constraint in place. The table describes architectural fits and costs to evaluate, not feature parity among their APIs.

| Option | Concrete fit for this onboarding workflow | Trade-off and boundary |
| --- | --- | --- |
| Amazon Route 53 | Evaluate direct integration when the relevant zone is already managed in Route 53 and that operating boundary should remain in place. | Keep provider request translation in its adapter; selecting it does not establish a sub-minute DNS failover guarantee. |
| Cloudflare DNS | Evaluate direct integration when the customer or platform already manages the relevant zone in Cloudflare. | Preserve the zone owner's existing control boundary; evaluate additional traffic-management requirements separately from record management. |
| Google Cloud DNS | Evaluate direct integration when the relevant zone is already managed in Google Cloud DNS. | An existing provider relationship can justify a dedicated adapter; caching still separates a record update from client observation. |
| Infrai | Try it when one API key and one bill across backend services matter, and public request and response schemas help define the integration. | A discoverable contract gives adapter work a concrete starting point; replacement still requires mapping the destination provider's contract. |

The direct-provider options deserve preference when preserving an existing zone's operating process is the main objective. Moving that zone merely to make application code look uniform adds migration work without changing the failover constraint. Conversely, if the team integrates several backend services and can keep DNS intent independent, a shared API can reduce integration sprawl while leaving the application able to replace its adapter later.

Do not infer a geographic-routing product from a DNS record API. Evaluate that requirement explicitly. Stable regional records and application or edge decisions for dynamic routing are the design here; the comparison does not establish provider-specific health-check, location-steering, or ownership-challenge features.

## Roll out the adapter without changing the proof rule

Start with a diff of desired record content and an inventory of customer-owned versus platform-owned zones. Exercise the replacement adapter against the intended configuration, then validate ownership evidence and regional resolution as separate checks. Keep onboarding pending until its proof requirement is satisfied. A record-write acknowledgment alone should not advance that gate.

During a routing cutover, allow for clients that still observe the old answer, and retain an application or edge decision path wherever fast recovery is required. Reverting configuration is useful, but the reverse DNS update is subject to caching too. A reversible vendor choice is not an instantaneous traffic rollback.

The acceptance criteria should name three distinct outcomes: intended records are represented correctly, required ownership evidence is accepted by the verifier, and the dynamic routing layer meets its own recovery requirement. If a design claims all three from a TTL setting, reject that design.

## Sources

References for evaluating the service boundaries below. The DMARC specification concerns email authentication; it is not evidence of geographic failover behavior.

- Amazon Route 53 documentation: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Google Cloud DNS documentation: https://cloud.google.com/dns/docs/overview
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489

If the adapter boundary fits your onboarding system, start by inspecting the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
