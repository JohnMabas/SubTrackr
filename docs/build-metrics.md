# Build Metrics Exporter

`backend/services/shared/buildMetricsService.ts`

## Overview

`BuildMetricsService` records CI/CD build pipeline telemetry — run outcomes, stage durations, artifact sizes — and renders it in the Prometheus text exposition format so build health is scrapeable and alertable next to the runtime metrics the backend already publishes.

It solves three gaps:

- Build duration regressions (p95) were only visible in CI logs, never graphed
- Bundle-size budget checks in CI failed the run but left no queryable history
- Pipeline failure rates could not be compared per pipeline or per failure reason

## Architecture

```
beginBuild('ci')                       recordBuildRun({ pipeline, status, ... })
      │                                          │
      ▼                                          ▼
 ┌────────────────┐                     ┌──────────────────────┐
 │ in-flight map  │──── endBuild() ────▶│ per-pipeline state  │
 │ + in_progress  │                     │  counters, samples, │
 └────────────────┘                     │  stages, artifacts  │
                                         └──────────┬───────────┘
                                                    │
                            ┌───────────────────────┴───────────────────────┐
                            ▼                                               ▼
                 GET /metrics/build (text)                    GET /build/metrics (JSON)
```

## Endpoints

| Method | Path | Content type | Description |
|---|---|---|---|
| `GET` | `/metrics/build` | `text/plain; version=0.0.4` | Prometheus exposition text |
| `GET` | `/build/metrics` | `application/json` | Aggregated summary for dashboards |

Both endpoints bypass rate limiting and the IP allow-list gate, matching the other `/metrics/*` routes.

## Recording Builds

### From in-process work

`beginBuild` / `endBuild` measure the duration server-side, so callers never need a timer:

```typescript
import { buildMetricsService } from '../services/shared/buildMetricsService';

const build = buildMetricsService.beginBuild('backend-bootstrap', {
  runId: process.env.GITHUB_RUN_ID,
  commitSha: process.env.GITHUB_SHA,
  branch: process.env.GITHUB_REF_NAME,
});

await warmCaches();

buildMetricsService.endBuild(build, {
  status: 'success',
  stages: [{ stage: 'cache-warm', durationMs: Date.now() - build.startedAt, status: 'success' }],
});
```

`startServer` already does exactly this for the plan-cache bootstrap, which is why `subtrackr_build_duration_ms{pipeline="backend-bootstrap"}` tracks the < 2 s startup budget in `AGENTS.md`.

### From a CI report

`ingestBuildReport` accepts untrusted JSON and never throws — bad entries are reported by index so a partially corrupt report still yields metrics:

```typescript
const result = buildMetricsService.ingestBuildReport(JSON.parse(await readFile('build-report.json', 'utf8')));

// result.accepted → BuildRunRecord[]
// result.rejected → [{ index: 0, reason: 'build run requires a finite "durationMs"' }]
```

Accepted shapes: a single run object, an array of runs, or `{ runs: [...] }` (also `builds` / `workflow_runs`). Field aliases are tolerated so reports straight out of CI work unchanged:

| Canonical field | Accepted aliases | Type coercion |
|---|---|---|
| `durationMs` | `duration_ms` | numeric strings coerced (`"1200"` → `1200`) |
| `runId` | `run_id`, `id` | numbers stringified |
| `commitSha` | `commit_sha`, `sha` | — |
| `branch` | `ref` | — |
| `failureReason` | `failure_reason` | — |
| `startedAt` / `completedAt` | `started_at` / `completed_at` | epoch ms, numeric strings accepted |
| `stages[].durationMs` | `stages[].duration_ms` | numeric strings coerced |
| `artifacts[].sizeBytes` | `artifacts[].size_bytes` | numeric strings coerced |

A missing `stages[].status` falls back to the run status, so a failed run attributes its failure to every stage that has no explicit status.

## Artifact Budget

The budget is a per-artifact ceiling, mirroring `bundleSizeBytes` in `performance-budget.json`. When configured, artifacts over the budget increment a breach counter and produce a ratio gauge:

```typescript
buildMetricsService.setArtifactBudget(5 * 1024 * 1024); // 5 MB
```

`setArtifactBudget(null)` or a non-positive value disables enforcement; the budget gauge then reports `0` and the ratio family is omitted entirely.

Alert on it with:

```promql
subtrackr_build_artifact_budget_ratio > 1
```

## Exposed Metrics

| Metric | Type | Labels | Description |
|---|---|---|---|
| `subtrackr_build_runs_total` | counter | — | Total recorded runs |
| `subtrackr_build_runs_total` | counter | `pipeline`, `status` | Runs per pipeline and outcome (all three statuses always emitted) |
| `subtrackr_build_success_runs_total` | counter | — | Successful runs |
| `subtrackr_build_failure_runs_total` | counter | — | Failed runs |
| `subtrackr_build_cancelled_runs_total` | counter | — | Cancelled runs |
| `subtrackr_build_pipelines` | gauge | — | Pipelines with recorded runs |
| `subtrackr_build_runs_rejected_total` | counter | `reason` | Rejected records (`validation`, `malformedStage`, `malformedArtifact`) |
| `subtrackr_build_artifact_budget_bytes` | gauge | — | Configured budget, `0` when disabled |
| `subtrackr_build_duration_ms` | gauge | `pipeline` | Most recent run duration |
| `subtrackr_build_duration_p50_ms` | gauge | `pipeline` | Median duration over the sample window |
| `subtrackr_build_duration_p95_ms` | gauge | `pipeline` | p95 duration over the sample window |
| `subtrackr_build_duration_p99_ms` | gauge | `pipeline` | p99 duration over the sample window |
| `subtrackr_build_in_progress` | gauge | `pipeline` | Builds currently running |
| `subtrackr_build_last_success_timestamp_seconds` | gauge | `pipeline` | Unix time of the last success, `0` if none |
| `subtrackr_build_info` | gauge | `pipeline`, `commit`, `branch` | Static metadata, always `1` |
| `subtrackr_build_stage_duration_ms` | gauge | `pipeline`, `stage` | Most recent duration for a stage |
| `subtrackr_build_stage_failures_total` | counter | `pipeline`, `stage` | Failed stage runs |
| `subtrackr_build_artifact_size_bytes` | gauge | `pipeline`, `artifact` | Most recent artifact size |
| `subtrackr_build_artifact_budget_ratio` | gauge | `pipeline`, `artifact` | Size ÷ budget; `> 1` breaches |
| `subtrackr_build_budget_breaches_total` | counter | `pipeline`, `artifact` | Budget breaches |
| `subtrackr_build_failures_total` | counter | `pipeline`, `reason` | Failures bucketed by reason |

Sample scrape:

```
# HELP subtrackr_build_duration_p95_ms Build duration p95 over the last 100 runs
# TYPE subtrackr_build_duration_p95_ms gauge
subtrackr_build_duration_p95_ms{pipeline="ci"} 184320
```

The namespace is configurable: `buildMetricsService.prometheusMetrics('acme_ci')` emits `acme_ci_*`.

### Prometheus text format correctness

- `HELP` and `TYPE` are emitted exactly once per metric family, regardless of how many pipelines contribute samples
- Label values escape `\`, `"` and newline, so hostile branch or pipeline names cannot break a scrape
- Missing data is reported as `NaN` (a legal sample value) rather than a magic sentinel, so dashboards can tell "no data" from a real zero
- Fresh instances still emit the aggregate counters, so a scrape before the first build is a valid document

## JSON Summary

`GET /build/metrics` (and `getMetrics()`) returns:

```json
{
  "totalRuns": 12,
  "successRuns": 10,
  "failureRuns": 1,
  "cancelledRuns": 1,
  "successRatePct": 83.33,
  "inProgress": 1,
  "trackedPipelines": 2,
  "lastCompletedAt": 1767225600000,
  "lastSuccessAt": 1767225600000,
  "artifactBudgetBytes": 5242880,
  "budgetBreaches": 1,
  "rejectedRecords": { "validation": 0, "malformedStage": 0, "malformedArtifact": 0 },
  "pipelines": {
    "ci": {
      "totalRuns": 12,
      "successRatePct": 83.33,
      "inProgress": 1,
      "lastStatus": "success",
      "lastDurationMs": 181000,
      "maxDurationMs": 240000,
      "durationP50Ms": 176000,
      "durationP95Ms": 184320,
      "durationP99Ms": 231000,
      "failureReasons": { "lint": 1 },
      "stages": { "lint": { "runs": 12, "failures": 1, "totalDurationMs": 24000, "lastDurationMs": 2000, "avgDurationMs": 2000 } },
      "artifacts": { "bundle.js": { "lastSizeBytes": 5300000, "maxSizeBytes": 5300000, "breaches": 1 } }
    }
  }
}
```

`durationP50Ms` / `p95` / `p99` are `NaN` (serialised as `null` in JSON) for a pipeline with no completed runs.

## Alerting Examples

```promql
# Build failure rate over the last day
sum(rate(subtrackr_build_failure_runs_total[1h])) / sum(rate(subtrackr_build_runs_total[1h])) > 0.1

# p95 build duration regression
subtrackr_build_duration_p95_ms > 300000

# Bundle size over budget
subtrackr_build_artifact_budget_ratio > 1

# Pipeline stuck in flight
subtrackr_build_in_progress > 0 and time() - subtrackr_build_last_success_timestamp_seconds > 3600

# Export is rejecting bad CI reports
increase(subtrackr_build_runs_rejected_total{reason="validation"}[1h]) > 0
```

## Completion Sinks

Register callbacks for alerting integrations. A throwing sink is logged and swallowed so it can never break metric collection:

```typescript
const unsubscribe = buildMetricsService.onBuildComplete((record) => {
  if (record.status === 'failure') {
    void pagerDuty.trigger({ pipeline: record.pipeline, reason: record.failureReason });
  }
});

unsubscribe(); // stop delivering
```

## Bounded Memory

All growth is capped so a long-running process cannot leak:

| Setting | Default | Constructor option |
|---|---|---|
| Retained runs (history) | 200 | `maxHistory` |
| Duration samples per pipeline (percentile window) | 100 | `maxSamples` |
| Distinct failure reasons per pipeline | 20 (excess bucket into `other`) | `maxFailureReasons` |

`getHistory(limit)` returns the newest runs first. `reset()` clears runs, in-flight builds and rejection counters — useful between tests.

## Validation

`recordBuildRun` and `beginBuild` throw `BuildMetricsValidationError` for programmer errors and increment `rejectedRecords.validation`:

- missing or blank `pipeline`
- `status` outside `success | failure | cancelled`
- `durationMs` that is negative, `NaN` or infinite
- `endBuild` with a handle that was never issued or already ended

Malformed nested entries are dropped rather than fatal, and counted separately so a scrape still shows the damage:

| Counter | Cause |
|---|---|
| `validation` | The run itself was unusable |
| `malformedStage` | A stage had no name or an unusable duration |
| `malformedArtifact` | An artifact had no name or an unusable size |

## Tests

```bash
npx jest --config jest.backend.config.js backend/services/shared/__tests__/buildMetricsService.test.ts
```

Covers counter aggregation, percentile windows, stage/artifact budgets, in-flight tracking, handle reuse rejection, malformed record rejection, untrusted report ingestion, label escaping, non-finite rendering, sink error isolation, and the HTTP surface in `backend/__tests__/server.test.ts`.
