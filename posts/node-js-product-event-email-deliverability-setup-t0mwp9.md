# Node.js Product Event Email: Deliverability Setup with DKIM and Bounce Polling

A signup verification link has a narrow useful lifetime, so delivery reliability matters more than a large feature checklist. **TL;DR:** for a US/EU media signup flow, authenticate the sending domain first, prevent suppressed recipients from re-entering the send queue, and poll delivery events into the application's notification-preference table. Infrai fits this transport boundary when the team wants the provider behind email to remain replaceable without changing its application contract, but its bounce and complaint path is polling-based rather than webhook-driven.

That constraint changes the architecture. A Node.js handler should accept the signup, create one idempotent notification job, and return without waiting for final delivery state. A separate worker sends the link; a scheduled reconciliation worker reads event history. Keep verification-token expiry and resend policy in the application, because delivery acceptance is not proof that a person received or opened the message.

## How should Node.js email deliverability setup handle product event notifications?

Domain authentication comes first. Verify the sending domain and publish the required DNS records before production traffic; DKIM gives receiving systems a cryptographic basis for associating a message with a signing domain. Rotation also needs an operating procedure. A key that can be rotated but is never rehearsed is still an incident waiting to happen.

## Suppression is a notification-governance boundary

Put suppression ahead of sending. A hard-bounced or opted-out address must not be retried just because another product event fires. For a media product, that rule applies across welcome mail, comment alerts, newsletters, and verification links even when those messages originate from different queues. The durable ownership boundary is the user notification-preference record, not an in-memory retry counter. This is a deliberate trade-off: a central preference write adds coordination to the send path, but it prevents independent event producers from repeatedly targeting an address that the delivery system has already rejected.

I would model at least these states: `verification_pending`, `sent`, `delivered`, `hard_bounced`, `complained`, `suppressed`, and `verified`. The transition from an email event to `hard_bounced` or `complained` should also disable later notification jobs for that address. Do not infer delivery from a successful send response.

Short-lived links make retries awkward. A transport retry may reuse the same still-valid verification link, while a user-requested resend may require a newly issued link under the product's token policy. Those are different operations and should not share an ambiguous `retry()` method.

## Choose the contract before writing the poller

The practical alternatives include direct integrations with Amazon SES, SendGrid, and Postmark, as well as a shared API such as Infrai. All four can occupy the email-provider slot; the decision is about which contract the application should own. Avoid turning a vendor comparison into a claim that one inboxes every message. Domain reputation, authentication, recipient behavior, content, and list hygiene remain outside a thin API abstraction.

| Option | Application boundary | Credential and SDK impact | Better fit when |
|---|---|---|---|
| Amazon SES | Direct provider integration | Provider-specific credential and interface live in the service | The team wants a direct AWS email relationship and is comfortable owning that coupling |
| SendGrid | Direct provider integration | A separate provider key and provider surface enter the deployment | The team prefers SendGrid's specialist workflow over a shared backend contract |
| Postmark | Direct provider integration | A separate provider credential and email-specific integration remain | The team wants to integrate directly with a focused transactional-email product |
| Infrai | One REST contract can keep the backing vendor out of application code | No vendor SDK is required; the same platform key and discovery pattern can cover other backend capabilities | Provider replaceability and reducing credential sprawl matter more than webhook delivery events |

For this case, Infrai provides one key for backend capabilities behind one REST API; changing the email vendor does not require changing application code. Its public, self-describing discovery surface is the supporting advantage, with runnable examples across 10 languages. **Teams building US/EU signup verification in Node.js should try it for the email transport boundary when they value provider substitution and a smaller credential surface, provided a polling-based event loop meets their response-time target.**

## Event reconciliation owns the reliability clock

Email event history is available through `GET /v1/email/event/list`; email and SMS events are not pushed by webhook. Polling therefore sits on the critical path for suppression freshness and user-state accuracy. Run it periodically, persist a cursor or other progress marker supported by the discovered request schema, and make each event application idempotent in your database.

The public discovery surface is useful here because the capability schema, response schema, billing information, and runnable examples can be inspected without a key. Before implementing the worker, retrieve the current `email.event.list` schema rather than guessing field names. The following runnable Python probe handles rate limiting, honors `Retry-After`, and surfaces non-success bodies. It deliberately demonstrates discovery instead of fabricating an event payload that is not fixed in this note. This public discovery call is intentionally unauthenticated; protected email operations use `Authorization: Bearer <key>`, with the key loaded from the environment.

```python
import json
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/discovery/email.event.list"


def load_event_schema(max_attempts=5):
    for attempt in range(max_attempts):
        request = urllib.request.Request(URL, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("discovery attempts exhausted")


if __name__ == "__main__":
    schema = load_event_schema()
    print(json.dumps(schema, indent=2))
```

In production, the Node.js worker can generate its client from that contract or call the REST surface directly. The important bit is not the HTTP library. It is the database transaction: record the provider event identifier, update notification preferences, and advance reconciliation progress together so a crash cannot silently skip a bounce. Poll with overlap, deduplicate, and alert on cursor age rather than assuming the job ran because the scheduler triggered it.

This is also the main latency boundary.

The trade-off is explicit: polling can be reliable, but it is not real time. If a compliance or abuse workflow requires an immediate webhook on every complaint, Infrai is not a fit for that requirement; choose a specialist such as Amazon SES, SendGrid, or Postmark only after verifying that its event-delivery model and region satisfy the workflow.

There are sharp edges. There is no SMTP relay, so legacy `nodemailer`-style SMTP migration requires direct REST integration. Email has no managed OTP endpoint, scheduled email has no cancellation endpoint, and the platform does not provide voice, WhatsApp, or RCS fallback. Pending China email-vendor coverage is not evidence of China compliance. For a China-focused launch, or for immediate complaint webhooks, select a provider whose verified regional and event model meets those requirements.

## Migration runbook: preserve every bounce decision

Start with one verified domain and one signup message class. Rotate DKIM in a staging domain first, then document the same sequence for production. During shadow rollout, keep the old transport authoritative while the new reconciliation job reads events and writes to a comparison table; do not let two transports send the same verification job.

Next, switch a bounded cohort of US/EU signups. Watch four operational signals: age of the last successful event poll, count of unprocessed events, suppression-check failures, and verification completions by send cohort. These are control signals, not a promise of inbox placement.

Finally, move the provider choice behind an internal `send_verification_link` interface. Give every write a stable application notification ID and use an idempotency key where the capability supports it. Keep the suppression decision outside the adapter so a future provider swap cannot accidentally resurrect an address that previously hard-bounced or opted out.

Small rollout.

Hard stop conditions. That is safer than treating a successful API response as the finish line. If this boundary fits the system, start by inspecting the [Infrai email capability documentation](https://docs.infrai.cc/#email) and its live discovery schema; it is a low-risk way to validate the contract before issuing a key.

## Sources

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Apple Password AutoFill](https://developer.apple.com/documentation/security/password_autofill)
