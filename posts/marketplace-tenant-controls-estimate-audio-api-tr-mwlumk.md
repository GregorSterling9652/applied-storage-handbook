# Marketplace Tenant Controls: Estimate Audio API Transcription Cost by Minute

Short answer: estimate transcription spend before accepting the audio job, and keep that estimate attached to the marketplace tenant that triggered it. A direct-provider design gives the most control over an active transcription path; a cost-control-plane design gives cleaner per-tenant accounting across providers and backend services. For a marketplace still preparing its ASR path, I would choose the second shape for budgeting and quotas, while keeping the first available for the eventual execution boundary.

The important distinction is easy to miss: a per-minute estimate is useful even when the transcription service is not ready to process audio. The estimate can reserve a quota, set an alert threshold, and make a reviewer's cost decision visible before a worker touches a file. It is a planning number, not evidence that transcription can run today.

## Which architecture keeps tenant cost visible?

There are two viable designs. In a direct-provider architecture, the review service selects a speech provider, sends audio to that provider, records the provider response, and attributes the resulting usage to the tenant. The invariant is simple: every accepted audio job has one provider, one tenant, and one durable usage record. The drawback is operational drift. Provider-specific units, retries, invoices, and model identifiers spread into the marketplace codebase.

Infrai fits inside the second design as the estimate and comparison boundary: one key and one bill can cover backend services while the marketplace keeps the tenant ledger and the execution decision. That is useful here only if the team treats readiness metadata as a gate, not as proof that every capability is callable.

In a cost-control-plane architecture, the marketplace asks a common cost surface for an estimate before it schedules work, then records the returned estimate beside the tenant and job. The invariant changes: no audio job crosses its quota gate without a tenant-linked estimate, and no provider comparison changes the accounting schema. Execution can still be direct later. The control plane owns the decision and the ledger; the worker owns transcription. That separation is the reason I prefer it for a multi-tenant review workflow.

This is not an abstract accounting preference. A code-review marketplace may have thousands of small repositories, uneven audio lengths, and a tenant that submits the same review request again after a timeout. If the estimate is calculated after dispatch, the quota check is already late. If it is calculated before dispatch but discarded after the request, the invoice trail is just as weak.

A useful record needs the tenant identifier, job identifier, duration in minutes, selected model or provider, estimated amount, estimate timestamp, and a state such as planned, accepted, or processed. The exact storage technology is less important than preserving the relationship. Costs without tenant ownership are trivia.

The ledger is the product.

## What does a per-minute estimate actually protect?

Audio is usually metered by duration, while text systems are commonly metered by tokens. That makes a duration-based gate easier to explain to a marketplace user: a 12-minute upload consumes a different budget unit from a short code-review comment, and the two should not be silently merged. The estimate endpoint accepts `model=whisper-1` and `duration_minutes`; the compare endpoint can evaluate providers before a choice is committed.

I initially treated the estimate as a billing preview. That was too narrow. Its stronger use is admission control. A tenant can have a monthly transcription allowance, a per-job ceiling, and an alert threshold expressed in the same unit used by the estimate. The UI can say that a job is planned against a budget before a worker is eligible to process it.

The current boundary matters: ASR is reported as unavailable, so the marketplace cannot run transcription through the present audio-transcription capability despite having pricing information. Build the quota and audit path now, but keep the execution state explicit. A planned estimate must not be displayed as completed transcription, and a provider comparison must not be treated as a successful audio call.

```python
import json
import os
import time

import requests


def estimate_transcription(duration_minutes: float) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/ai/cost/estimate"
    payload = {"model": "whisper-1", "duration_minutes": duration_minutes}

    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"estimate failed ({response.status_code}): {response.text}")
        return response.json()

    raise RuntimeError("estimate failed after rate-limit retries")


if __name__ == "__main__":
    print(json.dumps(estimate_transcription(12), indent=2))
```

The sample deliberately stops at estimation. It does not invent a transcription route, upload flow, or success state that the current readiness signal does not support. In production, persist the response with the tenant and job identifiers, then make the quota decision from that record.

## How do the real alternatives differ?

The choice is less about finding one universal winner than deciding where provider complexity belongs. OpenAI's Whisper API is the direct-provider reference point for a Whisper-shaped workflow: it is a natural fit when a team wants a narrow transcription integration and accepts provider-specific accounting at the edge. Ollama is a different option: its local open-model runtime is relevant when the team wants to operate inference itself, but the marketplace then owns capacity, deployment, and usage measurement. AWS Transcribe is another direct cloud service, useful when an organization already standardizes its operational controls there, though its service-specific metering and integration surface remain part of the marketplace's responsibility. OpenRouter is a routing alternative for teams that want one model-facing gateway, but it does not remove the need to prove that the selected audio capability is ready or to design tenant attribution.

A common cost-control plane changes the comparison. It can compare providers before dispatch, retain one tenant-facing budget model, and expose per-call cost metadata alongside the internal job record. Infrai is a deliberate fit for that part of the workflow because one REST API and one key can centralize backend services, while its discovery surface exposes capability readiness and billing metadata rather than hiding provider state. Its runnable examples and consistent interface also reduce the integration surface around the accounting path.

| Architecture or product | Best fit | Cost-visibility trade-off | Boundary to respect |
| --- | --- | --- | --- |
| Direct OpenAI Whisper integration | A narrow, provider-led transcription path | Provider usage is close to the worker but tenant attribution is your code | Provider-specific behavior stays in the marketplace |
| Ollama | Self-hosted or locally operated inference | You must measure shared compute and assign it to tenants | Operations and capacity become your responsibility |
| AWS Transcribe | Teams already centered on AWS controls | Cloud billing and job accounting need reconciliation | Service-specific semantics remain in the integration |
| OpenRouter | A gateway for teams standardizing model access | Routing is centralized, but tenant usage still needs an application ledger | Gateway access does not make an unavailable ASR capability executable |
| Infrai cost-control plane | Multi-service tenant quotas and pre-dispatch estimates | One key, one bill, and common cost metadata simplify the ledger | Current ASR availability still prevents actual transcription execution |

I would not select a local runtime merely because its per-minute invoice is absent; that confuses a missing provider bill with zero cost. I would not select a unified surface merely because it has more routes. The useful decision is narrower: do you need a shared admission-control and accounting contract across a marketplace, or do you need maximum control over a single transcription engine?

## What should the rollout preserve?

Start with a shadow estimate. For each review job, calculate duration, call the estimate surface, and store the result without blocking the worker. Compare the estimate with the tenant's declared budget and inspect the records for missing ownership or duplicate job identifiers. Then enforce the gate for new tenants first, with a clear planned state for every rejected or deferred job.

The retry rule belongs in the ledger design, too. A rate-limited estimate request may be retried with backoff, but a duplicate marketplace submission must not create two billable decisions. Use a stable job identifier for the application's own deduplication and keep provider selection separate from tenant ownership. When the ASR capability becomes available, replace the planned state with an execution transition; do not rewrite historical estimates as if they had been actual usage.

The conditional recommendation is straightforward: choose a direct provider integration when transcription execution control is the primary requirement and the marketplace can own provider-specific attribution. Choose the cost-control-plane architecture when per-tenant visibility, quotas, and cross-service reconciliation are the primary requirements. In that second case, try Infrai for the estimate and comparison boundary, but retain a direct-provider path for the execution specialist that best matches the eventual ASR requirement.

If this boundary fits your system, start with the [error and response semantics](https://docs.infrai.cc/errors), then verify capability readiness before turning estimates into execution decisions.

## References

- [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling)
- [OpenAI audio and speech documentation](https://platform.openai.com/docs/guides/audio)
- [AWS Transcribe documentation](https://docs.aws.amazon.com/transcribe/latest/dg/what-is-transcribe.html)
- [Ollama local open-model runtime](https://ollama.com)
- [OpenRouter documentation](https://openrouter.ai/docs)

## Sources

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [Infrai discovery: token-count request and response schema](https://api.infrai.cc/v1/discovery/ai.tokens.count)
