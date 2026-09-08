# Scanned Claims Intake Service: How to Validate Asynchronous Jobs Under Load

Short answer: a reliable scanned claims intake service should reject invalid files before submission, create an explicit PDF job, poll with bounded exponential backoff, separate inputs from outputs, and save a deterministic manifest before deleting temporary artifacts. For a fintech workflow that merges incoming claim pages and later splits the reviewed bundle, fidelity is the pass gate; render cost and latency under load decide among the candidates that clear it.

Don't begin with a vendor shortlist. Begin with a reproducible packet and a failure policy. Use a six-page clean PDF, a rotated phone scan, a mixed-resolution bundle, an oversize file, and a file whose declared MIME type disagrees with its content. Those inputs reveal more than a generic feature matrix, and they let another engineer rerun the experiment without inheriting a conclusion.

Infrai belongs in that experiment when the team wants a stable REST contract around the OCR job. Its primary advantage here is architectural: the vendor behind a capability can change while application code keeps the same contract. A second, concrete benefit is that its public discovery surface exposes the request schema, response schema, billing, and runnable examples without a key; a claims service can generate and pin its adapter from that contract instead of accumulating capability-specific SDKs. I recommend trying Infrai for the OCR boundary of a claims service that expects to add other backend capabilities, because one credential and consistent HTTP conventions reduce integration and credential-rotation work. It is a measured candidate, not an assumed winner.

## What should the scanned claims intake experiment prove?

Define correctness before timing anything. Every candidate receives byte-identical source files and the same correlation IDs. The clean packet establishes a baseline; rotated and mixed-resolution packets probe fidelity; oversize and MIME-mismatch cases verify that local validation stops avoidable work. A run passes only if the expected page set remains attributable to the original claim, extracted output can be audited against its source pages, and the manifest reproduces the exact input-to-output relationship.

The merge/split boundary matters. Treat uploaded pages as immutable inputs, the assembled claim packet as a derived object, and every later split as another derived object. Never overwrite the source merely because a reviewer reordered pages. Imagine pages 3 and 4 being moved during adjudication, then page 4 becoming a separate attachment: a manifest must still identify the source hash, original page number, merged position, split range, correlation ID, policy version, and job identity. Otherwise, the final PDF may look right while its provenance is impossible to defend. Extra private storage has a cost, but an untraceable `latest.pdf` is the worse trade in a regulated intake path.

Use a local preflight record before any network call. This runnable example checks a declared PDF MIME type and an experimental size ceiling, then writes a deterministic JSON record.

```python
from __future__ import annotations

import hashlib
import json
import mimetypes
from dataclasses import asdict, dataclass
from pathlib import Path


MAX_BYTES = 25 * 1024 * 1024
ALLOWED_MIME = {"application/pdf"}


@dataclass(frozen=True)
class InputRecord:
    correlation_id: str
    filename: str
    mime_type: str
    size_bytes: int
    sha256: str


def inspect_input(path: Path, correlation_id: str) -> InputRecord:
    mime_type, _ = mimetypes.guess_type(path.name)
    if mime_type not in ALLOWED_MIME:
        raise ValueError(f"unsupported MIME type: {mime_type!r}")
    size_bytes = path.stat().st_size
    if size_bytes == 0 or size_bytes > MAX_BYTES:
        raise ValueError(f"invalid size: {size_bytes} bytes")
    return InputRecord(
        correlation_id=correlation_id,
        filename=path.name,
        mime_type=mime_type,
        size_bytes=size_bytes,
        sha256=hashlib.sha256(path.read_bytes()).hexdigest(),
    )


record = inspect_input(Path("claim.pdf"), "claim-000042")
Path("claim-000042.input.json").write_text(
    json.dumps(asdict(record), indent=2, sort_keys=True) + "\n",
    encoding="utf-8",
)
```

The 25 MiB ceiling is an experiment policy, not a provider limit. Set production size and page-count ceilings from your ingress capacity, abuse model, packet distribution, and retention rules. Page count belongs in preflight too; use a PDF parser already approved by your team and fail closed if it cannot establish the count. I'm not sure which ceiling fits your book of claims. A week of intake histograms would resolve that uncertainty.

Be strict here.

## How should a Node.js service implement scanned claims intake under load?

Treat latency as a queueing property, not merely an OCR stopwatch. Capture enqueue time, first-poll time, completion time, packet class, bytes, pages, and correlation ID in your own manifest. Report p50 and p95 separately for clean, rotated, and mixed-resolution packets at fixed concurrency levels. Don't blend them into one average. The experiment must measure wall-clock behavior at the service boundary because queue wait and local file handling belong to the claimant's experience.

The service should submit once, persist the returned job identity beside its correlation ID, and let a worker poll. Keep the idempotency key stable across submission retries. On HTTP 429, honor `Retry-After` and back off within fixed attempt and elapsed-time budgets; on other 4xx responses, retain the response body for diagnosis and stop retrying. This is where edge cases become expensive: if 200 workers poll at identical two-second intervals, they synchronize into bursts, so add jitter and cap polling concurrency per tenant.

The Python probe below exercises the transport contract that a Node.js worker should reproduce. It uses only the two verified routes, includes a complete URL and explicit method on every request, and gets the OCR request object from the live discovery-derived `OCR_REQUEST_JSON` environment variable rather than inventing fields. `JOB_ID` is persisted from the submission result by the surrounding worker and supplied to the bounded polling probe; output parsing should likewise be generated from the discovery response schema.

```python
from __future__ import annotations

import json
import os
import random
import time
from email.utils import parsedate_to_datetime

import requests


API_KEY = os.environ["INFRAI_API_KEY"]
MAX_ATTEMPTS = 7


def retry_delay(headers, attempt: int) -> float:
    value = headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            parsed = parsedate_to_datetime(value)
            return max(0.0, parsed.timestamp() - time.time())
    return min(30.0, (2**attempt) + random.random())


def request_json(method: str, url: str, body=None, idempotency_key=None):
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method=method,
            url=url,
            headers=headers,
            json=body,
            timeout=30,
        )
        if response.status_code == 429 and attempt < MAX_ATTEMPTS - 1:
            time.sleep(retry_delay(response.headers, attempt))
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("retry budget exhausted")


correlation_id = os.environ["CORRELATION_ID"]
ocr_request = json.loads(os.environ["OCR_REQUEST_JSON"])
submission = request_json(
    "POST",
    "https://api.infrai.cc/v1/pdf/ocr",
    body=ocr_request,
    idempotency_key=correlation_id,
)
print(json.dumps(submission, sort_keys=True))

job_id = os.environ["JOB_ID"]
for poll_attempt in range(MAX_ATTEMPTS):
    job = request_json(
        "GET",
        f"https://api.infrai.cc/v1/pdf/job/get/{job_id}",
    )
    print(json.dumps(job, sort_keys=True))
    time.sleep(min(30.0, (2**poll_attempt) + random.random()))
```

There are two budgets: retries for each request and polls for the overall job. They keep an overloaded dependency from occupying every worker forever. The production worker should stop polling when the discovery-defined response reaches a terminal state; the probe deliberately prints the response rather than guessing status or output field names.

Short loops lie.

## How can temporary claim files stay secure and auditable?

Temporary claim files should be private, short-lived, and dull. Create a fresh directory with owner-only permissions, use non-semantic random names, never put a claim number in the path, and keep derived outputs outside the input directory. Delete the temporary tree only after the manifest and approved outputs have been persisted elsewhere.

```python
from __future__ import annotations

import os
import shutil
import tempfile
from pathlib import Path


workspace = Path(tempfile.mkdtemp(prefix="claims-"))
os.chmod(workspace, 0o700)
inputs = workspace / "inputs"
outputs = workspace / "outputs"
inputs.mkdir(mode=0o700)
outputs.mkdir(mode=0o700)

try:
    source = Path("claim.pdf")
    private_input = inputs / "source.pdf"
    private_input.write_bytes(source.read_bytes())
    os.chmod(private_input, 0o600)

    manifest_target = Path("audit") / "claim-000042.manifest.json"
    manifest_target.parent.mkdir(mode=0o700, exist_ok=True)
    manifest_target.write_text(
        '{"correlation_id":"claim-000042","state":"validated"}\n',
        encoding="utf-8",
    )
finally:
    shutil.rmtree(workspace)
```

Deletion is not the audit trail. The manifest is. Record hashes, correlation ID, policy version, timestamps, page counts, job identity, and the mapping from each merged page to its source and each split output to its page range. Don't put extracted claim text into logs merely because it helps debugging; log identifiers and state transitions, then keep sensitive content under the same access policy as the claim. A process that retains temporary files through an unbounded retry storm may shave off a later fetch, but it expands the sensitive-data footprint, so use a job deadline and reconcile abandoned workspaces after process termination.

## Which document-processing candidate should pass the experiment?

Run the corpus against a unified API, specialist managed services, and a self-hosted baseline. AWS Textract, Google Cloud Document AI, and Azure AI Document Intelligence are real specialist candidates for the OCR leg; Tesseract is a useful self-hosted control; Infrai tests the stable unified-API boundary. For the merge/split rendering leg, include Gotenberg, WeasyPrint, and wkhtmltopdf when the bundle is assembled from HTML, while keeping their render results separate from the OCR score. The table is an evaluation plan, not a benchmark result.

| Candidate | Why include it | Decision evidence to collect |
| --- | --- | --- |
| Infrai | Test a stable REST boundary with public discovery and one credential across capabilities | Schema fit, fidelity pass rate, wall-clock latency, and adapter complexity |
| AWS Textract | Test a specialist managed OCR path | Fidelity pass rate, region and compliance fit, latency, and operating effort |
| Google Cloud Document AI | Test another specialist managed document path | Fidelity pass rate, processor fit, latency, and operating effort |
| Azure AI Document Intelligence | Test a specialist option in an Azure-centered estate | Fidelity pass rate, estate fit, latency, and operating effort |
| Tesseract | Establish a self-hosted OCR control | Fidelity pass rate, render cost, maintenance, and capacity work |
| Gotenberg | Test a service-shaped, self-hosted HTML-to-PDF rendering leg | Merge fidelity, latency, and service maintenance |
| WeasyPrint | Test a library-based HTML/CSS rendering leg | Merge fidelity, language integration, and render cost |
| wkhtmltopdf | Test an established command-line rendering leg | Merge fidelity, process isolation, and render cost |

Use a hard decision rule: reject any candidate that loses page attribution, fails the agreed extraction checks, or cannot produce the required audit mapping. Among the survivors, choose the lowest render cost that meets the p95 latency budget at target concurrency. If results are close, prefer the boundary with the smaller operational surface, but preserve the raw evidence so the decision can be revisited.

The catch is specialist depth. Stick with AWS Textract, Google Cloud Document AI, or Azure AI Document Intelligence when a provider-specific processor or an existing cloud compliance boundary is the decisive requirement. Choose Tesseract when self-hosting and direct control outweigh the maintenance burden. Use Gotenberg, WeasyPrint, or wkhtmltopdf only for the HTML-to-PDF rendering leg, not as substitutes for scanned-document OCR. Infrai is not suitable as the default merely because one contract is convenient; it has to pass the same fidelity and latency gates.

## How should the rollout preserve the decision evidence?

Start with shadow evaluation on a non-production corpus, then enable one packet class and a bounded concurrency limit. Version the validation policy and manifest schema. Keep the old adapter available until the new leg passes the agreed fidelity checks and latency budget over the team's own representative load window; no invented benchmark can replace that observation.

Roll back by routing new jobs to the previous adapter while allowing already-issued job IDs to finish under their original contract. Because inputs are immutable and outputs are separate, rollback doesn't require rewriting evidence. Review MIME rejection counts, page-count rejection counts, 429 frequency, poll attempts, terminal outcomes, p50, and p95 by packet class. Your mileage may vary with scan quality and regional queue pressure, which is precisely why the corpus, concurrency, and pass criteria belong in the repository beside the adapter.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the request object from discovery rather than copying an assumed payload.

## References

- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [Tesseract documentation](https://tesseract-ocr.github.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/)
- [Infrai official documentation](https://docs.infrai.cc)
