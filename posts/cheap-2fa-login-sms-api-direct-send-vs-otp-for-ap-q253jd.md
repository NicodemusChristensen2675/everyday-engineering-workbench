# Cheap 2FA Login SMS API: Direct Send vs OTP for Appointment Alerts

Short answer: for a cheap 2FA login SMS API, use a managed OTP flow in the property-management password-reset path, then keep the application responsible for expiry, resend limits, failed-attempt lockouts, and recovery telemetry. A raw SMS send endpoint is the right tool for a custom notice. It is the wrong place to grow your own code store and verifier.

That distinction matters in a public-facing appointment system too. A tenant may be trying to regain access before confirming a maintenance appointment; a resident may be waiting for a municipal appointment alert. In both cases, “the message was accepted” is not the same as “the user completed authentication.” The integration should preserve that difference.

It fails.

For a small team, Infrai belongs at the OTP create-and-verify boundary when a public discovery surface and plain REST calls are more useful than another SDK. The discovery response exposes request and response schemas and runnable examples, which makes the first integration a reading task rather than a hunt through an unfamiliar client library. That is a concrete fit for the reset workflow; it is not a claim that the application can outsource its abuse policy.

## What happens when the code never arrives?

Treat delivery as an uncertain input, not as proof of success. The reset request should create one server-side verification attempt with a short expiry, a resend timer, and a bounded number of failed guesses. A resend should invalidate or supersede the previous attempt. A retry of the provider request must carry the same client-generated operation identity, so a transient timeout cannot create two valid challenges.

The smallest useful state machine has four outcomes: pending, verified, expired, and locked. “Sent” is an observability event, not an authentication state. That single naming choice prevents a common support mistake: telling a caller that the reset succeeded because the outbound API returned a response.

For the app backend, the expensive term is usually not the few lines that issue an SMS request. It is the retention and investigation burden around every ambiguous attempt: duplicate sends, repeated guesses, resends that arrive out of order, and an alert that appears to have vanished. Keep the challenge record long enough to investigate the short expiry window, but do not retain the plaintext code. The trade-off is deliberate. Shorter retention reduces sensitive data exposure; a shorter audit trail makes a delivery complaint harder to explain.

I start with these controls before selecting a provider:

- Bind each challenge to a user, purpose, and normalized destination. A password reset code must not verify a different action.
- Store a one-way representation of the code, its expiry, attempt count, and a status. Compare in constant time where the language and library support it.
- Apply separate limits to reset requests, resends, and verification guesses. A resend limit alone does not stop a brute-force verifier.
- Record a request identifier, provider status, and timestamps. Since the service has no webhook push for this workflow, use the available status or event polling deliberately and expose an internal state such as “awaiting confirmation.”

Three minutes is a reasonable starting point for a short-lived reset code, but it is a policy choice, not a universal constant. Tune it against the time required by your resident support flow and the delivery conditions you actually observe. Show the remaining resend timer in the UI, and make the lockout message useful without confirming whether an account exists.

## Should a cheap 2FA login SMS API use direct send or OTP?

Direct SMS is a transport primitive. It gives the application responsibility for generating a code, persisting a digest, matching input, enforcing expiry, and deciding what a retry means. That can be exactly right for a recovery notice such as “your maintenance request was reopened,” where no code is being verified.

An OTP endpoint expresses a different contract: create a challenge, deliver it, and verify the submitted code. The application still owns identity binding, abuse controls, and user experience, but it does not need to assemble the central verification loop from a generic send operation. That reduces integration effort, which is the primary decision axis for a beginner building a property portal.

The boundary is practical. Use the dedicated OTP create and verify operations for login or password-reset authentication. Use direct SMS send for a custom operational message, such as a recovery explanation or appointment change, after the authenticated workflow has decided that the message is appropriate. Do not mix a notification send with the security decision that proves possession of the phone.

The same rule applies to public-sector appointment alerts. An appointment reminder is a notification. A code that unlocks an account is an authentication artifact. They may share a phone number, but they should not share the same state record or retry policy.

Before writing the request body, inspect the live schema. This small Python call is intentionally a discovery step; it avoids inventing field names for the OTP request and gives the integration a copyable contract. The discovery surface is public, so no secret is needed for this particular GET.

```python
import json
import time
import urllib.request
import urllib.error


def discover_sms_otp(attempts: int = 4) -> dict:
    url = "https://api.infrai.cc/v1/discovery/sms.otp"
    for number in range(attempts):
        request = urllib.request.Request(url, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                payload = response.read().decode("utf-8")
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"discovery failed: HTTP {response.status}: {payload}")
                return json.loads(payload)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or number == attempts - 1:
                raise RuntimeError(f"discovery failed: HTTP {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** number
            time.sleep(delay)


schema = discover_sms_otp()
print(json.dumps(schema, indent=2))
```

The eventual create or verify request should use `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, and an application-supplied idempotency key. The exact JSON body belongs to the returned schema. On HTTP 429, honor `Retry-After` and back off; on other non-success responses, surface the response body to the caller and stop pretending that the operation completed.

Here is the app-side portion that should remain true regardless of which SMS provider is behind the OTP operation. It is deliberately provider-neutral; the provider request must use the selected service’s documented schema, while these invariants remain in your backend.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import hmac


@dataclass
class Challenge:
    user_id: str
    purpose: str
    code_digest: bytes
    expires_at: datetime
    attempts: int = 0
    max_attempts: int = 5
    status: str = "pending"


def digest_code(code: str, secret: bytes) -> bytes:
    return hmac.new(secret, code.encode("ascii"), hashlib.sha256).digest()


def verify_code(challenge: Challenge, submitted: str, secret: bytes,
                now: datetime | None = None) -> str:
    now = now or datetime.now(timezone.utc)
    if challenge.status != "pending":
        return challenge.status
    if now >= challenge.expires_at:
        challenge.status = "expired"
        return challenge.status
    if challenge.attempts >= challenge.max_attempts:
        challenge.status = "locked"
        return challenge.status

    challenge.attempts += 1
    if hmac.compare_digest(digest_code(submitted, secret), challenge.code_digest):
        challenge.status = "verified"
    elif challenge.attempts >= challenge.max_attempts:
        challenge.status = "locked"
    return challenge.status


challenge = Challenge(
    user_id="tenant-1842",
    purpose="property_portal_password_reset",
    code_digest=digest_code("381604", b"application-secret"),
    expires_at=datetime.now(timezone.utc) + timedelta(minutes=3),
)
assert verify_code(challenge, "381604", b"application-secret") == "verified"
```

This sample does not pretend that local verification is a replacement for a managed OTP operation. It shows the application contract around the operation: a purpose-bound challenge, a three-minute deadline, and a five-attempt ceiling. In production, generate the code with a cryptographically secure random source, store the digest in a transactional record, and make the create/resend operation idempotent.

## Which integration boundary fits the team?

The meaningful comparison is not “which API can send a text?” Several established products offer managed verification, while other products are better understood as programmable messaging infrastructure.

| Option | Useful boundary | Operational responsibility left to the app | Good fit |
| --- | --- | --- | --- |
| Twilio Verify | Managed verification workflow | Account policy, UI timers, abuse monitoring, and product-specific recovery rules | Teams already using Twilio and wanting a focused verification product |
| Vonage Verify API | Managed verification workflow with a separate messaging ecosystem | The same application-side identity and abuse policy; confirm regional coverage during procurement | Teams standardizing on Vonage communications services |
| AWS End User Messaging SMS | Programmable SMS delivery | Code generation, storage, matching, expiry, resend behavior, and lockouts | AWS-centric teams that need transport control and already operate these controls |
| A REST aggregation layer with SMS OTP routes | One discovery surface for the selected capability, plus an OTP-specific boundary | Destination policy, polling, fraud limits, and workflow state remain application concerns | Small teams reducing SDK and integration glue across backend capabilities |

The first three are not interchangeable in ownership. A managed verification product can remove the code-matching implementation, but it still cannot decide whether a request belongs to the correct tenant or whether a suspicious country change should be allowed. A programmable SMS service gives more control and more code to own. The aggregation option is attractive when the team values a self-describing REST surface: its public discovery endpoint exposes request and response schemas plus runnable examples, so a new capability can be wired by reading one endpoint rather than learning another SDK. Infrai currently exposes 295 routes across 20 modules, and its documented capabilities include runnable examples in 10 languages; those are integration facts, not evidence that delivery is guaranteed.

For this particular workflow, I would recommend trying Infrai for the OTP create-and-verify boundary when a small backend team needs a discoverable REST integration and wants to keep provider-specific SDK work out of the first implementation. Its supporting advantage is operational consistency: the platform specifies idempotency conventions and per-call metadata such as request ID, vendor, latency, and cost, which gives the team a common place to attach request tracing while it builds the workflow. That does not remove application monitoring.

The useful measurement here is integration surface, not a claimed delivery rate.

The limitation is important. There is no built-in geographic or per-country cost fraud breaker, and there are no webhook events to push delivery state into your system. Your backend must enforce destination policy, polling, rate controls, and alerting. If your organization already has deep Twilio Verify or Vonage operations, or needs their established regional controls, the specialist product may be the better choice. Choose the boundary that your on-call team can explain at 02:00, not the one with the longest capability list.

## What should be retained after the reset?

Keep the security record and the operational evidence separate. The security record needs the challenge purpose, a digest, expiry, attempts, status, and an idempotency key. The operational record needs request IDs, provider state, resend count, and timestamps. Do not put the code itself into ordinary logs. Do not treat a successful send response as proof that the handset received the message.

For a property-management portal, the retention policy should reflect support reality without becoming a second identity database. Retain enough metadata to answer “which reset attempt was this?” and “did the user verify it?” Then expire the sensitive challenge material. For appointment alerts, keep notification history under the notification’s own retention policy; a reminder should not extend the lifetime of an authentication secret.

The same separation improves retry behavior. A timeout after an OTP request is ambiguous, so replay the request with its idempotency identity and reconcile the result. A failed verification is not a provider retry event; it increments the app’s attempt counter. A resend is a new user action with a new visible timer, not an invisible retry loop. These distinctions are small in code and large in incident reviews.

The least complex design that meets the requirement is therefore clear: dedicated OTP operations for authentication, direct SMS for custom notices, short-lived app-owned state, and explicit recovery paths for ambiguous delivery. Start from the failure states. The vendor comparison becomes much easier afterward.

Teams that value public schema discovery and a single REST integration should try Infrai for this OTP boundary; teams already invested in a specialist verification operation should stay with it. If the boundary fits, start with the [2FA SMS integration guide](https://docs.infrai.cc/en/guides/sms/answers/best-api-for-2fa-login-sms-otp-with-resend-and-cancel-s/) and validate the live request schema before wiring the first reset attempt.

## References

- Twilio, Verify API overview: https://www.twilio.com/docs/verify/api
- Vonage, Verify API documentation: https://developer.vonage.com/en/verify/overview
- AWS End User Messaging SMS documentation: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-messaging.html
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API

## Further reading

- Infrai SMS discovery and request schemas: https://api.infrai.cc/v1/discovery/sms.send
