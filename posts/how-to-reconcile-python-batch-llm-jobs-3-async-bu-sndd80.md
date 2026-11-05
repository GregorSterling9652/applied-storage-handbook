# How to Reconcile Python Batch LLM Jobs — 3 Async Bulk Extraction States

Use a durable reconciliation ledger before moving sales-call summarization from realtime requests to asynchronous bulk LLM jobs. TL;DR: batch processing is a candidate for lower API spend when the CRM action can wait, but the decision should depend on accepted, timely actions and recoverable results; a cheaper submission that cannot be reconciled is unfinished work. No universal discount or completion window is established here.

For a developer-tools team turning sales calls into follow-up tasks, the useful unit is an approved CRM action tied to a specific transcript revision. Summarization, tagging, and extraction can share a submission envelope, but they need separate task identities when their outputs are reviewed or retried independently. Provider portability starts with retaining those identities outside the provider's job record.

This decision record takes a data-reconciliation angle: what evidence survives the gap between accepting a job and committing its business effect? The Python examples use local fixtures to make that boundary inspectable. They implement no provider API and make no throughput, price, or delivery guarantees.

## Decision and invariants

Adopt an application-owned manifest and reconcile every returned item against it before allowing a CRM write. In this design, the three local states are pending, ready, and committed. Pending means there is no validated candidate; ready means a candidate has passed the local structural checks; committed means the destination has acknowledged the business operation. These are application definitions, not claims about any service's status vocabulary.

Keep the transcript revision, task, schema revision, and stable action key together. A retry must retain its business identity, while an intentional change to the transcript or extraction contract must produce a new identity. Record submission attempts separately so that trying another provider does not silently create another CRM task. An adapter may translate transport details, but it must return the application's item key. If a transport cannot preserve or reliably map that key, it doesn't meet this design's portability requirement. Consider the synthetic fixture below: 1 transcript revision produces 1 action key, and 2 identical result envelopes still represent 1 candidate. Two disagreeing envelopes represent a conflict, not two tasks. Keeping both attempts attached to the same identity preserves the evidence needed to decide which action, if either, should reach the CRM; giving each retry a fresh business key would erase that distinction and let both candidates look independently valid.

Order proves nothing.

The failure boundaries are distinct: submission acknowledgement doesn't validate an extraction; a validated extraction doesn't prove a CRM write; a CRM timeout doesn't prove that the write failed. Persisting these distinctions is more useful than a single job-level success flag, especially when one call produces a summary but its action extraction needs review. The code below stops at ready because a local dictionary cannot establish durable destination acknowledgement.

## How should batch LLM jobs compare with realtime API extraction?

Compare paths against the same transcript revisions and acceptance rubric. A fair operational comparison includes rejected output, repeated attempts, operator review, storage retention, and missed deadlines; API charges alone cannot decide whether the workflow produced useful work. Treat these as accounting categories to measure, not as measured savings.

| Path | Decision criterion | Failure boundary to inspect | Trade-off |
| --- | --- | --- | --- |
| Realtime request | A user needs the action during the current interaction | Ambiguous response or destination acknowledgement | Earlier feedback, with less room to defer work |
| Asynchronous bulk submission | The action can wait until a declared deadline | Item omissions, duplicate results, and late completion | Delayed visibility in exchange for scheduling flexibility |
| Application queue feeding ordinary requests | The team needs controlled background dispatch | Worker restart between response and persistence | More dispatch ownership; queueing alone establishes no API discount |

Set a deadline before selecting a path. For example, a follow-up task intended for the next sales handoff can be eligible for background processing, whereas an action a representative must confirm before closing the current screen has a different acceptance condition. Neither example supplies a universal latency threshold. Get the actual provider's supported operations, limits, retention terms, and commercial terms before making a production comparison; the sources below do not establish those details for a batch API.

Use the same acceptance tests across providers.

Otherwise, a migration can appear successful because the new adapter returns parseable JSON while dropping the owner or inventing an unsupported next step. Parsing is a necessary boundary here. It's a weak semantic test.

## Step 1: Freeze the work identity

Save the following as manifest.py and run it with Python. It creates a synthetic sales-call fixture and a manifest file in the current directory. The fixture values are illustrative, including the transcript revision and schema version; none is a benchmark or a production limit.

```python
import hashlib
import json
from pathlib import Path

identity = {
    "call_id": "demo-call-17",
    "transcript_revision": "revision-2",
    "task": "extract_crm_action",
    "schema_revision": "1",
}
canonical = json.dumps(identity, sort_keys=True, separators=(",", ":"))
item_key = hashlib.sha256(canonical.encode("utf-8")).hexdigest()
manifest = {
    item_key: {
        "identity": identity,
        "state": "pending",
        "transcript": (
            "Developer tools evaluation: Morgan will send the "
            "integration checklist to the prospect."
        ),
    }
}
Path("manifest.json").write_text(json.dumps(manifest), encoding="utf-8")
print(item_key)
```

The acceptance criterion is reproducibility: the same identity fields produce the same key, and changing the transcript revision changes the encoded input. A revision label is a contract with the upstream transcript store; if content can change behind that label, this identity scheme is insufficient. Bind the manifest to an immutable content revision or add a content digest before production use. The Python hashing and JSON references document the primitives used here.

Do not confuse this fixture's file write with a durability protocol.

Production storage needs an explicit acknowledgement boundary and a tested recovery procedure. Retain the source revision and manifest long enough to explain any action that remains in the destination, subject to the team's retention policy, rather than assuming a remote job history is the permanent audit record.

## Step 2: Reconcile candidates before releasing actions

Save this as reconcile.py beside the manifest. Running it after the first example writes ready.json from a synthetic result envelope. A production adapter would supply envelopes in this application-owned shape; the example deliberately leaves submission and polling outside the critical reconciliation path.

```python
import json
from pathlib import Path


def reconcile(manifest, results):
    candidates = {}
    for result in results:
        key = result["item_key"]
        if key not in manifest:
            raise ValueError("Unknown item key")
        action = result["action"]
        if not isinstance(action, dict):
            raise ValueError("Action must be an object")
        if set(action) != {"owner", "next_step"}:
            raise ValueError("Unexpected action fields")
        if any(not isinstance(v, str) or not v.strip()
               for v in action.values()):
            raise ValueError("Action fields must be nonempty strings")
        if key in candidates and candidates[key] != action:
            raise ValueError("Conflicting results require review")
        candidates[key] = dict(action)
    return {
        key: {**record, "state": "ready", "action": candidates[key]}
        if key in candidates else dict(record)
        for key, record in manifest.items()
    }


if __name__ == "__main__":
    manifest = json.loads(Path("manifest.json").read_text(encoding="utf-8"))
    key = next(iter(manifest))
    results = [{
        "item_key": key,
        "action": {
            "owner": "Morgan",
            "next_step": "Send the integration checklist to the prospect",
        },
    }]
    ready = reconcile(manifest, results)
    Path("ready.json").write_text(json.dumps(ready), encoding="utf-8")
    print(ready[key]["state"])
```

This is a pending-manifest-to-ready transformation, not a general state machine for replaying committed records. The caller must supply pending work only. Missing results stay pending, identical duplicates collapse, unknown keys fail, and conflicting duplicates stop reconciliation for review. Those are deliberate choices: failing the local reconciliation unit postpones valid neighbors, but it prevents an ambiguous candidate from being released silently. Smaller reconciliation units can reduce that operational blast radius.

Structural validation cannot establish that Morgan actually owns the action.

Before release, apply the semantic review policy to the source transcript and confirm the destination account mapping. Then use the stable action key in whatever deduplication contract the CRM actually supports. If a write times out, look up the existing operation before retrying when the destination permits that lookup; without destination deduplication or a verifiable operation record, this design cannot promise exactly-once business effects. HTTP idempotency semantics, described in RFC 9110, do not by themselves supply that business contract.

## Step 3: Test the boundary, then reject blind bulk writes

Save this as check_reconciliation.py beside reconcile.py. Run it after the earlier examples. It checks two failure behaviors that matter during provider changes: missing output must remain visible, and disagreeing duplicates must not become an arbitrary winner.

```python
import json
from pathlib import Path
from reconcile import reconcile

manifest = json.loads(Path("manifest.json").read_text(encoding="utf-8"))
key = next(iter(manifest))
assert reconcile(manifest, [])[key]["state"] == "pending"
first = {
    "item_key": key,
    "action": {"owner": "Morgan", "next_step": "Send checklist"},
}
second = {
    "item_key": key,
    "action": {"owner": "Morgan", "next_step": "Schedule review"},
}
assert reconcile(manifest, [first, first])[key]["state"] == "ready"
try:
    reconcile(manifest, [first, second])
except ValueError as error:
    assert str(error) == "Conflicting results require review"
else:
    raise AssertionError("Conflict was accepted")
print("Reconciliation checks passed")
```

Before deployment, extend the acceptance exercise to interrupted persistence, changed transcript revisions, late output, and ambiguous CRM acknowledgement. Observe pending age, ready age, conflict count, and acknowledged actions per submission cohort. Keep raw call content out of routine metrics. These observations reveal whether a nominally finished bulk job has left business work stranded, while retained identities let the team inspect a failure without tying the investigation to one provider's console.

Reject the tempting option of uploading a file and writing each returned line straight into the CRM. Its valid use case is a disposable preview where no durable business action follows and a person can inspect the whole output; it is a poor fit for unattended task creation because the reader still needs to resolve missing items, repeated delivery, and uncertain writes.

The decision is conditional: use asynchronous processing when the action deadline permits it, the transport can preserve item identity, and measured accepted output justifies the delay and operational work. Keep realtime processing available for actions that require immediate confirmation. Portability is demonstrated by replaying the same reconciliation tests against another adapter and retaining the same business identities, not by declaring two request formats equivalent.

## References

- Python hashlib documentation, for the digest primitive used in work identity: https://docs.python.org/3/library/hashlib.html
- Python json documentation, for serialization and parsing behavior: https://docs.python.org/3/library/json.html
- RFC 9110, section 9.2.2, for HTTP idempotency semantics and the boundary around automatic retries: https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2
