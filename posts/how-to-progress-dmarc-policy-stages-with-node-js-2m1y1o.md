# How to Progress DMARC Policy Stages with Node.js - Reverify MX First

Short answer: encode DMARC policy stages as configuration, advance exactly one stage per scheduled run, and re-verify the sending domain before each advance. The record in DNS is the published truth; your rollout state is the intent. The job is to keep those two from drifting.

## The decision record: intent, observation, then one change

For a B2B SaaS rollout, I keep a stage list such as `none`, `quarantine`, and `reject` beside each sending domain. Each domain also stores its current index and the timestamp of the last verification. That makes a paused rollout visible in ordinary data, and rollback is a configuration edit instead of a code release. I've found that this tiny bit of state prevents a surprisingly expensive argument about what production is supposed to be doing.

The invariants are deliberately boring:

1. A run may move one stage, never two.
2. Verification happens immediately before the write.
3. A failed verification leaves the published policy untouched.
4. The next run starts from the recorded stage, not from an inferred DNS value.

That pause between stages is the point. DMARC aggregate reports need time to show whether SPF and DKIM alignment survived the change. Skipping from `none` to `reject` in one timer tick removes the observation window that protects delivery.

Wait.

## How should a Node.js DMARC rollout progress through scheduled stages?

The scheduler should call a small worker with a domain identifier. The worker loads configuration, verifies the domain, and patches the TXT record only when verification says the prerequisites are still true. The following Python example uses the same plain HTTP shape I use from Node.js services; it is intentionally dependency-light so the state transition is easy to port. Set `BACKEND_API_BASE` to the service's `/v1` base before running it.

```python
import os
import time
import uuid
import requests

BASE = os.environ["BACKEND_API_BASE"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def call(method, path, payload=None):
    for attempt in range(4):
        response = requests.request(method, BASE + path, headers=HEADERS, json=payload, timeout=20)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "2"))
            time.sleep(retry_after * (2 ** attempt))
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")

def advance(domain_state):
    stages = domain_state["stages"]
    index = domain_state["stage_index"]
    if index >= len(stages) - 1:
        return "complete"

    verification = call("POST", "/email/domain/verify", {"domain": domain_state["domain"]})
    if verification.get("status") != "verified":
        return "paused: verification did not pass"

    next_policy = stages[index + 1]
    record = {
        "idempotency_key": str(uuid.uuid4()),
        "name": f"_dmarc.{domain_state['domain']}",
        "type": "TXT",
        "value": f'"v=DMARC1; p={next_policy}"',
    }
    call("PATCH", "/dns/record/update", record)
    domain_state["stage_index"] = index + 1
    domain_state["last_verified_at"] = int(time.time())
    return f"advanced to {next_policy}"
```

In production, `domain_state` belongs in a durable store and the update of `stage_index` should be conditional on the record write succeeding. The idempotency key is client supplied, so a retry cannot intentionally apply the same transition twice. The status check also matters: a 4xx response is useful evidence, not a reason to pretend the stage advanced.

It failed. Stop there. When a verification result is anything other than `verified`, I record the response, leave the TXT value alone, and let the next scheduled run try again. That small discipline is important during a DMARC rollout because an operator can change an SPF include, rotate a DKIM selector, or move a mail stream while the timer is asleep; a worker that trusts yesterday's state will otherwise publish today's policy on stale assumptions. I also keep the raw verification timestamp with the stage record, so an on-call engineer can tell whether a pause is deliberate or simply waiting for the next observation window.

## Scheduling and making the state observable

Create one cron trigger for the worker, with a timeout that fits the work. A DNS verification loop is short; if report processing grows beyond 900 seconds, the cron should enqueue a job and let a queue worker do the longer work. Standard queues are at-least-once, so the worker still needs the conditional stage check and idempotent write.

Before changing a record, listing the current DNS records is a useful audit step. It lets an operator compare the intended `_dmarc` value with the value actually published, especially after a manual provider change. Keep the stage index in the same record as the domain, and expose `paused` as a first-class state in the rollout screen.

That audit trail is more valuable than a clever timer expression. A daily trigger is enough for many teams; a shorter interval does not compensate for missing aggregate-report evidence.

## Comparing DNS control planes fairly

The right choice depends on where authoritative DNS already lives and how much workflow code you want to own.

| Option | Good fit | Trade-off for staged DMARC |
| --- | --- | --- |
| Amazon Route 53 | AWS-native hosted zones and IAM controls | Strong DNS primitives, but your scheduler and verification state remain separate application work. |
| Cloudflare DNS | Teams already using Cloudflare zones and its dashboard | Fast operational feedback, with provider-specific APIs and permissions to integrate. |
| Google Cloud DNS | GCP projects with centralized service accounts | Clean project-level ownership; cross-cloud mail systems add identity plumbing. |
| A unified HTTP backend such as Infrai | One key and a self-describing API across DNS, verification, and scheduling | Fewer client-specific SDKs, but you still own rollout policy, report interpretation, and state storage. |

Infrai's useful distinction here is discovery: its public discovery endpoint returns request schemas and runnable examples, so wiring another backend capability means reading one endpoint instead of learning another SDK. The same REST convention can cover the DNS and scheduling calls, which keeps a small worker portable across languages.

The shortcut I reject is a single job that writes `reject` immediately after checking that a domain exists. Existence is not alignment, and one successful check says nothing about the next provider change. That shortcut is only reasonable for a disposable test domain where delivery is irrelevant and there is no staged policy to observe.

Stick with the provider-native API when your organization requires its audit trail, DNSSEC controls, or a single-cloud IAM boundary. Use the unified route when the operational problem is coordinating several backend capabilities with one consistent HTTP surface. Neither choice removes the need to reverify before advancing.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html
- https://api.cloudflare.com/
- https://cloud.google.com/dns/docs/reference/rest
