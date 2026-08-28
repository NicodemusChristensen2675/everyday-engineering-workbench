# Polling APIs for Custom-Domain Bounce, Complaint, and Suppression Control

Short answer: choose an email delivery service only after its polling API proves that your SaaS can reconstruct bounce and complaint outcomes, update one suppression authority, and stop a custom-domain ramp without relying on a dashboard.

Warmup is downstream of that decision. A gradual traffic schedule cannot compensate for an unauthorized sender, a stale suppression list, or an event cursor that advances before data is committed. The simplest service is the one whose failure boundaries your team can test and operate.

## Decision record: protect the recipient before the transport

This decision is about ownership, not a feature score. The application owns recipient eligibility and message intent. The delivery service accepts a message and reports later outcomes. A reconciliation worker turns those outcomes into durable policy. Those responsibilities should remain distinct even when one managed service supplies the transport, dashboard, and event API.

Three invariants set the boundary:

1. Every producer checks the same authoritative suppression decision before enqueueing a message.
2. An accepted send request records submission, not delivery.
3. A polling cursor advances in the same transaction that records the fetched events.

The first invariant closes an easy-to-miss gap in a small SaaS. Product mail, billing notices, invitations, and authentication messages often begin in separate workers. If each worker keeps its own suppression cache, a permanent outcome observed by one path may be invisible to another. The authority therefore needs the normalized recipient, the reason, the observation time, and enough provenance to audit the decision. Cache it for speed if necessary, but don't let the cache become policy.

Identity is a separate gate. RFC 7208 defines SPF as a way for a domain to authorize hosts to use its identity during mail transfer. It does not establish recipient consent, guarantee inbox placement, or turn an application response into a delivery receipt. I treat custom-domain ownership verification, DNS policy publication, event reconciliation, and production enablement as separate release conditions — all must pass before a tenant can ramp traffic.

Stop there for a moment.

This separation also clarifies what “warmup” means operationally. It is a controlled release of eligible traffic from an authenticated identity, with a pause control and observable outcomes. It isn't list cleaning, permission, or a retry policy. A candidate service that presents warmup as a self-contained deliverability solution is leaving the most important decisions outside the frame.

## How should a small SaaS polling API handle bounce and complaint tracking?

Treat polling as an at-least-once ingestion path. Fetch a stable page, write every previously unseen event, apply any resulting suppression transition, and persist the next cursor atomically. A process restart may replay work; it must never create a second policy transition or skip an uncommitted page. That is why the remote event identifier and your own immutable message identifier both belong in the ledger.

Recovery comes first.

The selection demo should exercise recovery, not just the happy path. Ask the candidate to show pagination while new events arrive, an empty page, a duplicate page, and a restart before commit. Establish how far back events can be queried and whether the cursor has a documented lifetime. I'm not sure a polling-only design is adequate until those retention terms are compared with the team's worst credible interruption; a live test and the service contract resolve that uncertainty.

Event vocabulary matters too. Your local model needs to preserve the difference between a temporary outcome, a permanent outcome, a recipient complaint, and a confirmed delivery when the service exposes them. Don't flatten everything unsuccessful into `failed`. The suppression policy may respond differently, operators need the original evidence, and later adapters should not have to infer meaning from a lossy status.

Polling delay becomes policy delay. For that reason, the worker's schedule cannot be chosen only from infrastructure convenience. The team should decide how long it is willing to keep sending after a suppressing event has occurred but before it has been observed, then set the interval and alert threshold accordingly. Your mileage may vary: an authentication stream and a periodic account digest have different urgency, even though both must consult the same recipient authority.

There is a catch. Polling is not suitable when the required reaction time is shorter than a defensible poll-and-process cycle. In that case, use authenticated event push for the fast path and retain polling or export-based reconciliation for gap detection. Conversely, stick with polling when the team cannot safely expose and operate a callback receiver, provided the remote retention window and the agreed suppression delay leave clear recovery margin.

## Compare control planes, not dashboard polish

The options differ mainly in where recovery complexity lives. This table is the decision record; it isn't a ranking.

| Control plane | Valid use case | Failure boundary the team owns | Reason to reject it |
| --- | --- | --- | --- |
| Polling events | A small team can operate one scheduled worker and tolerate explicit detection delay | Cursor durability, pagination, deduplication, and lag alerts | Remote retention or cursor semantics cannot cover the recovery window |
| Pushed events | Fast policy updates justify an authenticated public receiver | Authentication, retry handling, deduplication, ordering, and dead letters | There is no replay or reconciliation path after a missed callback |
| Push plus polling | Fast reaction and independent completeness checks are both required | Two ingestion paths must share one idempotent ledger | The team cannot test the combined state transitions |
| Self-operated transfer | The organization already owns mail queues, abuse handling, and sender operations | Delivery infrastructure and feedback processing remain on call | The staffing burden conflicts with the goal of a simple SaaS stack |

Cost belongs in the review, but per-message price is a weak proxy for simplicity. Include event retention, log export, identity isolation, callback infrastructure, reconciliation work, and the operational load of tenant offboarding. A service with an attractive send rate can still be the expensive choice if it makes policy evidence hard to retrieve. I would rather see a plain, testable cursor contract than a sophisticated chart that cannot be reconciled with application intent.

Charts don't reconcile state.

Custom-domain isolation needs an equally concrete question: what exactly can be paused? A tenant, a domain, a message class, or only the whole account? The right boundary depends on the product, but the control must align with the unit whose identity and traffic are being introduced. Shared emergency controls are useful; they don't replace tenant-scoped eligibility.

## Put the critical path in one Python transaction

The following code is deliberately transport-neutral. It shows the contract I want to test: suppression is checked before submission, events are idempotent, and the cursor moves only inside the ledger transaction. Network authentication, request shapes, and pagination fields belong in a service-specific adapter whose behavior is covered by contract tests.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Protocol


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    recipient: str
    outcome: str
    observed_at: datetime


class EventSource(Protocol):
    def fetch_after(
        self, cursor: str | None
    ) -> tuple[list[DeliveryEvent], str | None]: ...


SUPPRESSING_OUTCOMES = {"permanent_bounce", "complaint"}


def submit(transport, ledger, suppressions, message):
    if suppressions.contains(message.recipient):
        ledger.record_blocked(message.id, reason="suppressed")
        return

    ledger.record_intent(message.id, message.recipient)
    transport.submit(message)
    ledger.record_accepted(message.id)


def reconcile(source, ledger, suppressions, cursor_store):
    cursor = cursor_store.read()
    events, next_cursor = source.fetch_after(cursor)

    with ledger.transaction():
        for event in events:
            if ledger.has_event(event.event_id):
                continue

            ledger.record_event(event)
            if event.outcome in SUPPRESSING_OUTCOMES:
                suppressions.upsert(
                    recipient=event.recipient,
                    reason=event.outcome,
                    observed_at=event.observed_at,
                    source_event_id=event.event_id,
                )

        cursor_store.write(next_cursor)
```

The most valuable tests interrupt this code at transaction boundaries. Replay the same page. Deliver pages in a different order if the API permits that. Return no events with a changed cursor. Raise a client-side timeout before a response is known, then verify that submission retry rules cannot create uncontrolled duplicate intent. For reconciliation, assert that a replay leaves one event record, one resulting suppression state, and a cursor that never points beyond committed evidence.

Observability should follow the same model. Alert on cursor age, oldest unresolved intent, age between acceptance and a terminal outcome, duplicate-event volume, and attempts blocked by suppression. Raw send-call success is useful for adapter health, but it is not the product outcome. This distinction is especially important for one-time codes: the application must own code expiration and must not treat late arrival as success. The WebOTP API can assist credential input on supported browsers; it does not replace server-side expiration, recipient eligibility, or delivery-state tracking.

Rollout should begin with reconciliation that records events without changing recipient eligibility. Compare the resulting ledger with the service's own event view, then enable policy updates for controlled traffic before expanding the tenant ramp. The exact cohort size is a product decision and should come from the team's risk tolerance and observed traffic, not a universal warmup recipe.

## Rejected option, and where it still fits

The rejected design is “accept the send response, inspect the dashboard, and warm the domain.” It fails the invariants because the application has no replayable account of later outcomes, suppression depends on manual attention, and a successful API call can be mistaken for delivery. It also makes tenant offboarding and compliance review harder: the team cannot answer why a recipient remained eligible from its own durable evidence.

That design has a narrow valid use case. An internal prototype that sends only to controlled team addresses, has no external users, and does not depend on email for authentication can use manual inspection while the product contract is unsettled. Keep it there until the message taxonomy and consent model are understood. Before external recipients or custom domains are enabled, replace it with the ledger, suppression authority, and tested event path.

Managed warmup is also a poor primary selection criterion. It may be a useful traffic-control feature when its cohorts and pause behavior match the application's policy, but it cannot authorize a sending host, grant consent, or repair missing outcome evidence. Reject it when the ramp is opaque or cannot be stopped at the identity boundary you operate. Prefer a plain scheduler over an automation you cannot audit.

The final decision is intentionally unglamorous: verify sender identity, centralize suppression, preserve message intent, ingest outcomes idempotently, rehearse cursor recovery, and make the ramp stoppable. A service that demonstrates those behaviors is simple in the way that matters — its failures remain visible and bounded.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
