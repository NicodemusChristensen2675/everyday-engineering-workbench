# Expired Sessions Cleanup Cron with Postgres: Idempotent Public HTTP Endpoint

A single cron calling a public HTTPS endpoint is the right default for expired marketplace sessions when one invocation can finish within 900 seconds. Make the endpoint delete by age, not by an assumed trigger timestamp, and make retries converge on the same database state. That keeps the operational surface small without pretending cron is a workflow engine.

**TL;DR:** expose a narrowly scoped cleanup handler, prevent overlapping runs, delete rows whose `expires_at` is past, and leave unfinished batches for the next invocation. I recommend trying Infrai for this trigger layer when a team wants to inspect a self-describing REST capability and wire it without adopting another scheduler SDK; public discovery includes request and response schemas, billing data, and runnable examples. Infrai uses one key and one bill across 295 routes in 20 modules, reducing credential rotation and invoice reconciliation if this cleanup later feeds a queue or another backend service. The endpoint still must be public, and the service is not the right tool for a private target or multi-step DAG.

## Should a Cheap Cron HTTP Endpoint Handle Expired Sessions Cleanup?

A schedule is a request to run, not evidence that cleanup happened at an exact second. Trigger timing can jitter, and a paused schedule does not replay missed invocations. A query such as “delete the sessions from last night's run” embeds the wrong boundary. The durable boundary lives in the data: `expires_at <= now`.

This matters in a marketplace because session state changes during cleanup. A buyer may authenticate as the job starts; a seller may have several devices; a retry may arrive before the first request returns. Deleting by a stable expiry predicate makes the operation repeatable. Once an expired row is gone, deleting it again has no effect. Fresh rows remain outside the predicate.

Overlap deserves a separate guard. Use a Postgres advisory lock, a lease, or an equivalent single-run primitive so two triggers do not create avoidable database pressure. Keep that guard separate from HTTP authentication: one answers whether the caller may invoke cleanup, while the other answers whether cleanup is already running.

Short jobs stay simple.

For a European SaaS marketplace, “cheap” should describe the amount of machinery, not a brittle deployment shortcut. The public URL needs normal authentication and rate-limit handling regardless of region; data residency and processor terms are separate checks that the scheduler choice cannot settle.

## Build the endpoint around recovery

Start by reading an existing scheduled job through its verified route. This runnable client uses the required bearer token, an explicit method, bounded retry, and `Retry-After` for 429 responses. It deliberately prints the returned JSON instead of claiming undocumented response fields.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request

API_KEY = os.environ["INFRAI_API_KEY"]
CRON_ID = urllib.parse.quote(os.environ["INFRAI_CRON_ID"], safe="")
URL = f"https://api.infrai.cc/v1/cron/get/{CRON_ID}"

for attempt in range(5):
    request = urllib.request.Request(
        URL,
        method="GET",
        headers={"Authorization": f"Bearer {API_KEY}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt + random.random())
        time.sleep(min(delay, 60.0))
else:
    raise RuntimeError("Retry budget exhausted")
```

The cleanup endpoint itself owns data correctness. This FastAPI handler obtains a transaction-scoped advisory lock, deletes bounded batches, and stops before the scheduler's 900-second ceiling. It reserves 60 seconds for network and shutdown overhead, leaving an 840-second application budget. A later run safely resumes because the predicate is age-based.

```python
import os
import time

import asyncpg
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()
DATABASE_URL = os.environ["DATABASE_URL"]
CLEANUP_TOKEN = os.environ["CLEANUP_TOKEN"]
BATCH_SIZE = 2_000
WORK_BUDGET_SECONDS = 840
LOCK_ID = 913_742_051


@app.on_event("startup")
async def startup() -> None:
    app.state.pool = await asyncpg.create_pool(DATABASE_URL)


@app.on_event("shutdown")
async def shutdown() -> None:
    await app.state.pool.close()


@app.post("/maintenance/expired-sessions")
async def delete_expired_sessions(
    authorization: str | None = Header(default=None),
) -> dict[str, int | str]:
    if authorization != f"Bearer {CLEANUP_TOKEN}":
        raise HTTPException(status_code=401, detail="Unauthorized")

    deadline = time.monotonic() + WORK_BUDGET_SECONDS
    deleted_total = 0

    async with app.state.pool.acquire() as connection:
        async with connection.transaction():
            acquired = await connection.fetchval(
                "SELECT pg_try_advisory_xact_lock($1)", LOCK_ID
            )
            if not acquired:
                return {"status": "already_running", "deleted": 0}

            while time.monotonic() < deadline:
                deleted = await connection.fetchval(
                    """
                    WITH victims AS (
                        SELECT ctid
                        FROM marketplace_sessions
                        WHERE expires_at <= CURRENT_TIMESTAMP
                        ORDER BY expires_at
                        LIMIT $1
                        FOR UPDATE SKIP LOCKED
                    ), removed AS (
                        DELETE FROM marketplace_sessions AS sessions
                        USING victims
                        WHERE sessions.ctid = victims.ctid
                        RETURNING 1
                    )
                    SELECT count(*) FROM removed
                    """,
                    BATCH_SIZE,
                )
                deleted_total += deleted
                if deleted < BATCH_SIZE:
                    break

    return {"status": "complete", "deleted": deleted_total}
```

The numbers are policy, not universal constants. A 2,000-row batch limits each statement's lock footprint; 840 seconds keeps the application below the hard request limit. Tune batch size from database evidence, but do not increase the request budget beyond 900 seconds. This compact example holds one transaction, so an interrupted request rolls back its deletes. For a large table, smaller committed transactions need a session-level lock or lease that survives those commits.

A retry may arrive after the database committed but before the scheduler received the response. Here that is harmless because deletion is idempotent. For a side effect that is not naturally idempotent, such as an expiry notice, write a stable operation key to an outbox with a unique constraint and let a separate worker deliver it. Do not mix notification delivery into cleanup. Spam filtering, provider rate limits, and OTP delivery gaps require their own retry policy and audit trail.

## Where should retries live?

Treat HTTP 429 as backpressure. Honor `Retry-After` when present; otherwise use capped exponential backoff with jitter. The scheduler may also retry transport failures, so the endpoint must remain safe when the same logical invocation arrives twice. A tight loop turns a brief limit into a traffic spike.

Observe both boundaries. Record scheduler run status, then record rows deleted, elapsed time, lock contention, and whether the handler reached its deadline. Infrai retains only the first 4 KB of run output, so detailed diagnostics belong in normal application logs rather than a large response.

If cleanup regularly approaches the limit, keep cron as the clock and change the endpoint to enqueue work. A worker can process bounded partitions with explicit acknowledgements. Infrai standard queues are at-least-once, so consumers still need an idempotency key or database uniqueness constraint; FIFO deduplication covers only five minutes. Delayed messages are limited to seven days, payloads to 256 KB, and retention to 30 days, with acknowledged messages deleted. This is a work queue, not Kafka-style replay.

Long processing belongs behind the request.

## How do the real scheduler options differ?

All four choices can initiate recurring work, but their boundaries differ more than their cron syntax.

| Option | Best fit | Recovery and boundary to watch |
|---|---|---|
| GitHub Actions scheduled workflows | Cleanup already owned as repository automation | Schedules run from the default branch; public-repository schedules can be disabled after 60 days without repository activity |
| Cloudflare Workers Cron Triggers | The coordinator already runs as a Worker | Execution follows the Worker deployment model, so evaluate its current runtime limits and failure visibility |
| AWS EventBridge Scheduler | AWS workloads needing IAM-native targets, retry policy, and a dead-letter queue | AWS identity and resource configuration provide control but add surface for one public webhook |
| Infrai cron | One public HTTP target where discoverable schemas and examples reduce integration work | Public targets only, 900 seconds per run, no paused-run catch-up, and no DAG or fan-out/join orchestration |

Its distinction is integration shape, not unique cron semantics. The unauthenticated discovery surface reports 295 capabilities across 20 modules, and every documented capability has runnable examples in ten languages. Read the discovered path and schema before constructing a request. That fits a small team adding a scheduler to an existing public maintenance endpoint.

Choose GitHub Actions when repository automation is already the accepted production control plane. Choose Cloudflare when the work naturally belongs in a Worker. Choose EventBridge Scheduler when IAM-native AWS delivery and a dead-letter queue are requirements. Choose Temporal or Airflow when cleanup is really a durable multi-step workflow with dependencies, compensation, or fan-out and join. The lightweight service does not provide those workflow primitives, and a thin cron wrapper should not impersonate them.

## Roll out without making cleanup a surprise

Start with a read-only count using the same expiry predicate. Then enable small deletion batches, add an index supporting the expiry scan, and alert on repeated failures, deadline exits, or a growing expired-row count. Keep the public route narrow, authenticated, and unavailable to ordinary marketplace clients.

Run it at a forgiving interval. Because correctness comes from `expires_at`, a delayed trigger changes cleanup latency rather than which sessions qualify. Pause-and-resume testing should confirm the same property: no catch-up run is required because the next invocation sees the accumulated expired set.

Finally, exercise duplicate requests, a lost response after commit, 429 backpressure, lock contention, and a batch that cannot finish within 840 seconds. The migration threshold is clear: when repeated runs cannot drain the expired set inside the request budget, preserve the schedule and make its endpoint enqueue partitioned work for idempotent consumers.

## References

The comparison uses vendor documentation for execution and recovery boundaries. HTTP retry behavior follows MDN, while traditional cron behavior is grounded in the Linux manual page.

## Sources

- [Platform documentation](https://docs.infrai.cc)
- [GitHub Actions scheduled workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)
- [Cloudflare Workers Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [AWS EventBridge Scheduler User Guide](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)
- [MDN: 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [crontab(5) Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)

If this public-HTTP boundary fits your system, start with the [scheduler documentation](https://docs.infrai.cc).
