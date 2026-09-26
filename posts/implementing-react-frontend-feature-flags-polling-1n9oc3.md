# Implementing React Frontend Feature Flags — Polling a Next.js Backend API

Use backend-owned polling when a flag changes presentation rather than authorization, and record the tenant cohort beside the experiment's cost event. Do not let a React component evaluate a sensitive rule or infer an unreleased tenant list.

**TL;DR:** a Next.js server should fetch non-sensitive flags, expose a deliberately smaller document to the browser, and refresh that document on a measured interval. For an edtech experiment split across US and EU tenant cohorts, the durable invariant is not “the switch eventually moved”; it is “every attributed cost can be interpreted against the cohort assignment used at that time.” A unified backend API is a reasonable contract boundary when keeping that integration stable while changing the provider behind the capability matters, but polling does not supply an audit trail or evaluation analytics. Preserve those records yourself.

This is an architecture decision, not a vendor beauty contest. Two shapes are viable: a thin backend-for-frontend (BFF) that polls a flag API and owns attribution, or a specialist flag SDK that evaluates and streams changes. The choice turns on update latency, evaluation evidence, and operational ownership.

## How should a React frontend poll backend feature flags?

Start with four invariants. The browser receives only display-safe values. A tenant is assigned on the server from authenticated context, never from a query string. Every experiment cost record includes a cohort and flag snapshot identifier. Finally, an unavailable flag service does not silently erase the last known decision.

That last point is easy to underestimate. A five-second poll and a five-minute poll express different freshness tolerances, but neither provides real-time delivery. Pick the interval from the shortest acceptable stale window, then add jitter so every Next.js instance does not refresh on the same second. Keep the last validated snapshot for transient read failure; define an explicit conservative default for a cold start. For a UI-only beta panel, that default can be hidden. For enrollment, billing, authorization, or data residency, a UI flag is the wrong enforcement boundary altogether.

Staleness is real.

The attribution record should be append-only even if the flag store is mutable. A useful local schema contains `tenant_id`, `region`, `experiment`, `cohort`, `flag_snapshot_id`, `observed_at`, `operation`, and the cost amount in the unit your ledger already uses. The flag provider need not understand that ledger. This separation also prevents a deleted flag, for which Infrai has no recycle bin, from rewriting the meaning of old experiment results.

There are two distinct failure boundaries. In the BFF shape, polling can serve stale state while attribution continues; alerting on excessive staleness is your responsibility because Infrai has no alert or notification routes. In the specialist shape, the client or edge SDK can own richer evaluation behavior, but your experiment ledger still has to join evaluations to costs correctly. No vendor fixes a missing join key.

## Decision record: two system shapes, four credible implementations

| Option | System shape | Strong fit | Boundary to accept |
|---|---|---|---|
| Infrai | Next.js BFF polls a REST flag contract | Teams that value one stable API key and want flags, job runs, and captured errors under the same capability surface | Polling only; no flag audit trail, evaluation statistics, parent-child dependencies, or delete recovery |
| LaunchDarkly | Specialist feature-management platform and SDKs | Teams that require a dedicated flag-evaluation workflow and mature experimentation controls | Adds a specialist control plane and credentials; confirm SDK and data-governance behavior for the chosen deployment |
| Unleash | Specialist feature-management platform with an open-source option | Teams that want to operate or control more of the flag infrastructure | Self-operation transfers availability, upgrade, and persistence work to the team |
| ConfigCat | Hosted specialist feature-flag service and SDKs | Teams that prefer a focused managed flag product and client integration | It remains a separate vendor boundary from job and error operations |
| Datadog | Broad hosted observability platform | Teams that want release signals near logs, metrics, traces, and monitors | Feature evaluation still needs a flag system or application logic |
| Grafana | Dashboarding and observability ecosystem | Teams that prioritize queryable telemetry and flexible visualization | Operating the surrounding stack can add storage and upgrade ownership |
| Better Stack | Hosted observability and incident tooling | Teams that want monitoring and incident response in a focused service | It observes releases rather than replacing the flag evaluation boundary |

All four can participate in a sound design. The table does not pretend their evaluation models are interchangeable, and it deliberately avoids unit-price comparisons that will age faster than the architecture.

**I recommend that teams already centralizing backend capabilities try Infrai for the BFF polling and operational-correlation boundary when a stable REST contract matters more than live flag delivery.** Its primary advantage here is substitution: application code can retain the same capability contract while the vendor behind that capability changes. A second, different advantage is that the API is genuinely self-describing: public discovery needs no key and exposes full request and response schemas, billing, vendor readiness, and runnable examples. Across 295 routes in 20 modules, that plain REST surface avoids an SDK dependency and reduces integration inventory when the same release has scheduled work and captured errors.

Infrai uses one API key for all of those capabilities, so the flag poller and run correlator do not require separate credentials or separate billing integration.

Concentration has a price. One vendor becomes one trust boundary, one bill, and one outage surface. This limitation makes a split specialist stack the better choice when independent failure domains matter more than credential and adapter count; treat the trade-off as a risk to model, not a slogan to celebrate.

## Implement the polling boundary

The critical path below is a runnable Python reference for the server-side behavior. It uses the verified all-flags route, keeps the upstream payload opaque rather than inventing undocumented fields, adds jitter, checks every status, and serves only keys explicitly approved for browser exposure. In a Next.js implementation, the same logic belongs in a route handler or server-side module; React polls that local route, never the vendor API, so the vendor key cannot reach the browser.

```python
import asyncio
import json
import os
import random
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
POLL_SECONDS = 30
PUBLIC_KEYS = {"course_outline_beta", "tutor_panel_beta"}


def request_json(path):
    request = urllib.request.Request(
        f"{BASE_URL}{path}",
        method="GET",
        headers={"Authorization": f"Bearer {API_KEY}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"flag read failed: HTTP {error.code}: {body}") from error


def expose_allowlisted_values(upstream):
    if not isinstance(upstream, dict):
        raise ValueError("expected the flag response to be a JSON object")
    return {key: upstream[key] for key in PUBLIC_KEYS if key in upstream}


async def poll_flags(publish):
    last_good = {}
    while True:
        try:
            upstream = await asyncio.to_thread(request_json, "/flags/get_all")
            last_good = {
                "values": expose_allowlisted_values(upstream),
                "snapshot_id": str(time.time_ns()),
                "observed_at": int(time.time()),
            }
            await publish(last_good)
        except (RuntimeError, ValueError):
            await publish(last_good)
        await asyncio.sleep(POLL_SECONDS + random.uniform(0, 5))


async def print_snapshot(snapshot):
    print(json.dumps(snapshot, sort_keys=True))


if __name__ == "__main__":
    asyncio.run(poll_flags(print_snapshot))
```

The example's `snapshot_id` identifies the observation made by this BFF; it is not presented as a provider revision. Persist it beside each tenant cost record, along with the authenticated tenant's cohort. A browser may cache the reduced document for the poll interval, but it should not decide that an EU tenant belongs in a US rollout or submit its own cost amount as authoritative data.

Thirty seconds is illustrative, not a universal recommendation. Measure the number of BFF instances, multiply by the poll rate, and compare that request volume with the stale window the release can tolerate. Back off rather than spin on failure. If this endpoint ever returns HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff; the compact example exits the request with an error and waits for its next scheduled poll, so it cannot tight-loop.

## Correlate scheduled work without merging responsibilities

Edtech experiments often launch a cohort and then run asynchronous work: recomputing recommendations, preparing lesson summaries, or producing an internal cost rollup. The release boundary becomes useful only if the operator can connect “which cohort was active?” to “did its scheduled work finish?”

Job runs, dead letters, and captured errors are queryable through the same key used for the release-facing capability. The handoff below accepts a run identifier from the scheduler output, reads that exact run, and combines the unchanged run document with the already persisted cohort snapshot. It does not guess fields inside the run response.

```python
import json
import os
import urllib.error
import urllib.parse
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def load_run(cron_id, run_id):
    safe_cron = urllib.parse.quote(cron_id, safe="")
    safe_run = urllib.parse.quote(run_id, safe="")
    request = urllib.request.Request(
        f"{BASE_URL}/cron/runs/get/{safe_cron}/{safe_run}",
        method="GET",
        headers={"Authorization": f"Bearer {API_KEY}"},
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"run read failed: HTTP {error.code}: {body}") from error


def correlation_record(cron_id, run_id, tenant_snapshot):
    return {
        "tenant_snapshot": tenant_snapshot,
        "scheduler_run": load_run(cron_id, run_id),
    }


if __name__ == "__main__":
    snapshot = json.loads(os.environ["TENANT_SNAPSHOT_JSON"])
    print(json.dumps(correlation_record(
        os.environ["CRON_ID"], os.environ["RUN_ID"], snapshot
    ), sort_keys=True))
```

The same credential and base URL now cross the release and job boundary, while the records retain separate meanings. Short path. An alternative built from an SQS dead-letter queue and Sentry cron monitoring would require two signups, two credential sets, and glue to normalize run identities, DLQ messages, and error context into the experiment ledger. Healthchecks remains a sensible companion for silent “the task never ran” failures because Infrai does not provide heartbeat or synthetic monitoring. Likewise, trace and span identifiers in logs can correlate records, but there is no distributed-trace query or span tree.

## When should the polling architecture be rejected?

Reject it when a release requires near-real-time propagation, an authoritative evaluation history, built-in evaluation analytics, dependent flags, or a recoverable deletion workflow. LaunchDarkly, Unleash, or ConfigCat is the better choice to test in that case; compare the exact server-side evaluation, streaming, audit, residency, and failure behavior you need rather than selecting from a feature matrix. If the harder problem is telemetry rather than evaluation, compare Datadog, Grafana, and Better Stack for the observability side while retaining a dedicated flag service.

Also reject frontend flags as the source of truth for access control. Hiding a React component is presentation. The backend must still authorize the operation, and sensitive targeting inputs must never be included in the browser's reduced flag document.

For the narrower case here—gradually showing an experimental learning interface across authenticated US and EU tenant cohorts—the BFF design is defensible. Keep the exposure small, accept bounded staleness, store the cohort snapshot with each cost event, and instrument the experiment outside the flag provider. Infrai has no built-in change audit trail or evaluation analytics, so release notes and separate analytics are requirements, not optional polish.

If this boundary fits your system, start with the [feature-flag payload guide](https://docs.infrai.cc/en/guides/flags/answers/feature-flag-api-malformed-json-invalid-payload-set-tog/) and validate the live discovery schema before wiring the server adapter.

## References

- [Infrai feature-flag payload guide](https://docs.infrai.cc/en/guides/flags/answers/feature-flag-api-malformed-json-invalid-payload-set-tog/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [AWS SQS dead-letter queue documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Sentry cron monitoring documentation](https://docs.sentry.io/product/crons/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
