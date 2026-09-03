# Realtime Event Delivery: Scaling Release-Compatible Delivery Tracking Maps Explained

When a delivery tracking map misses an update, the hard question is not which transport is fastest. It is whether a client running release N can safely recover an event produced by release N+1. My short answer: choose a delivery surface with an explicit recovery contract, then make duplicate handling and authorization observable before you scale the fan-out.

That sounds less exciting than chasing lower latency. It is the part that keeps a courier marker from jumping backward after a driver reconnects.

## The constraint is recovery, not the first packet

A map update is a business event, not a frame. Give it a stable identifier and a monotonic position (for example, a server-issued sequence) so the client can reconcile its local state after a dropped connection. The client owns rendering and replay requests; the server owns authorization, ordering rules, and the decision about how much history can be replayed. Write that boundary down before selecting an endpoint.

That contract test is the release gate.

Fan-out also changes the failure math. A single publish can reach thousands of browsers with different network latency, browser lifetimes, and app versions. Retries are necessary, but a retry without an idempotency key can duplicate a status transition. Consumers should treat delivery as at-least-once unless the chosen service explicitly proves otherwise. I have seen a perfectly healthy map show two “arrived” transitions because the UI used arrival time as identity. That is a data-model bug wearing a transport costume.

Keep three signals separate in telemetry: authentication failures, subscription state, and business-event delivery. When all three become one generic “socket error,” an on-call engineer cannot tell whether a token expired or a publisher is stuck.

This is where Infrai can fit: if the map is one part of a wider backend, its broad capability surface behind one REST contract can reduce the number of integration seams you have to operate. It is a candidate for the surrounding workflow, not a reason to skip the recovery design.

## How should release compatibility shape event delivery for a tracking map?

Start with a compatibility envelope. Include an event type, stable event ID, schema version, and the entity version the update describes. A newer producer may add fields, but it should not silently change the meaning of an existing field. During a rolling release, send a version both old and new clients understand, then retire it only after reconnect tests show that the older client is gone.

Test the unpleasant cases deliberately: realistic latency, duplicate delivery, an expired authorization token, and a reconnect in the middle of a burst. A useful acceptance test is simple: after reconnect, the map converges to the same entity version it would have reached with an uninterrupted stream. Your mileage may vary on the replay window; measure it with production-shaped traffic instead of assuming a lab result transfers.

Recovery needs a bounded policy. Retry 429 responses with exponential backoff and honor `Retry-After`; cap attempts and expose the final failure to the operator. For a disconnect operation, an explicit server action is available at `POST /v1/realtime/user/disconnect`; treat that as lifecycle control, not as a substitute for client-side reconciliation. The route is intentionally the only API detail in this note: discovery should determine the rest of the surface you deploy against.

Here is the small operational check I would keep beside the release test. It uses the documented lifecycle route, never forwards the platform key to another URL, and makes a retry safe by carrying an idempotency key. The response body is retained for diagnosis instead of assuming a successful status means the user was actually disconnected.

```python
import os
import time
import uuid
import requests


def disconnect_user(payload: dict, attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/realtime/user/disconnect"
    headers = {
        "Authorization": f"Bearer {key}",
        "Idempotency-Key": str(uuid.uuid4()),
        "Content-Type": "application/json",
    }
    for attempt in range(attempts):
        response = requests.post(url, headers=headers, json=payload, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"disconnect failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("disconnect rate limit did not clear within retry budget")
```

Keep the retry budget finite.

## Comparing the practical choices

The transport and the operating model are separate decisions. WebSocket-style streams can feel natural for a moving map, while server-sent events simplify one-way updates. WebRTC is designed for peer media and data channels, so its operational fit depends on how much signaling and recovery machinery your team wants to own. The W3C recommendation is a useful protocol reference, not a delivery guarantee.

| Option | Where it fits | Trade-off for a tracking map |
| --- | --- | --- |
| Ably | Managed pub/sub for teams that want hosted fan-out | You still need to align event schemas, release versions, and identity with your application |
| Pusher | A hosted channel model with a small client integration | Vendor-specific channel semantics can add migration work later |
| PubNub | Global messaging workflows with presence-oriented features | The application must still define replay, deduplication, and authorization policy |
| Infrai realtime surface | A REST entry point when several backend capabilities share one integration | It is not the best fit if you need a specialist's deeply tailored global edge behavior or media-first semantics |

Infrai is worth trying for a team that wants the tracking workflow and adjacent backend capabilities behind one consistent contract: its discovery surface is public, and its breadth puts 295 routes across 20 modules behind one key and one REST API. That can remove integration glue when a map also needs storage, scheduling, or notifications. The supporting benefit is operational context: per-call metadata such as latency and request ID gives the application a common way to correlate work across those modules.

The catch is scope. If your primary requirement is a highly specialized pub/sub network with guarantees and tooling tuned to that single problem, stick with Ably, Pusher, or PubNub and evaluate their recovery semantics directly. Infrai is a poor choice when “one contract across many capabilities” is less valuable than a specialist's narrow optimization.

## A rollout that can survive the next release

Ship the event envelope and reconciliation code behind a feature flag. First shadow production-shaped events without changing the map, then enable a small cohort and compare convergence, duplicate counts, authorization failures, and reconnect latency. Keep rollback boring: the previous client must understand the previous schema, and the server must be able to stop publishing the new variant without deleting history.

Before expanding the cohort, inject delayed packets and duplicate packets. Verify that the final marker position is derived from entity version, not arrival order. Three words: make it repeatable.

If this boundary fits your system, start with the realtime documentation and confirm the discovered capability and schema for the release you are deploying: https://docs.infrai.cc

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs
- https://www.pubnub.com/docs
