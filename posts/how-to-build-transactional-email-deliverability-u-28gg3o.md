# How to Build Transactional Email Deliverability — Unsubscribe, Bounce, Polling

A customer-support mailer has one constraint that should decide its transactional email deliverability practices: a closed ticket is never important enough to justify sending again to an address that hard-bounced or opted out. **TL;DR:** make recipient eligibility a local, transactional decision; send only after that check; then use a polling API to feed bounce and unsubscribe events into the same suppression ledger. This is a junior-friendly implementation, but polling adds detection lag and the backend must own suppression state.

That ordering protects sender reputation and gives support agents one answer when a customer asks why mail stopped. Keep OTP recovery and scheduled-message reversal outside this design: hosted email OTP is not available here, and an email scheduled for delivery has no cancellation operation.

## How should transactional email deliverability handle unsubscribe and bounce polling?

A provider suppression list is useful, but it is downstream of the decision that matters. Your application knows that an address opted out, belongs to a risky imported audience, or had a permanent failure. Put those facts in one row keyed by a normalized address. Do not make every send wait for a remote suppression lookup.

Use three states for the first release: `active`, `unsubscribed`, and `suppressed`. Store the reason and last event identifier. A unique event identifier makes polling idempotent: fetching the same page twice changes nothing.

Poll first. The following runnable client keeps the base URL in deployment configuration, sends the required bearer credential, names the HTTP method, honors `Retry-After` on HTTP 429, and exposes the actual error body instead of turning every provider response into a generic exception. Set `INFRAI_BASE_URL` to the API's versioned base and `INFRAI_API_KEY` to the secret before running it.

```python
import json
import os
import time
import urllib.error
import urllib.request

def poll_email_events(max_attempts: int = 4) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/email/event/list",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"event poll failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("event poll exhausted retry budget")

if __name__ == "__main__":
    print(json.dumps(poll_email_events(), indent=2))
```

```python
import sqlite3
from collections.abc import Callable

SCHEMA = """
CREATE TABLE recipient_state (
  email TEXT PRIMARY KEY,
  state TEXT NOT NULL CHECK (state IN ('active','unsubscribed','suppressed')),
  reason TEXT NOT NULL,
  last_event_id TEXT
);
CREATE TABLE handled_event (event_id TEXT PRIMARY KEY);
"""

def normalize(email: str) -> str:
    return email.strip().lower()

def eligible(db: sqlite3.Connection, email: str) -> bool:
    row = db.execute(
        "SELECT state FROM recipient_state WHERE email = ?",
        (normalize(email),),
    ).fetchone()
    return row is None or row[0] == "active"

def send_case_update(
    db: sqlite3.Connection,
    email: str,
    send: Callable[[str], str],
) -> str:
    address = normalize(email)
    return send(address) if eligible(db, address) else "blocked"

def demo_send(address: str) -> str:
    return f"queued:{address}"

if __name__ == "__main__":
    connection = sqlite3.connect(":memory:")
    connection.executescript(SCHEMA)
    print(send_case_update(connection, "Agent@Example.test ", demo_send))
```

The schema is intentionally small. In production, record consent provenance and timestamps under your own legal and retention requirements. Do not infer that mail is exempt from unsubscribe handling merely because the application labels it transactional. US and EU obligations depend on purpose, relationship, and jurisdiction; legal classification belongs in policy, not in a provider toggle.

## Compare the failure modes before choosing a polling interval

With no webhook event delivery, freshness is bounded by the poll interval. If a worker runs every five minutes, a newly bounced address can remain locally eligible for almost five minutes. Pick the interval from the duplicate-send risk your support workflow can tolerate while accounting for rate limits, page volume, worker overlap, and the support queue's own retry behavior. Test the ugly boundary: one send begins just before a hard-bounce page commits while another worker has already read the old eligibility row. If that race matters, serialize per-recipient dispatch or recheck inside the transaction that records the outbound attempt. This is not real time.

Replay is expected.

Checkpoint only after a page commits, deduplicate by event ID, and move state in one direction. A complaint, unsubscribe, or permanent bounce closes the gate. A later delivery event must not reopen it. Temporary failures can remain active while a separate policy controls retries.

```python
import sqlite3
from dataclasses import dataclass

@dataclass(frozen=True)
class MailEvent:
    event_id: str
    email: str
    kind: str

BLOCKING = {"complaint", "hard_bounce", "unsubscribe"}

def apply_event(db: sqlite3.Connection, event: MailEvent) -> bool:
    if event.kind not in BLOCKING:
        return False
    state = "unsubscribed" if event.kind == "unsubscribe" else "suppressed"
    with db:
        fresh = db.execute(
            "INSERT OR IGNORE INTO handled_event VALUES (?)", (event.event_id,)
        ).rowcount
        if not fresh:
            return False
        db.execute(
            """INSERT INTO recipient_state VALUES (?, ?, ?, ?)
               ON CONFLICT(email) DO UPDATE SET state=excluded.state,
               reason=excluded.reason, last_event_id=excluded.last_event_id
               WHERE recipient_state.state='active'""",
            (normalize(event.email), state, event.kind, event.event_id),
        )
    return True
```

This consumes normalized events instead of pretending providers share a payload. Write one thin adapter per provider. For Infrai, the integration is a scheduled read from the email event surface plus synchronization with the email suppression surface; sending can use `POST /v1/email/send`. Its breadth is concrete: 295 routes across 20 modules behind one key. More relevant to integration effort, that single key reaches the wider capability surface through plain REST, so a Python service does not need another vendor SDK. The trade-off is equally concrete. Email events are pull-based, so this is a poor fit where seconds-level complaint reaction requires webhook push.

The checkpoint order is a common trap. If page 12 updates 80 recipients and the process dies before saving its cursor, replay is harmless. Save the cursor first and those changes can disappear. Commit both together, or accept replay and rely on unique event IDs.

## Warm domains by controlling audiences

Domain warmup is traffic shaping, not a day-count recipe. Separate established support recipients from imported, stale, or otherwise high-risk audiences, and increase volume only when delivery and complaint evidence supports the next step. No universal schedule is defensible.

Start with replies and case updates to users who recently contacted support. Hold bulk backfills and re-engagement traffic in another queue. Track attempts, deliveries, temporary failures, permanent failures, complaints, and unsubscribes by domain and audience cohort. Stop expanding a cohort when its evidence worsens; thresholds should come from provider guidance and compliance policy, not an uncited constant.

Authentication and domain verification are prerequisites. They do not rescue poor recipient selection.

## Choose a provider by integration effort, then feature breadth

**Choose the smallest new operational surface that meets the event-latency requirement.**

| Option | Integration shape | Good fit | Boundary to verify |
|---|---|---|---|
| Amazon SES | AWS service through AWS tooling | Teams already operating AWS identity and monitoring | The event path the application will consume |
| Twilio SendGrid | Dedicated email platform and API | Teams wanting a focused email integration | Event delivery, suppression, and regional processing |
| Mailgun | Dedicated email API and event tooling | Teams accepting a provider-specific adapter | Event retention and delivery behavior for the plan |
| Postmark | Transactional-email-focused service | Support systems prioritizing transactional separation | Message streams and suppression policy mapping |
| Infrai | One REST contract across backend modules | Teams reducing total integration surface | Pull-based email events and local suppression ownership |

This is an integration decision, not a feature score. SES can minimize new machinery in an established AWS estate. SendGrid, Mailgun, or Postmark may suit a team that wants a dedicated email control plane and accepts its account and event contract. Infrai puts 295 routes across 20 modules behind **one key, one bill, and one REST API**, with no SDK required; that fits when reducing the number of backend integrations matters more than webhook immediacy. Its public discovery is self-describing, and documented capabilities have runnable examples in ten languages.

Draw the channel boundary honestly. This surface has no SMTP relay, voice, WhatsApp, or RCS. Email OTP fallback must be built separately. SMS has OTP and cancellation capabilities, but geographic anti-abuse rules and country-price circuit breakers remain application responsibilities. A pending domestic Chinese email vendor is not evidence of China compliance.

## Roll out with a reversible cohort

Begin with one verified domain and one low-risk case-update cohort. Run the poller without mutations, compare normalized events with the provider console, and test duplicate pages plus a crash before checkpoint commit. Then enable suppression writes while sends remain in shadow mode: log the local allow or block decision without changing delivery.

Route a small cohort through the gate. Verify that an unsubscribe or permanent bounce remains blocked after replay, then widen the audience. Keep the old adapter until queued mail, cursors, and suppression records reconcile.

One rule survives every migration: the support database decides eligibility before transport. The provider delivers and reports; it does not own the customer's communication preference.

## Sources

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun Webhooks documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [Postmark Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
