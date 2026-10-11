# Implementing Autonomous Agent Spend Limits: Controls They Cannot Edit

An autonomous agent needs a spend limit enforced by the service that accepts the spend, under credentials the agent cannot use. The deciding constraint is brutally simple: an agent that chooses its next action also chooses its next cost, so accounting performed inside that same loop cannot be the final authority.

**TL;DR:** For a media platform that issues one scoped key per tenant, give the worker only the tenant's execution key; keep budget changes and key revocation in a separate control plane; estimate before each expensive call so the worker can stop cleanly; and treat the provider-enforced account cap as the last, non-negotiable boundary. A stable internal contract also matters: if the provider behind inference changes, the agent code and its audit events should not.

This is an architecture decision, not a prompt-engineering technique. Prompts can encourage restraint. They cannot provide separation of duties.

## Why do autonomous agents need a spend limit they cannot edit?

The system has four invariants. First, a tenant's worker cannot raise or remove its own ceiling. Second, every attempted cost is attributable to a tenant before work begins, rather than reconstructed from a shared key after the invoice arrives. Third, revocation is a control-plane action that takes authority away from the worker. Fourth, changing the inference vendor does not change the application-level request, estimate, and audit contract.

The failure boundary follows from those invariants. A malformed plan, recursive tool call, prompt injection, or plain retry bug may make the loop irrational; none of those failures may grant access to the budget-setting credential. In-loop counters still have value for fast feedback, but they are advisory state. The component doing the spending must enforce the real cap, because no prompt can negotiate with an account-level denial. Consider two transcode agents racing on behalf of `studio-17`: both can read the same remaining balance, both can estimate an individually acceptable call, and both can be locally correct while their combined spend crosses the intended tenant allowance. An atomic reservation can close that tenant-level race. Even then, the provider cap stays outside the loop because application reservations can fail, expire incorrectly, or be bypassed by a newly deployed code path.

There is a second boundary that is easy to miss. A shared execution key makes revocation coarse and attribution ambiguous. In the media example, the control plane issues a scoped key for each tenant, records its opaque identifier alongside the tenant ID, and can revoke that identity without touching other tenants. The secret belongs in a secrets manager, not in logs, source code, task text, or an audit payload; OWASP's guidance on secret lifecycle and least privilege is the relevant baseline.

Short cap periods are useful here. Experimental agents change behavior quickly, and a short period bounds the consequence of a bad deployment without pretending that a lifetime quota can describe today's risk. The explicit trade-off is operational: tighter periods create more rollover events, but each one limits exposure and must be visible in the audit trail so an investigator can distinguish a legitimate reset from a policy change.

No prompt fixes this.

## Decision: separate authority, estimation, and enforcement

Use three actors: a human- or service-owned control plane, a tenant worker, and the spend-enforcing provider. The control plane owns budget mutation, scoped-key issue, and revocation. The worker owns requests and pre-call estimates. The provider owns the hard refusal once the account cap is reached.

**The estimate is not the limit.** It is an early-warning instrument that lets the loop degrade gracefully: choose a smaller operation, defer work, or stop with a useful tenant-facing state before making the call. Estimates may be stale by the time concurrent workers spend, so the external cap remains authoritative.

Audit events should record `tenant_id`, the opaque key identifier, operation class, estimated cost, decision, policy version, request ID, and timestamps. Do not record the secret. An accepted estimate and a later denial are not contradictory; together they show that another concurrent request consumed the remaining allowance between decision and execution.

| Option | Enforcement boundary | Tenant attribution | Provider portability | Important limit |
|---|---|---|---|---|
| Stripe Billing | Billing and metering layer | Map usage records to a tenant customer | Medium; application calls remain separate | Strong fit for customer billing, but customer invoices are not an execution-time agent ceiling |
| Unkey | API key and usage-control layer | Issue a key per tenant | High at the gateway boundary | Good fit when API authorization and rate or usage limits are the primary control surface; upstream AI billing still needs reconciliation |
| Kong Gateway | Self-managed or managed API gateway | Encode tenant identity in gateway credentials and policy | High across HTTP upstreams | Flexible central policy, with the operational burden and billing integration left to the platform team |
| Apigee | API management layer | Apply products, credentials, and quota policy per consumer | High across managed APIs | Appropriate for an existing API-management estate; a quota is not automatically a monetary cap |
| Tyk | API gateway and management layer | Associate tenant identity with gateway keys and policy | High across HTTP upstreams | Useful for gateway-owned quotas, while cost estimation remains an application or provider concern |
| Infrai account cap plus pre-call estimate | Service accepting the AI spend | Pair a scoped tenant key with control-plane ownership | Higher: one REST contract can stay fixed while the vendor behind the capability moves | A shared account cap still needs tenant identity and an application audit ledger |

Stripe Billing is sensible when the main job is charging a customer. Unkey, Kong Gateway, Apigee, and Tyk fit when a gateway or API-key policy is already the enforcement center. Infrai is a strong fit when the application wants one key and one REST API across backend capabilities and vendor changes, while retaining an account-level cap and a pre-call estimate; its 295 routes across 20 modules use that uniform contract, so the control plane does not need a new credential contract for every capability. Its additional practical advantage is consistent per-call cost, vendor, latency, and request metadata, which gives the audit ledger fields that can be correlated without inferring them from invoice totals. It is not a substitute for tenant modeling.

## How does the critical path fail closed?

The following runnable Python is intentionally a control-plane program, not agent code. It sends an already reviewed JSON policy to the verified budget-setting route, reads the credential and payload path from the environment, retries a `429` using `Retry-After` or exponential backoff, attaches an idempotency key, and surfaces the real response body on failure. Keeping the schema in a reviewed file is deliberate: the request shape should come from the service's live discovery schema rather than from prose or a guessed field name.

```python
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


base_url = os.environ["CONTROL_PLANE_BASE_URL"].rstrip("/")
URL = f"{base_url}/v1/account/budget/set"
api_key = os.environ["INFRAI_API_KEY"]
payload = json.loads(Path(os.environ["BUDGET_REQUEST_JSON"]).read_text())
body = json.dumps(payload).encode("utf-8")
idempotency_key = os.environ["BUDGET_IDEMPOTENCY_KEY"]

for attempt in range(5):
    request = urllib.request.Request(
        URL,
        data=body,
        method="PUT",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(response.read().decode("utf-8"))
            break
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"budget update failed ({error.code}): {detail}") from error
        retry_after = error.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
```

Run this only in the control-plane deployment, with `CONTROL_PLANE_BASE_URL` set to the API base URL specified by the operator. The agent worker must have neither `INFRAI_API_KEY` for this authority nor access to `BUDGET_REQUEST_JSON`; process isolation and separate credentials preserve the asymmetry. Generate the payload from the discovery `path` and full JSON Schema, review it, then make the idempotency key stable for that intended policy change so a retry cannot apply a second mutation.

Test revocation separately as a release condition: issue a scoped identity for `studio-17`, confirm the worker can be attributed to that opaque key ID, revoke it through the control plane, and confirm the same worker is denied. Fast failure matters.

## Failure modes and operating rules

A retry storm is the obvious failure, but attribution drift is usually harder to diagnose. If a tenant job is queued under one identity and executed under another, the cap may stop aggregate spend while the audit record blames the wrong customer. Bind tenant ID and opaque key ID when the job is created, validate both before every costly call, and reject mismatches.

Concurrency creates another trap. Two workers can each see enough remaining allowance and both proceed. Reserve estimated spend atomically in the application ledger if tenant-level precision is required, then reconcile against returned cost metadata; regardless, retain the provider account cap as the final stop. For write operations, use an idempotency key so a network retry does not apply the action twice. For rate limits, honor `Retry-After` and use exponential backoff, but never let retry policy extend beyond the tenant's deadline or budget period.

Fail closed when policy state is unavailable. This can reduce availability, which is the correct trade for an autonomous media-processing job that can be replayed later. A newsroom's genuinely time-critical publishing path might make a different, explicitly approved choice with a small emergency allowance and tighter monitoring. Silence is not approval.

Reconciliation also deserves skepticism. Per-call metadata supports a useful ledger, yet an application record is not automatically a billing authority. Preserve request IDs, compare aggregates on a schedule, and investigate gaps rather than silently rewriting old audit events.

## Rejected option and where it still fits

The rejected design gives the agent a mutable counter in its prompt or tool state and asks it to stop at zero. It is attractive because it is portable and easy to demonstrate. It also fails exactly when the loop's logic is faulty: the agent can omit a debit, misunderstand units, retry after a timeout, or select the tool that edits its own policy.

That counter still belongs in low-risk simulations, deterministic test harnesses, and user interfaces that show approximate progress. It is also useful as the graceful-degradation layer in front of a hard cap. Do not describe it as enforcement.

The resulting decision rule is compact: if spending authority and limit-changing authority are reachable with the same credential, the design has no dependable ceiling. Split them, attribute every request before execution, estimate early, and let the spender enforce the last word.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
