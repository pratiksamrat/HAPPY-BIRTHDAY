# OfficeBox — Observability Assessment

**Scope:** `in-mvc-20` (ASP.NET MVC web app) and `In.OfficeBox.Api` (+ module APIs: Auth, Inventory, Accounts, CRM, HR, SAP, Dashboards) and supporting components (`OBTaskManager` Windows service).
**Date:** 2026-06-25
**Method:** Static review of configuration, startup/error pipelines, logging usage, background processing, integrations, and deployment.

---

## 1. Executive Summary

The product currently has **near-zero effective observability**. Telemetry libraries are present in the dependency manifests but are largely **unconfigured, dead, or unused**:

- **Logging** relies on log4net **1.2.10** (released ~2010, end-of-life), configured only in `in-mvc-20` to write **two flat files on the local IIS disk** (`C:\Logs\Summary\*.txt`). Backend APIs reference log4net but **do not configure it**, and application code makes **almost no logging calls**.
- **Application Insights 2.0.0** (2016, EOL) is referenced in the main API and its HTTP module is registered, but there is **no `ApplicationInsights.config` and no instrumentation key** — so it collects nothing.
- Unhandled errors are **not persisted**: the MVC `Application_Error` handler stashes the exception in an in-memory `Application[...]` variable and returns; the API has **no global error handler at all**.
- The codebase contains **~7,000+ empty `catch {}` blocks** (mvc-20 ≈ 3,081; main API ≈ 2,268; module APIs ≈ 1,714), meaning the overwhelming majority of failures are silently swallowed.
- There is **no metrics system, no distributed tracing, no health-check endpoints, no dashboards, and no alerting** anywhere in the stack.

**Net effect:** Failures, latency, and degradation across web, APIs, background jobs, and integrations are effectively **undetectable and undiagnosable** in production today. This is the single highest operational risk in the assessment.

---

## 2. Current-State Inventory

| Capability | State | Evidence |
|---|---|---|
| Structured logging | ✗ Effectively absent | log4net configured only in `in-mvc-20/Web.config` (2 file appenders); APIs reference the package but have no `<log4net>` section |
| Centralized log aggregation | ✗ None | Logs written to local disk `C:\Logs\Summary\Summary.txt` / `DataAccess.log` on each IIS node |
| Application logging calls in code | ✗ Negligible | Only ~8 files in mvc-20 reference a logger/EventLog; no `TrackException/TrackEvent/TrackMetric` calls in app code |
| Unhandled-exception capture (web) | ✗ Not persisted | `in-mvc-20/Global.asax.cs` `Application_Error` writes to in-memory `Application["ErrorInfo"]`; EventLog write commented out |
| Unhandled-exception capture (API) | ✗ Missing | `In.OfficeBox.Api/Global.asax.cs` has no `Application_Error`; `WebApiConfig` registers only `OBAuthFilter`, no exception filter/logger |
| APM / metrics | ✗ Not collecting | App Insights 2.0.0 referenced, HTTP module registered, **no instrumentation key / no `ApplicationInsights.config`**; no Prometheus/Datadog/NewRelic/StatsD |
| Distributed tracing | ✗ None | mvc-20 calls 7 backend services via appSettings URLs (`auth/inv/acnt/crm/hr/sap/dashboards`) with **no correlation/trace-id propagation** |
| Health checks | ✗ None | No `/health` endpoints; deployment relies on `stopsite.bat`/`startsite.bat` only |
| Background-job monitoring | ✗ None | `OBTaskManager` service logs to local `OBTaskService.log.txt`; no heartbeat/alert. `TimerTasks.cs` loop body is **empty** (dead infra) with an empty catch |
| Integration monitoring | ✗ None | SAP/Tally import, SMTP email, SMS, DB-Sync have no success/failure metrics or alerting |
| Dashboards | ✗ None | No Grafana/Kibana/Splunk/AI dashboards in use |
| Alerting | ✗ None | No alert rules anywhere |
| Existing partial telemetry | ~ Minimal | DB-based user-session log (`updateUserSessionStartLog`) and feature-usage logging exist (business analytics, not operational) |
| Deployment/infra signal | ~ Limited | AWS CodeDeploy → IIS/Windows; no monitoring hooks in `appspec.yml` |

---

## 3. Identified Gaps, Impact & Priority

### HIGH (operational blind spots — fix first)

**H1. Unhandled exceptions are not persisted anywhere**
*Evidence:* MVC `Application_Error` writes to in-memory state only; API has no error handler.
*Impact:* Production 500s leave no trace. Incidents are discovered via user complaints, with no stack trace to diagnose. MTTR is unbounded.

**H2. ~7,000+ empty `catch {}` blocks swallow failures silently**
*Evidence:* mvc-20 ≈ 3,081, main API ≈ 2,268, module APIs ≈ 1,714.
*Impact:* Data-integrity bugs, failed saves, and integration errors fail silently and surface much later as corrupt/missing data. Root cause is usually unrecoverable from logs.

**H3. Application Insights is referenced but collects nothing**
*Evidence:* No instrumentation key, no `ApplicationInsights.config` in the API; AI 2.0.0 is EOL.
*Impact:* The team believes APM exists, but there is **zero** request/dependency/exception telemetry. False sense of coverage.

**H4. No request/latency/error metrics for APIs**
*Evidence:* No metrics pipeline; AI inactive.
*Impact:* Cannot detect latency regressions, error-rate spikes, or throughput drops on any of the 7 backend services. No SLOs possible.

**H5. Logs are local-disk flat files, not centralized**
*Evidence:* `C:\Logs\Summary\*.txt` rolling files per node.
*Impact:* Cross-service correlation requires manual file access on each server; logs lost on instance replacement (CodeDeploy redeploys). Not searchable.

### MEDIUM

**M1. No distributed tracing across mvc-20 → backend APIs**
*Impact:* A single user action fans out to multiple services; without a correlation id, failures/latency cannot be traced end-to-end.

**M2. Background processing is unmonitored (and partially dead)**
*Evidence:* `OBTaskManager` logs only to a local text file with no heartbeat; `TimerTasks.cs` execution loop is empty.
*Impact:* If the stale-session-removal service stops, sessions accumulate silently. Confirm whether `TimerTasks` is intended to run; it currently does nothing.

**M3. Integrations (SAP/Tally import, email, SMS, DB-Sync) have no success/failure telemetry**
*Impact:* Failed imports or undelivered notifications go unnoticed until users report missing data.

**M4. No health-check endpoints**
*Impact:* Load balancer / CodeDeploy cannot verify app + dependency (DB, downstream API) health; bad deploys can go live.

**M5. EOL telemetry libraries**
*Evidence:* log4net 1.2.10, App Insights 2.0.0.
*Impact:* No security patches; missing modern features (structured/JSON logs, sampling, async sinks).

### LOW

**L1. Inconsistent log levels / format** — root logger at `DEBUG` in production is noisy and costly once centralized.
**L2. No business/operational dashboards** — existing session/feature-usage data is not surfaced.
**L3. No log retention/rotation policy beyond size-based rolling (10×10MB).**

---

## 4. Recommendations

### Phase 0 — Stop the bleeding (1–2 sprints, HIGH)
1. **Add a global exception sink in both apps.**
   - API: implement an `IExceptionLogger`/exception filter registered in `WebApiConfig` that logs full exception + request context.
   - MVC: in `Application_Error`, actually persist the exception (file + central sink) before clearing.
2. **Stand up centralized logging.** Migrate to a maintained logger (Serilog) writing to a central sink (Seq/ELK/Application Insights/CloudWatch). Keep a local rolling-file fallback.
3. **Triage empty catch blocks.** Add a Roslyn analyzer/CI rule to flag new empty catches; bulk-replace existing `catch {}` with `catch (Exception ex) { Log.Error(...); }` starting with controllers, services, and integration code.
4. **Either configure App Insights properly (key + config + `TelemetryClient`) or remove it** to eliminate the false sense of coverage. If kept, upgrade off 2.0.0.

### Phase 1 — Make services measurable (MEDIUM)
5. **Add `/health` endpoints** to each API and the MVC app (self + DB + downstream-API checks); wire into the load balancer and CodeDeploy validation hook.
6. **Emit request metrics** (count, latency p50/p95/p99, error rate) per controller/action via App Insights or a metrics exporter.
7. **Propagate a correlation id** from mvc-20 into every downstream API call (header + log enrichment) to enable end-to-end tracing.
8. **Instrument background jobs & integrations** — `OBTaskManager`, SAP/Tally import, email, SMS, DB-Sync: log start/finish/duration/row-counts and emit a heartbeat + failure metric. Decide the fate of dead `TimerTasks`.

### Phase 2 — Dashboards & alerting (MEDIUM/LOW)
9. **Dashboards:** API health (error rate, latency, throughput per service); background-job heartbeats & last-run status; integration success/failure; top exceptions; infra (CPU/mem/IIS queue).
10. **Alerts:** API error-rate > threshold; p95 latency breach; any unhandled exception burst; background service heartbeat missed; integration failure; health-check failing.
11. **Right-size log levels** (INFO in prod, DEBUG via config switch) and define retention.

---

## 5. Proposed Dashboards & Alerts (starter set)

| Dashboard | Key panels | Source |
|---|---|---|
| API Health | Error rate %, p50/p95/p99 latency, RPS — per service | request metrics |
| Exceptions | Top exceptions, count trend, by endpoint | exception sink |
| Background Jobs | Last run, duration, success/fail, heartbeat age | job instrumentation |
| Integrations | SAP/Tally import rows & failures, email/SMS delivery | integration logs |
| Infrastructure | CPU/mem, IIS request queue, app pool recycles | server agent |

| Alert | Condition (starting point) | Priority |
|---|---|---|
| API error spike | 5xx rate > 2% over 5 min | High |
| Latency breach | p95 > 2s over 10 min | High |
| Unhandled exception burst | > N exceptions/min | High |
| Job heartbeat missed | no heartbeat > 5 min | High |
| Integration failure | any import/notification failure | Medium |
| Health check failing | /health non-200 | High |

---

## 6. Actionable Backlog (suggested stories)

1. **[H]** Add global exception logging filter to all Web API projects.
2. **[H]** Persist unhandled exceptions in `in-mvc-20` `Application_Error`.
3. **[H]** Introduce Serilog + central sink; replace log4net file-only config.
4. **[H]** CI analyzer to block new empty catch blocks; remediation plan for existing ~7,000.
5. **[H]** Configure or remove Application Insights (key + config); upgrade if kept.
6. **[M]** Add `/health` endpoints (self + DB + downstream) to all services + LB wiring.
7. **[M]** Add per-endpoint request/latency/error metrics.
8. **[M]** Correlation-id propagation mvc-20 → backend APIs with log enrichment.
9. **[M]** Instrument `OBTaskManager` + decide fate of dead `TimerTasks`; add heartbeat + alert.
10. **[M]** Instrument SAP/Tally import, email, SMS, DB-Sync with success/failure telemetry.
11. **[M]** Build API Health, Exceptions, Background Jobs, Integrations dashboards.
12. **[M/L]** Configure alert rules per §5.
13. **[L]** Standardize log levels/format; define retention/rotation policy.

---

## 7. Acceptance Criteria Mapping

- ✅ All critical components assessed: web app, 7 backend APIs, background service, integrations, deployment.
- ✅ Gaps documented with supporting evidence (file references in §2–3).
- ✅ Recommendations categorized High/Medium/Low (§3–4).
- ⏳ Review with engineering + product stakeholders — to be scheduled.
- ⏳ Follow-up implementation stories — drafted in §6; create in the tracker.
