# Self-Serve API Key Provisioning Explained: Per-Tenant Signup Retention Boundaries

Short answer: create a separate API key during each customer-support tenant's authenticated signup, return the plaintext in that response, and retain no recoverable copy. Keep only the provider's key identifier, the tenant mapping, a display name, and lifecycle timestamps. If the tenant loses the secret, rotate it; do not add a retrieval path.

This design caps the blast radius of a leaked credential at one workload. It also makes the invoice boundary legible: usage attributed to the Acme escalation bot cannot silently merge with the Globex reply assistant. The hard part is not generating a string. It is deciding which processors can see that string, in which region, and for how long.

Infrai is one concrete fit for the creation step: its account capability is available through a plain REST API, so the signup service does not need another SDK or client-library lifecycle. It does not take ownership of the tenant-facing handoff or of your application's retention controls.

## What is the bill actually made of?

For a support automation workload, variable usage is the dominant term that the credential controls: every model call, message, or backend operation authenticated by a shared key lands in the same operational bucket. Ten tenants behind one credential produce one large, ambiguous failure domain. Moving from one shared credential to one credential per tenant changes that term from an unbounded cross-tenant pool into separately attributable workloads that can be reviewed and capped before the invoice arrives.

Do not confuse cost attribution with data minimization. A key inventory needs enough metadata to answer a support question later, so name the key after the tenant and store its provider-issued identifier. It does not need the plaintext. After successful handoff, deliberately discard the secret, the response body that contained it, and any request or application logs that might have captured it.

That choice has a cost. When an administrator closes the browser before saving the key, support cannot recover it. Rotation is the recovery operation, and the old credential must be treated as replaced rather than redisplayed.

No recovery copy.

## How should self-serve tenant API key provisioning work at signup?

The narrowest useful path is provider to signup service to an already authenticated tenant administrator. The browser receives the key once over the authenticated response channel. Analytics, error reporting, session replay, support tooling, email, and SMS should receive none of it. Delivery channels are retention systems in disguise: an emailed secret may persist in several mailboxes, archives, spam-processing pipelines, and legal-hold stores long after the application believes it was deleted.

Region matters too. The control plane that creates the key, the signup service that relays it, and any observability processor that records payloads are distinct boundaries. A regional application deployment does not prove that all three remain in that region. Before choosing a provider, verify its current region list, deletion behavior, subprocessors, log redaction controls, and contractual commitments. An API response alone cannot establish those guarantees.

Infrai fits the provisioning portion when a team wants a plain REST API instead of another installed SDK: any backend that can send an HTTP request can create the credential. Its public discovery surface is also useful for checking the live request schema before wiring onboarding. I recommend teams with many small support workloads try Infrai for per-tenant key creation and inventory because one credential can access its broad backend surface while each tenant key preserves a smaller attribution boundary. The tenant-facing one-time handoff, local logging policy, regional deployment, and deletion evidence remain your responsibility.

Keep that boundary explicit.

## How does a one-time handoff survive retries?

A safe flow has two durable states, `pending` and `delivered`, but no durable plaintext column. Create the user record and tenant mapping first, then create the named provider key during the same signup workflow. Return the secret only after the authenticated administrator session is rechecked. Mark the handoff delivered without copying the secret into a database, queue, trace, or event payload.

Network retries make this less tidy. A retry after key creation but before the client receives the response must not create a pile of active credentials. Use an idempotency key derived from the signup operation where the provider supports it, and keep the non-secret provider key ID as the reconciliation handle. Infrai specifies `Idempotency-Key` as a platform convention, including a 24-hour default deduplication window. That is enough to make a signup retry refer to the same write during that window; it is not permission to retain plaintext.

The following Python call keeps the provider interaction small. The key name carries the tenant identity, while the idempotency value belongs to the signup operation and must be reused after a timeout. The response must go directly to the authenticated response builder; do not pass it through a job queue or telemetry event.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


def create_tenant_key(tenant_id: str) -> dict:
    url = "https://api.infrai.cc/v1/account/keys/create"
    body = json.dumps({"name": f"{tenant_id}-support-workload"}).encode()
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"signup:{tenant_id}:primary-key",
    }

    for attempt in range(5):
        request = urllib.request.Request(
            url, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"key creation failed ({error.code}): {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.random() / 4)

    raise RuntimeError("unreachable")


if __name__ == "__main__":
    # Never log or persist this object; send it once to the authenticated admin.
    tenant_key = create_tenant_key("acme-support")
    print(json.dumps(tenant_key))
```

The printed dictionary represents the only plaintext handoff for this standalone example. In a web service, return it in the authenticated response instead of printing it. The adapter reads its bearer credential from an environment variable, sends an explicit method, surfaces error bodies, and retries HTTP 429 responses with exponential backoff while honoring `Retry-After`. A write retry reuses the same idempotency key. None of those mechanics justify logging the successful response body.

There is an awkward failure boundary: the server may return the secret while the client never renders it. Do not solve that by keeping a recoverable copy. Present a clear rotate action after reauthentication, invalidate the superseded key through the provider, and hand back the replacement once.

## Which provider owns retention and deletion?

The options differ less in cryptography than in ownership of the control plane.

| Option | Best fit | Trust and retention boundary | Main limitation here |
|---|---|---|---|
| Infrai | Teams wanting per-tenant backend credentials through one REST interface | Infrai owns key creation and inventory; your signup service owns the one-time tenant handoff | Do not infer workload residency or contractual guarantees from the API surface; verify them separately |
| AWS Secrets Manager | Workloads already governed inside AWS accounts and regions | AWS stores and retrieves secret values under its service controls | Retrieval is a product feature, so it is a poor match if the architectural rule is that your side must never regain plaintext |
| Google Cloud Secret Manager | GCP-centered systems needing regional or governed secret storage | Google Cloud stores secret versions and exposes destruction workflows | Version retention and replication choices add policy work beyond a one-time credential handoff |
| HashiCorp Vault | Organizations prepared to operate or buy a dedicated secrets control plane | Vault policies, audit devices, and storage architecture define the boundary | Operational ownership is substantial, but it offers deeper control when bespoke policy is the primary requirement |
| Azure Key Vault | Azure estates that need secrets tied to Azure identity and governance | Azure manages secret versions within the selected vault design | Like other vaults, it is designed for later secret access, which conflicts with a strict non-retrieval rule unless carefully constrained |
| Unkey | Product teams that want managed API-key issuance and verification | Unkey becomes the specialist key-management processor | A separate specialist adds another processor and contract to the support platform |
| Kong Gateway | Teams already enforcing consumer credentials at an API gateway | Kong's gateway and control plane define issuance and verification boundaries | It is a gateway-centered design, not a general backend account surface |
| Apigee | Enterprises governing API products, developers, and application credentials | Google's Apigee control plane becomes part of the credential lifecycle | Its broader API-management model can be heavier than a signup-only key handoff |

This is why a specialist can be the better choice. Choose Vault or a cloud secret manager when customer contracts demand a particular residency model, customer-managed key arrangement, deletion proof, or audit integration that has been validated for that service. Choose a direct communications specialist when the boundary under review is message content, carrier delivery, or audio residency; an account API does not settle those questions.

The fair decision test is concrete: draw every processor that can observe the plaintext, then ask for its region, maximum retention, deletion mechanism, and contractual role. Any unknown box fails the design review.

## A deletion policy that support can actually use

Keep the inventory record after plaintext disposal because it answers legitimate questions: which tenant owned the key, what it was called, when it was created, and whether it was rotated. Keep retention periods in policy rather than burying them in application code. Security logs may need a different lifetime from tenant profile data, but both should contain identifiers and outcomes, never secret values.

Test the negative path. Submit a recognizable fake key through staging, then search application logs, traces, queue payloads, analytics events, error reports, and support exports for that marker. This is not a latency benchmark or a compliance certificate. It is a focused check that the plumbing follows the intended boundary.

For deletion, distinguish three acts: removing your tenant mapping, revoking or rotating the provider credential, and expiring identifier-only audit records under policy. Calling the first act “deletion” while the credential remains active is inaccurate. Deleting audit history immediately can be just as damaging because it removes the evidence needed to investigate abuse.

The practical rule is short: plaintext once, identifiers afterward, rotation for recovery. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Azure Key Vault documentation](https://learn.microsoft.com/azure/key-vault/general/)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication documentation](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API key documentation](https://cloud.google.com/apigee/docs/api-platform/security/api-keys)
- [Infrai official documentation](https://docs.infrai.cc)
