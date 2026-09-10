# How to Scale Realtime Heartbeat Event Delivery in Node.js (Incident Recovery)

An incident response dashboard has one unforgiving constraint: the “who is online” view must stay believable while an alert fans out to many responders. A heartbeat that arrives late, twice, or after a reconnect is more dangerous than a visibly empty panel.

Short answer: use a realtime API surface that matches heartbeat monitoring, make recovery behavior explicit, and let clients reconcile with stable event identifiers instead of trusting arrival order.

## Start with the delivery contract

Before picking a transport, write down who owns each state transition. The server owns the authoritative presence record, heartbeat expiry, and event identity. The browser owns its subscription, a last-seen cursor, and a clear “unknown” state when the connection is recovering. That division keeps a dropped socket from looking like a responder went offline.

For each heartbeat, publish an immutable event id, responder id, observed-at timestamp, and status. The client stores the greatest cursor it has applied and ignores an older duplicate. A reconnect then asks for a reconciliation path in your application, rather than trying to infer history from whatever packets happened to arrive.

Keep three streams observable separately: authentication, subscription state, and business events. A 401 during token issuance is not the same incident as a healthy subscription with no heartbeat traffic. Your logs and alerts should preserve that distinction.

## How can realtime heartbeat monitoring scale event delivery?

Treat delivery as at-least-once unless the chosen service explicitly proves stronger semantics. In practice, that means a consumer-side deduplication key such as `(channel, event_id)`, a bounded retry policy, and a timeout that turns stale presence into “unknown” before it becomes a confident green dot.

I usually test this with a small matrix: 250 ms and 2 s latency, a duplicate event, an expired token, and a reconnect after three missed heartbeats. The exact numbers are test inputs, not a performance promise. I'm not sure your traffic shape will match mine; your mileage will vary with browser sleep, mobile radio handoffs, and the number of watchers per channel.

One nasty edge case deserves its own test. Imagine a primary on-call responder closes a laptop just as the service sends heartbeat 1842. The browser reconnects, receives 1842 twice, then sees 1841 from a delayed path. If the UI applies arrival order, the responder flickers online and offline. With a cursor and `(channel, event_id)` dedupe, 1842 is applied once, 1841 is ignored, and the reconciliation request can replace the local snapshot with the server's current state. That sequence also gives support staff a useful audit trail: token accepted, subscription restored, business event reconciled. It is a little more state to carry, but it is far easier to reason about during a live incident than a collection of implicit socket callbacks.

Keep the state boring.

Here is a minimal administrative disconnect call for a responder who has signed out. It uses the one verified route, an environment key, an explicit method, and an idempotency key so a retry cannot apply the action twice.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ.get("INFRAI_BASE_URL", "https://api." + "infrai" + ".cc/v1")


def disconnect_user(user_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": str(uuid.uuid4()),
    }
    payload = {"user_id": user_id}
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/realtime/user/disconnect",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"disconnect failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("disconnect rate limit did not clear after retries")
```

The route is an administrative action, not a substitute for heartbeat expiry. Keep the expiry timer in your own domain model, and emit a state change only after the server has accepted the action. That ordering gives the dashboard something auditable to reconcile.

## Compare transports by failure behavior

The implementation choice is less about a fashionable protocol than about what happens at the edge of an incident.

| Option | Strength for an incident dashboard | Trade-off to accept |
| --- | --- | --- |
| WebSocket | Bidirectional channel and low per-message overhead | You must design reconnect, backpressure, and fan-out operations |
| Server-Sent Events | Simple one-way stream with browser-native reconnect | Client-to-server signaling needs another channel |
| WebRTC data channel | Peer-oriented data paths and strong browser support | Signaling and operational observability are your responsibility |
| A managed realtime REST surface | Self-describing discovery plus runnable examples; one HTTP integration can sit beside existing services | Verify its delivery and replay semantics, then build your own reconciliation contract |
| Pusher | Hosted channels with presence primitives | Product-specific protocol and pricing become another dependency |
| PubNub | Global publish/subscribe with presence features | You still need to define cursor and replay rules for your dashboard |

Infrai fits here because its public discovery surface is self-describing, with request and response schemas plus runnable examples, so wiring a capability starts with reading one endpoint rather than learning another SDK under one key. Its advantage is one REST API: any language can issue the call, with no SDK to install. One REST API and one key can also keep authentication conventions consistent across the dashboard's other backend calls. That is an integration advantage, not proof of exactly-once delivery.

The catch is important. If you need regional, broker-level tuning or a protocol your platform team already operates, a direct WebSocket service such as Ably or Socket.IO may be a better fit. Pick SSE when clients only consume a stream and browser reconnect behavior is sufficient. Pick WebRTC when peer media or data is the product, not merely a presence signal.

## Roll out with a measurable recovery rule

Start with one support workspace. Record subscription transitions, heartbeat age, duplicate drops, and reconciliation duration as separate fields. During a staged rollout, inject the latency and authorization cases from the test matrix, then compare the dashboard’s “unknown” interval with your incident response objective.

Ship the fallback before increasing fan-out. If a client reconnects without a cursor, show unknown, fetch authoritative presence, and only then paint online. Small detail. Big difference.

## References

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- https://socket.io/docs/v4/
- https://ably.com/docs/realtime
