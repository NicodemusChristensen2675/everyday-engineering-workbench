# Node.js Feature Flags API for Percentage Rollout Audits (Under Retention Limits)

The bill for a percentage rollout is mostly an event-volume problem: decisions per day multiplied by bytes per decision, retention days, and stored copies. Start there. For a fintech experiment split across tenant cohorts, the least complex design that still supports incident reconstruction is a deterministic assignment function plus one compact, immutable decision event at the boundary where the flag affects behavior.

**TL;DR:** hash a stable tenant key with a flag-specific salt, map it into a fixed bucket range, and compare that bucket with the rollout threshold. Record the flag key, configuration version, bucket, result, tenant pseudonym, and trace identifier. Keep configuration changes longer than sampled request evidence, and never attach raw account, email, phone, or payment data to flag labels. This preserves the question that matters during an incident: "Which rule put this tenant on this path at that moment?"

That answer is deliberately narrower than "log everything." A flag evaluation can happen on every request, while a configuration mutation is rare. Treating those two streams as if they deserve the same payload and retention creates cost without guaranteeing a replayable decision.

## What is the observability bill actually made of?

Model the dominant term before choosing a collector, database, or dashboard. If `D` is evaluated decisions per day, `B` is average encoded bytes per event after batching, `R` is retention in days, and `C` is the effective number of stored copies, retained decision data is approximately `D × B × R × C`. Indexes, transport, query scans, and backups add more, but they do not rescue a design with careless cardinality.

Use a hypothetical workload to make the trade-off visible. Suppose an API makes 40 million relevant decisions per day. A 300-byte event retained for 30 days with two stored copies represents 720 GB before index and query overhead. This is arithmetic, not a benchmark or a price claim. Halving a field name is minor. Emitting one event per tenant, flag version, and observation window instead of one per request can change the dominant term, provided the aggregation still preserves the evidence required for an investigation.

The tempting fields are often the dangerous ones. A tenant identifier has high cardinality but is necessary for cohort reconstruction; an HMAC-derived pseudonym can retain joinability without placing the original identifier in telemetry. Request URLs, exception text, phone numbers, and free-form labels can carry regulated or identifying data and can also explode an index. Keep them outside the decision event. Link to separately governed request evidence with a trace identifier.

For this system, I would retain three different things:

| Evidence | Typical volume | Retention decision | Why it exists |
|---|---:|---|---|
| Configuration mutations | Very low | Longest | Reconstruct the rule and approval history |
| Aggregated cohort outcomes | Moderate | Long enough to compare experiment windows | Decide whether tenant cohorts diverged |
| Per-request decision events | High | Short, sampled, or incident-scoped | Connect a specific request to its chosen path |

The table avoids a fake universal number. Retention must follow the organization's investigation window, regulatory duties, deletion policy, and recovery objectives. Those inputs vary. The architecture should make each class independently configurable rather than hide one global retention knob.

## How should a Node.js feature flags API replay percentage rollout decisions?

Random choice at request time is the wrong primitive. A tenant can alternate between variants, retries can take another path, and an investigator cannot reproduce the original result from the configuration alone. Deterministic bucketing fixes those failures. Define the assignment input precisely: namespace, flag key, assignment version, and a stable tenant identifier encoded as UTF-8 with unambiguous separators. Apply a keyed hash, take an unsigned integer from a documented slice of the digest, and map it to a fixed range such as 0 through 9,999. A rollout of 12.50% enables buckets below 1,250. The comparison boundary, encoding, byte order, and version are part of the contract.

Do not use a language runtime's ordinary hash function. Its output may be process-specific or may differ after a runtime change. Also avoid changing the salt casually: that reshuffles the cohort, turning a rollout adjustment into a new experiment.

Replay must be boring.

The following Python function is useful as a cross-language test oracle for the Node.js service. It is not a second production evaluator. The Node.js implementation should produce the same bucket for the same fixture bytes before deployment.

```python
import hashlib
import hmac


BUCKETS = 10_000


def tenant_bucket(secret: bytes, flag_key: str, version: str, tenant_id: str) -> int:
    parts = ("fintech", flag_key, version, tenant_id)
    message = b"\x00".join(part.encode("utf-8") for part in parts)
    digest = hmac.new(secret, message, hashlib.sha256).digest()
    return int.from_bytes(digest[:8], byteorder="big", signed=False) % BUCKETS


def enabled(bucket: int, rollout_basis_points: int) -> bool:
    if not 0 <= rollout_basis_points <= BUCKETS:
        raise ValueError("rollout_basis_points must be between 0 and 10000")
    return bucket < rollout_basis_points
```

Keep fixed fixtures in both repositories: empty-looking edge cases, non-ASCII tenant keys, the boundary bucket, 0%, and 100%. The fixture secret must be synthetic. Production assignment secrets belong in the same controlled secret-management process as other application keys, while telemetry records a non-secret assignment version rather than the key itself.

Targeting rules need an explicit precedence order too. A practical order is deny override, allow override, eligible tenant cohort, then percentage bucket, with a default at the end. Record which rule matched. Without a rule identifier, `result=false` cannot distinguish an ineligible tenant from a tenant above the threshold, and those cases imply different rollback decisions.

## Record a decision, not a copy of the customer

In Express, evaluate once near the request boundary and attach the result to request-local context. Downstream handlers consume that frozen result; they do not evaluate the flag again. This matters when a long request overlaps a configuration change. One request should not debit an account under one variant and format its response under another.

The decision event can stay compact:

| Field | Example | Investigation use |
|---|---|---|
| `flag_key` | `risk_review_v2` | Names the controlled behavior |
| `config_version` | `cfg_01842` | Finds the immutable rule snapshot |
| `assignment_version` | `hmac-sha256-v1` | Replays the bucket contract |
| `tenant_ref` | `tnt_7e...` | Joins events without exposing the source ID |
| `bucket` | `0837` | Verifies the threshold comparison |
| `rule_id` | `eligible_percentage` | Explains precedence |
| `result` | `true` | States the selected path |
| `trace_id` | `4bf92f...` | Connects separately governed evidence |

`config_version` must resolve to an immutable snapshot containing the threshold and normalized rule set. A mutable row called "current" is operational state, not historical evidence. Store mutation time, actor or automation identity, review reference, and the previous version in an append-only audit stream. Clock timestamps help order events, but the version is the stronger join key because clock skew and ingestion delay are normal distributed-system conditions.

Keep experiment outcomes separate from assignment mechanics. For a fintech cohort comparison, useful outcome counters might include authorized requests, challenged requests, declines, timeouts, and latency distributions, split by low-cardinality dimensions such as flag version and variant. Do not put tenant IDs into metric labels. Use decision events for tenant-level reconstruction and metrics for population-level detection.

If the variant changes a user-facing flow, Core Web Vitals provide defined user-experience measures: LCP, INP, and CLS, assessed at the 75th percentile with separate thresholds documented by web.dev. They should be segmented by assigned variant before comparison. A global percentile can conceal a regression in a small rollout cohort. Backend authorization and fraud outcomes still need domain-specific measures; web performance is not a proxy for them.

## What must an incident timeline prove?

An investigator should be able to start with a trace, recover the tenant pseudonym and frozen decision, fetch the exact configuration snapshot, recompute the bucket, and confirm the matched rule. Then the investigator compares cohort outcomes before and after the mutation window. That sequence is the acceptance test for the telemetry model.

Consider an experiment where eligible tenants move from 5% to 20%. A spike in OTP challenges appears after the change. The team needs to separate at least four explanations: the newly exposed bucket range behaved differently; tenant eligibility changed; the decision service used inconsistent configuration versions; or the downstream OTP path changed independently. A chart labeled only `variant=on` cannot answer any of them. Versioned decisions plus deployment and downstream-service markers can.

This is where compliance discipline improves debugging instead of fighting it. The event says enough to replay the choice but does not carry an email address, phone number, card data, request body, or full rule input. Access to tenant-level events can be narrower than access to aggregated experiment metrics. Deletion and retention controls can operate on the evidence class that actually contains a joinable tenant reference.

Test reconstruction before the rollout. Generate a synthetic tenant set, evaluate it in the Node.js service, verify every result against the Python oracle, and archive the synthetic fixtures with the assignment version. During deployment, alert on unknown configuration versions, missing rule identifiers, evaluation errors, and disagreement between requested and active versions. A flag lookup failure should follow a declared fail-open or fail-closed policy based on the controlled risk; for a financial authorization path, that choice needs security and compliance review rather than a library default.

## Change the dominant term, then accept the blind spot

The material cost reduction comes from emitting fewer high-volume decision events and retaining them for less time, not from weakening the configuration audit trail. Aggregate repeated decisions within a bounded window when the same tenant, flag version, rule, and result recur. Sample routine successful traces if the incident model permits it. Preserve all configuration mutations and increase decision-event capture around rollout changes or declared incidents.

This creates a real trade-off. Once unsampled per-request events expire, the team may still prove how a tenant would have been assigned from the immutable configuration and deterministic algorithm, but it may no longer prove that a particular old request loaded that configuration, matched that rule, or reached a downstream dependency. Aggregates can show cohort divergence; they cannot reconstruct an individual timeline.

Stop keeping raw request payloads and indefinite full-fidelity evaluations by default. The cost is reduced certainty for late, request-specific investigations. Make that loss explicit in the retention review, align the window with reporting and incident-discovery obligations, and provide a controlled way to raise capture before risky changes. Cheap telemetry that cannot answer the incident question is waste. Unlimited telemetry is a liability.

## Further reading

- Martin Fowler, "Feature Toggles": https://martinfowler.com/articles/feature-toggles.html
- web.dev, "Web Vitals": https://web.dev/articles/vitals
