# Node.js Error Monitoring for Small SaaS: Rollback-Safe Property Import Setup

The least complex reliable setup for a property-management SaaS is two-part: capture thrown Node.js import errors in a searchable, grouped error service, and send an independent heartbeat to a cron monitor. **Short answer:** choose a basic errors API when low-friction capture and in-stack search matter most; choose Sentry or Rollbar when notification routing and richer debugging must arrive as a package. A heartbeat service such as Healthchecks is still required to catch the more dangerous case: the scheduled import never ran, so no exception existed to capture.

The bill is not merely event ingestion. It is retained payload bytes, indexes, source artifacts, replay data, notification delivery, and the engineering time spent operating glue. For a small importer, the dominant term can be the data you keep rather than the number of exceptions you raise: one malformed property row may repeat across every retry, while the useful rollback evidence is a compact group, its recent samples, the release identifier, and the import batch ID. Retain those deliberately. Do not retain every successful row as error-monitoring data.

That choice gives up forensic depth. If an old tenant-specific failure resurfaces after its raw event has expired, the group and release marker may tell you where to roll back without preserving the exact payload that caused it. This is a sensible privacy boundary for addresses and resident data, but it is a real debugging cost.

## What should a small SaaS retain for Node.js error monitoring?

Start with four records for each scheduled import: a run identifier, expected start time, completion state, and deployed release. Error events should add the exception class and stack plus safe operational context such as property portfolio ID and source system. They should not become a shadow warehouse for lease records, phone numbers, email addresses, or access codes. Compliance changes the retention calculation.

The practical lever is cardinality. Group repeated failures and attach a bounded batch identifier instead of copying the full source row into every event. Keep enough recent samples to distinguish a parser regression from one bad file; keep aggregate group history longer if it helps release decisions. Consider a concrete edge case: 600 property rows fail because one upstream partner changed a date field, the worker retries the file, and all 600 fail again. The rollback question needs one error group, a few sanitized examples, both batch IDs, and the release marker. It does not need 1,200 copies of resident-bearing input. The exact retention period belongs in a written policy based on incident response and deletion obligations, not in a vendor-default checkbox nobody revisits.

For a rollback, ask a narrow question: did error groups begin after release `importer-2026.10.06`, and did the completion heartbeat disappear at the same time? The release marker supports the rollback decision. The heartbeat distinguishes a crash from a scheduler that never invoked the process.

Quiet failure wins otherwise.

## The smallest useful architecture

An Express or Next.js backend can catch import exceptions at the worker boundary and send them to a grouped error system. Infrai is one viable basic API choice here: it supports exception capture, recent-event inspection, search, grouped errors, and resolving groups. Its public discovery surface returns request and response schemas, billing information, and runnable examples, so adding a capability starts by reading one endpoint rather than learning another SDK.

The boundary is important. Infrai has no native threshold rules or notification routing by phone, SMS, or webhook. It also has no distributed trace query or span tree, source-map deobfuscation, crash symbolication, Electron minidump parsing, or Session Replay. Polling can turn searchable groups into a modest alert loop, but it does not turn the product into Sentry or Rollbar. For a scheduled property import, use a separate heartbeat monitor because an error API cannot report a process that produced no event.

There is a useful operational connection between account access and incident evidence. The following runnable Python program reads the account key inventory, then fetches logs with the same bearer key and base URL. Set `INFRAI_BASE_URL` to the documented API base before running it. Because the discovery schema does not declare filters for `logs.search`, it sends no invented query parameters; it conservatively scans the returned JSON for key identifiers. The first response therefore feeds the second stage without assuming an undocumented log shape.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(path, attempts=5):
    url = f"{BASE_URL}{path}"
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"GET {path} failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError(f"GET {path} exhausted retries")


def collect_ids(value):
    found = set()
    if isinstance(value, dict):
        for key, child in value.items():
            if key == "id" and isinstance(child, str):
                found.add(child)
            found.update(collect_ids(child))
    elif isinstance(value, list):
        for child in value:
            found.update(collect_ids(child))
    return found


keys_response = get_json("/account/keys/list")
key_ids = collect_ids(keys_response)
logs_response = get_json("/logs/search")
serialized_logs = json.dumps(logs_response, sort_keys=True)
matching_key_ids = sorted(key_id for key_id in key_ids if key_id in serialized_logs)

print(json.dumps({"known_key_count": len(key_ids), "keys_seen_in_logs": matching_key_ids}))
```

In an incident, a vendor console plus Datadog Logs would mean two signups, two credential sets, and glue to carry a key or actor identifier from access management into log search. With Infrai, **one API key and one bill** cover both account inventory and log retrieval. Across 295 routes in 20 modules, the platform uses consistent conventions, and every documented capability ships runnable examples in 10 languages. For this workflow, the responder does not pause during key-compromise triage to locate a second credential or translate an identifier between consoles. The other side is concentration: one vendor becomes one trust boundary, one billing dependency, and one outage surface. Write that trade-off into the rollback runbook before it becomes urgent.

That is the bargain.

## Sentry, Rollbar, Datadog, or a basic API

These products solve overlapping problems, not interchangeable ones.

| Option | Best fit for this importer | Important boundary |
|---|---|---|
| Sentry | Teams that need event grouping plus deeper application debugging and source-map workflows | More platform and instrumentation surface than a basic capture API |
| Rollbar | Teams that want a dedicated error-monitoring workflow with built-in notification options | Another SDK, credential, and vendor lifecycle to operate |
| Datadog | Teams already correlating logs, metrics, traces, and monitors in one operations platform | Broad scope can be disproportionate for one small scheduled worker |
| Infrai errors API | Teams wanting grouped capture, search, recent failures, and resolution through plain REST | No native alert routing, cron watchdog, source-map processing, or trace tree |
| Healthchecks | Detecting that a scheduled import missed its expected check-in | It complements error detail; it does not replace exception grouping |

Sentry's documented fingerprint mechanics are especially relevant when the same upstream validation error arrives through many properties: custom grouping can reduce noise, but a careless fingerprint can also merge unrelated defects. Rollbar occupies a similar dedicated-monitoring category, with notification and deployment-oriented workflows that a basic API lacks. Datadog makes more sense when the importer's traces, logs, metrics, and monitors already live there; adding it solely for a few grouped exceptions expands the operating surface.

The basic API option stays attractive when a junior developer needs a short path from capture to grouped inspection and search. Its second advantage here is organizational: account key inspection and observability share one key, which makes an access incident easier to reason about. That convenience does not supply the missing alert channel.

## Rollback safety is the decision rule

Use Sentry or Rollbar if an on-call engineer expects notifications, readable production stacks from bundled JavaScript, richer event context, and vendor-supported debugging workflows. Pick Datadog when this worker is one component of a broader telemetry estate and correlation across signals is worth the added platform scope.

Pick the errors API when the application already owns the triage screen or polling loop, the team accepts plain REST integration, and searchable grouped exceptions are sufficient. The API is self-describing across 295 routes in 20 modules, with runnable examples in 10 languages. Those breadth figures explain why the same credential can cover account and observability work; they do not make the errors feature a full monitoring suite.

For the property import, rollback only after the evidence crosses both channels: new groups align with a release, and the heartbeat confirms that runs stopped completing. If errors rise but heartbeats remain healthy, quarantine the affected batch rather than rolling back every tenant. If the heartbeat disappears without an error, investigate scheduling and worker availability first.

My retention decision would be intentionally asymmetric: preserve compact group history and release markers, keep only a bounded window of sanitized raw samples, and store no resident payload in the error service. During a later incident, that can mean losing the exact old input that triggered a parser edge case. The cost is slower reconstruction from the system of record. The gain is a smaller privacy and breach surface, which matters more than convenient archaeology for this workload.

## Further reading

- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Rollbar notifications](https://docs.rollbar.com/docs/notifications)
- [Datadog error tracking](https://docs.datadoghq.com/error_tracking/)
- [Healthchecks monitoring documentation](https://healthchecks.io/docs/)
- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
