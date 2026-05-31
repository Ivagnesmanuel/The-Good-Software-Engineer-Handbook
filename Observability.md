# Observability

A quick-reference guide covering how to know what a running system is doing: the three pillars (metrics, logs, traces), the discipline of SLOs and error budgets, the methods (USE, RED) for reasoning about resources and services, alerting, dashboards, and SRE practice.

This is the operational, production-facing companion to `Performance Optimization.md`. Where that document covers how to *make code fast* (memory hierarchy, data layout, compilers), this one covers how to *see what production is doing* and operate it reliably. The two overlap at profiling and latency — handled here from the observation angle, with cross-references to the engineering treatment.

---

## Table of Contents

1. Observability vs Monitoring
2. The Three Pillars — Metrics, Logs, Traces
3. Metrics in Depth
4. Logs in Depth
5. Distributed Tracing
6. SLI / SLO / SLA / Error Budgets
7. The USE and RED Methods
8. Alerting
9. Dashboards and Visualization
10. Profiling in Production
11. Latency Tail and p99 Thinking
12. Load Testing
13. SRE Practice
14. Tools Reference

---

## 1. Observability vs Monitoring

These are often conflated. The distinction matters.

```
Monitoring:    the set of things to watch is predefined; alerts fire on known
               patterns. Best for known failure modes.

Observability: new questions can be asked of the system after the fact,
               without redeploying. Best for unknown failure modes.
```

A monitored system reports when a known metric crosses a known threshold. An observable system permits telemetry to be sliced by labels not anticipated in advance — "p99 for user 12345 in region eu-west, broken down by endpoint, over the last 6 hours."

Both are necessary. Monitoring catches the known unknowns; observability enables investigation of the unknown unknowns.

---

## 2. The Three Pillars — Metrics, Logs, Traces

### Metrics

Numbers over time. Aggregated. Cheap to store and query at scale.

```
Examples:
  http_requests_total{method="POST", path="/orders", status="200"}
  cpu_usage_percent
  db_connections_active
  payment_failures_total
```

Used for: dashboards, alerting, capacity planning, SLOs.

**Limit:** aggregation loses detail. You know "1% of requests are failing" but can't always tell *which* requests.

### Logs

Discrete events with details. High volume, structured ideally.

```
2025-01-15T10:30:42.123Z INFO  request_id=abc-123 user_id=42 path=/orders status=200 latency_ms=87
2025-01-15T10:30:42.500Z ERROR request_id=def-456 user_id=99 path=/payments error="card declined"
```

Used for: forensic investigation, audit trails, debugging specific requests.

**Limit:** expensive to store and query at high volume. Sampling helps but loses information.

### Traces

The path of a single request through the system, across services.

```
Request → API Gateway → Auth Service → Order Service → Payment Service
                                            ↓
                                         Database
                                            ↓
                                         Kafka publish

Each span: which service, what operation, how long, with context.
```

Used for: latency debugging, dependency mapping, finding bottlenecks across service boundaries.

**Limit:** high overhead to capture every span; usually sampled.

### How They Combine

```
Alert fires (metric):     "p99 latency on /orders > 1s"
                                |
                                v
Dashboard (metrics):       Which endpoints? Which time range? Correlated with what?
                                |
                                v
Traces (distributed):      Which sub-service is slow? Where in the call chain?
                                |
                                v
Logs (per service):        What was the actual error? What was the input?
```

Modern platforms (Datadog, Honeycomb, Grafana stack, New Relic) link them: from a metric spike, click through to traces; from a trace, click through to logs for that request.

---

## 3. Metrics in Depth

### Metric Types

**Counter:** monotonically increasing total. Reset only on restart.

```
http_requests_total{status="200"} = 12,453
```

Query as a rate: `rate(http_requests_total[5m])` → requests per second.

**Gauge:** instantaneous value that can go up or down.

```
cpu_temperature_celsius = 67
queue_depth = 1,243
memory_usage_bytes = 8e9
```

**Histogram:** distribution of values bucketed into ranges.

```
http_request_duration_seconds_bucket{le="0.005"} = 12000
http_request_duration_seconds_bucket{le="0.01"}  = 14000
http_request_duration_seconds_bucket{le="0.025"} = 14500
http_request_duration_seconds_bucket{le="0.1"}   = 14700
http_request_duration_seconds_bucket{le="+Inf"}  = 14750
http_request_duration_seconds_sum = 47.2
http_request_duration_seconds_count = 14750
```

Used for percentiles, averages, distributions. Buckets are pre-defined.

**Summary:** percentiles computed on the client. Cheaper to query but can't be aggregated across instances.

Histograms are the standard modern choice — server-side aggregation enables cross-instance percentiles.

### Cardinality

The number of unique label combinations for a metric.

```
http_requests_total{method, path, status, user_id, region, ...}
```

If `user_id` has millions of values, the result is millions of time series. **High-cardinality labels destroy time-series databases.**

Rules of thumb:

- Bounded labels (status code, method, region, env): fine.
- Per-user, per-request-ID, per-IP: usually bad. Log it, don't metric it.
- Hundreds of values per label: usually fine. Thousands: think carefully. Millions: never.

### Metric Sources

```
Application metrics       /metrics endpoint scraped by Prometheus
System metrics            node_exporter, cAdvisor, fluent-bit, etc.
Cloud metrics             CloudWatch, GCP Cloud Monitoring, Azure Monitor
Database metrics          Postgres pg_stat_statements, MySQL performance_schema
Network metrics           ipt netfilter, eBPF
Custom business metrics   "orders_completed_total{plan='pro'}"
```

### Prometheus Model

The de-facto standard for metrics in cloud-native.

- **Pull-based:** Prometheus scrapes /metrics endpoints periodically.
- **Time-series database:** stores samples efficiently.
- **PromQL:** powerful query language.
- **Alertmanager:** evaluates alert rules, deduplicates, routes notifications.

```promql
# Requests per second by endpoint
rate(http_requests_total[5m]) by (path)

# p99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

### OpenTelemetry

Vendor-neutral standard for telemetry (metrics, logs, traces). Replaces older APIs (OpenCensus, OpenTracing). Most languages have an OTel SDK that emits to any compatible backend.

```
App with OTel SDK -> OTel Collector -> {Prometheus, Datadog, Honeycomb, ...}
```

OTel allows backends to be swapped without changing application code.

---

## 4. Logs in Depth

### Structured Logging

Log records as structured data (JSON), not free-form strings.

```json
{
  "ts": "2025-01-15T10:30:42.123Z",
  "level": "info",
  "service": "orders",
  "trace_id": "abc-123",
  "span_id": "def-456",
  "user_id": 42,
  "event": "order_created",
  "order_id": "o-789",
  "duration_ms": 87
}
```

Queryable, parseable, machine-readable. Compared to `"Order created for user 42 in 87ms"`, it's:

- Searchable on any field.
- Aggregatable (e.g., "sum duration_ms by user_id").
- Joinable with trace IDs.

### Log Levels

```
TRACE / DEBUG  detailed diagnostic info, off in production by default
INFO           normal events worth recording
WARN           something unexpected but recoverable
ERROR          something failed, needs attention
FATAL          process is about to die
```

Production typically logs INFO + above. DEBUG is dynamically enabled when investigating.

### What To Log

```
Request start / end with: request ID, user ID, latency, status
External calls with: target, duration, success/failure
Business events: order created, payment processed, alert triggered
Errors with: stack trace, request context
Significant state changes
```

### What NOT To Log

```
Passwords, tokens, credit card numbers, API keys (even on error paths!)
Full PII unless required and audited
Full request/response bodies for sensitive endpoints
Inside hot loops (one log per millisecond = millions per day)
```

### Log Aggregation

Logs flow from many sources to a central store for searching.

```
App -> stdout -> log shipper (Fluent Bit, Vector, Filebeat)
                       |
                       v
                  Log backend (Elasticsearch/OpenSearch, Loki, Datadog, Splunk)
                       |
                       v
                  Search / dashboards / alerts
```

**Don't write to log files in containerized apps.** Write to stdout/stderr; let the platform capture and ship. This is the 12-factor convention.

### Log Volume and Cost

Logs are often the most expensive observability data because they're high-volume and full-text indexed.

Levers:

- Sample (keep 1% of INFO, all WARN+).
- Drop noisy logs (health checks, repetitive lines).
- Aggregate frequent logs into metrics (count by template, not log every event).
- Use cheaper backends for cold data (Loki, S3 archive).
- Set retention tiers (hot 30 days, archive longer).

### Correlation IDs

Every log line from a request includes a request/trace ID. Generated at the edge, propagated via headers.

```
Incoming HTTP: X-Request-ID: abc-123
  |            (generated if missing)
  v
Logs: request_id=abc-123 ...
  |
  v
Outgoing HTTP (to downstream): X-Request-ID: abc-123
                               (downstream uses the same ID)
```

When investigating an incident: grep by request ID across all services to reconstruct the timeline.

---

## 5. Distributed Tracing

A trace is the journey of a single request through the system. Composed of **spans** (units of work) with parent-child relationships.

```
Trace: GET /orders/42

Span: api-gateway   GET /orders/42       0.0  ─────────────────────────  120ms
  Span: auth-svc    verify-token         5.0  ──────  35ms
  Span: orders-svc  GET /orders/42       40.0 ─────────────────  100ms
    Span: db        SELECT * FROM ...    50.0 ─────── 70ms
    Span: cache     GET order:42         50.0 ─ 3ms (miss)
```

Reading: most of the latency is in `db`, specifically the SELECT. The trace shows exactly where the time went.

### OpenTelemetry Tracing

Same SDK as for metrics. Each operation becomes a span with attributes (tags) and events.

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("user.id", user_id)
    
    with tracer.start_as_current_span("validate"):
        validate(order)
    
    with tracer.start_as_current_span("charge_payment"):
        result = payment_service.charge(order.amount)
        span.set_attribute("charge.result", result)
```

The SDK propagates trace context across service boundaries via HTTP headers (`traceparent`).

### Sampling

Capturing every span is expensive. Sample.

- **Head-based sampling:** decide at the start of the trace (e.g., 1% of requests). Simple, biased toward fast paths.
- **Tail-based sampling:** decide at the end (after seeing the full trace). Can keep all errors, all slow traces, sample the rest. More useful but requires holding traces in memory until decision.

Honeycomb's Refinery and the OTel Collector's tail-sampling processor implement tail-based sampling.

### Trace Backends

| Backend | Notes |
|---------|-------|
| Jaeger | Open source, mature, good UI |
| Tempo | Grafana's trace backend, integrates with Loki/Prometheus |
| Zipkin | The original, still in use |
| Datadog APM | Commercial, integrated metrics/logs/traces |
| Honeycomb | Best for high-cardinality analysis |
| New Relic | Commercial, broad |
| AWS X-Ray | AWS-native |
| Lightstep / ServiceNow Cloud Observability | Commercial |

### When Tracing Helps

- Latency debugging in microservices (which hop is slow?).
- Identifying unexpected dependencies (this code path calls 14 services?).
- Understanding cold-start vs warm-path differences.
- Capacity planning per dependency.

When it doesn't help: monolith with no cross-service calls (logs and profiling are sufficient).

---

## 6. SLI / SLO / SLA / Error Budgets

The Google SRE framework for setting reliability targets.

### Definitions

**SLI (Service Level Indicator):** a measurement. "Fraction of requests returning 2xx in under 500ms."

**SLO (Service Level Objective):** a target for the SLI. "99.9% of requests in 28 days."

**SLA (Service Level Agreement):** an external commitment, typically with consequences (refunds) if missed.

**Error budget:** the inverse of the SLO. 99.9% SLO → 0.1% error budget.

### Why Error Budgets

At a 99.9% SLO with current performance at 99.95%, there is surplus budget — feature delivery can proceed quickly. Once the budget is exhausted, the priority shifts to reliability.

This frames reliability work as a continuous tradeoff rather than a demand for perfection.

```
SLO 99.9% over 28 days:
  Total budget: 0.1% * 28 days * 24 hours = ~40 minutes of "downtime"
                                              (or equivalent in error rate)

With 35 min consumed in 2 weeks:
  Action: freeze risky changes, focus on stability work.
```

### Choosing SLIs

Pick a few that **directly reflect user experience.**

Common SLIs:

- **Availability:** fraction of requests that succeed.
- **Latency:** fraction of requests under threshold (e.g., p99 < 300ms).
- **Throughput:** lower bound on QPS.
- **Quality:** fraction of "good" responses (no fallback, no degraded mode).
- **Freshness:** for derived data, max age (data is at most 5 min stale).

The wrong SLI ("CPU usage < 80%") tracks a means, not an end. Users don't care about CPU. They care about whether the app responds.

### SLO Window

Most SLOs measure over a **rolling 28-day window**. Long enough to absorb single incidents, short enough to react.

### Error Budget Policy

The agreement around what happens when the budget is exhausted:

```
- All planned changes pause; only reliability fixes deploy.
- Postmortem required on any incident that consumed >X% of budget.
- Engineering leadership reviews ongoing reliability work.
- After three consecutive periods of missing SLO, formal reliability sprint.
```

This is a managerial mechanism, not just a metric.

---

## 7. The USE and RED Methods

Two complementary frameworks for what to measure.

### USE — for Resources

For every resource (CPU, memory, disk, network), measure:

- **Utilization:** average % busy.
- **Saturation:** queued/waiting work.
- **Errors:** error events.

```
CPU:       Utilization (%) | Saturation (run queue len, load avg) | Errors (rare)
Memory:    Used / Total    | Swap usage, page faults              | OOMs
Disk:      I/O utilization | I/O wait time, queue depth           | I/O errors
Network:   Bandwidth used  | Drop counters, retransmits           | Frame errors
```

USE is a checklist when investigating a host. Each line is a question; missing data indicates incomplete observability.

### RED — for Services

For every service, measure:

- **Rate:** requests per second.
- **Errors:** failed requests per second (or fraction).
- **Duration:** latency (with percentiles).

```
GET /orders:
  Rate:     1250 req/s
  Errors:   0.3% (5xx + relevant 4xx)
  Duration: p50 45ms, p99 280ms, p999 1.2s
```

RED for every service plus USE for every host constitutes the foundation of an observability strategy.

### The Four Golden Signals

Google SRE's variant:

- Latency
- Traffic
- Errors
- Saturation

Saturation is the "near-the-cliff" signal — how full are buffers, queues, thread pools? Before users see errors, saturation rises.

---

## 8. Alerting

Alerts wake people up. Each alert is a cost — to the alerted human, to the team's morale, and to credibility.

### Good Alerts

```
- Specific:  "p99 latency on /payments > 1s for 10 min" (not "things look slow")
- Actionable: there's something to do about it
- Symptom-based: alert on what users see, not on internal causes
- Linked to a runbook: what to check, what to do
- Has a clear owner
```

### Bad Alerts

```
- "CPU usage > 80%"       (not inherently user-visible)
- "Free disk < 20%"       (in many cases, not an incident)
- "Restarted again"       (informational, not actionable)
- Flapping                (fires and clears repeatedly)
- Inverted                (clears at 80%, fires at 85% — fires twice during normal recovery)
```

### Alert Fatigue

Too many alerts → people ignore them → real ones get missed.

Treatments:

- Continuously audit: which alerts fire and produce action? Cut the rest.
- Alert on symptoms, not causes. (One SLO-burn alert beats 30 cause alerts.)
- Tier severity: pager for SLO breach; ticket for trends.
- Alert on rates and ratios, not absolutes (a sudden 10x increase in errors is more informative than "errors > 100").

### Burn Rate Alerts

For SLO-driven alerting: page when the budget is being consumed faster than a defined rate.

```
For an SLO of 99.9% over 28 days:
  Sustained 14.4x burn rate exhausts budget in 2 hours.
  Sustained 1x burn rate exhausts in exactly 28 days.

Page if:
  Last 1h has 14.4x burn rate AND last 5m has 14.4x burn rate.

(Two-window: avoid pager on a transient spike that's already over.)
```

Google's SRE book has the detailed math.

### Runbooks

Each alert links to a runbook:

```
ALERT: p99 latency on payments > 1s

WHAT IT MEANS:
  Users may be seeing slow checkout.
  
FIRST CHECK:
  - Dashboard: link to latency breakdown by endpoint
  - Recent deploys to payments service: link
  - Downstream health (Stripe API, etc.): link

MITIGATION:
  - If a recent deploy: roll back
  - If Stripe is degraded: enable fallback queue (link to ops doc)
  - If DB is slow: link to DB triage runbook

ESCALATION: on-call for payments team
```

Even a rough runbook is better than waking an on-call engineer who must reconstruct all context from scratch.

---

## 9. Dashboards and Visualization

### Dashboard Hierarchy

```
Tier 1 (overview):     "is the system healthy?"
                       SLOs, error budgets, top-line metrics

Tier 2 (per-service):  "is this service healthy?"
                       RED metrics, dependency health, deploy events

Tier 3 (deep dive):    "what specifically is wrong?"
                       Granular metrics, traces, related logs
```

A good top-level dashboard answers "is everything OK?" in 10 seconds. Drill-downs support deeper investigation.

### Principles

- **Show user-visible metrics first.** Latency, errors, availability. Resource metrics come second.
- **Annotate deploys.** Most regressions correlate with recent changes; deploy markers on graphs save minutes per incident.
- **Comparable time series.** "now vs last week" overlays surface anomalies that absolute thresholds miss.
- **Avoid graphing everything.** A dashboard with 200 panels is unreadable.
- **Make graphs interactive.** Click to filter, click to see traces, click to see logs.

### Good Visualization Choices

- **Line charts** for time series.
- **Heatmaps** for distributions over time (great for latency).
- **Single-stat / gauge** for current value + threshold.
- **Stacked area** for composition (errors by status code).

Avoid 3D anything, exploded pie charts, rainbow gradients on continuous data.

---

## 10. Profiling in Production

Profilers report where time and memory are going in a live system. This section covers profiling as an *operational* tool — attaching to running processes, fleet-wide continuous profiling, and reading the result during an incident. For profiling as part of the optimization loop (sampling vs instrumentation, flame-graph mechanics, microbenchmark pitfalls), see `Performance Optimization.md` section 3.

### Profiling a Live Process

The key operational property is **low overhead** and **no redeploy**: the goal is to attach to a process that is already misbehaving in production.

```
perf (Linux)           system-wide or per-process; needs no app changes
pprof (Go, others)     Go exposes /debug/pprof over HTTP; grab a profile live
async-profiler (Java)  low-overhead JVM profiler, attaches to a running PID
py-spy (Python)        sampling, attaches to a running process without restart
rbspy (Ruby)
```

The output is typically a **flame graph** (wide bars at the bottom = hot leaf functions; see `Performance Optimization.md` for how to read one).

### Memory Profiling

Find allocations and leaks in a running service.

```
Go: go tool pprof -alloc_space (allocation count)
    go tool pprof -alloc_objects (object count)
    go tool pprof -inuse_space (currently held)

Java: jcmd <pid> GC.heap_info, async-profiler -e alloc
Python: tracemalloc, memray
C/C++: valgrind --tool=massif, heaptrack
```

A memory leak shows up as growing in-use memory over time. An allocation hot spot shows up as high alloc_objects in a particular function.

### Continuous Profiling

Run profilers continuously in production at low overhead (~1%). Aggregate over time and across instances — this is the observability framing of profiling: an always-on telemetry signal, not a one-off investigation.

Tools: Pyroscope, Parca, Polar Signals, Datadog Continuous Profiler. These answer questions such as "what was the hottest function last Tuesday at 3pm?" or "what changed when v2.3.1 was deployed?"

### Per-Operation Profiling

To understand a specific request rather than the aggregate:

- **Distributed traces** (section 5) — where time went across services.
- **strace** — which syscalls a process is making and how long they take.
- **Application-level tracing** (e.g., Postgres `EXPLAIN ANALYZE`) for individual queries.

### Reading a Profile During an Incident

Common patterns that jump out of a production profile:

- **Hot lock contention:** time in `sync_lock_acquire` / `futex`.
- **Excessive allocation:** `mallocgc`, `malloc`, `tcmalloc` near the top.
- **Slow syscalls:** `read`, `write` (`epoll_wait` at the top is usually fine — it's just waiting).
- **Reflection / serialization:** JSON marshaling, reflection-heavy ORMs.
- **Compression:** gzip/zlib at the top means too much compression on the hot path.

Once the hot spot is located, the *fix* is an optimization problem — see `Performance Optimization.md`.

---

## 11. Latency Tail and p99 Thinking

The hardest part of latency is the long tail. This section is about *observing and reasoning about* the tail; for the causes (queueing, Little's Law) and engineering techniques to reduce it (hedged requests, headroom, bounded queues), see `Performance Optimization.md`.

```
Service A:  p50 = 50ms, p99 = 200ms, p999 = 5s
```

p50 appears excellent. But for a page that makes 10 parallel calls to A, the probability that all 10 return within 200ms is `0.99^10 = 90.4%`. **10% of pages experience the p99 of A.**

```
Probability that user sees > Tservice:
  1 - (1 - tail_prob)^N
  where N = number of calls per user request
```

The implications for what to measure and target:

- The more dependencies a request has, the more p99 (not p50) matters.
- For systems with fan-out, the SLO must target the tail, because the tail is what users hit.
- Watch p99/p999 trends, not just the median — a regression often appears in the tail first.

### p99 vs Average

Averages are misleading. A service with 99% of requests at 10ms and 1% at 10s has an average of ~110ms — but the actual experience is bimodal.

Look at distributions (histograms, heatmaps), not averages. This is why histogram metrics (section 3) and latency heatmaps (section 9) exist: they make the tail visible where an average hides it.

---

## 12. Load Testing

Verify the system meets its non-functional requirements under load.

### Types

```
Smoke test:        small load, validate basic correctness
Load test:         expected load, validate performance under it
Stress test:       beyond expected load, find the breaking point
Spike test:        sudden burst, validate response
Soak test:         sustained load for hours/days, find leaks
```

### Tools

```
k6                 modern, scripts in JS, great UX
Gatling            JVM, sophisticated reporting
Locust             Python, easy to write scenarios
wrk / wrk2         simple, fast HTTP load tester
Vegeta             Go, command-line, attack lists
Artillery          JS, similar to k6
JMeter             venerable, GUI, complex
```

Example k6 script:

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    ramp_up: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(99)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://api.example.com/orders');
  check(res, {
    'status 200': (r) => r.status === 200,
    'latency under 200ms': (r) => r.timings.duration < 200,
  });
}
```

### Designing Realistic Load

Bad load test: repeatedly issue the same request to one endpoint from a single IP.

Good load test:

- Mix of endpoints proportional to real traffic.
- Realistic think time between requests.
- Variability in payloads and parameters.
- Distributed source (not just one machine).
- Warm-up phase before measurement.

### What to Measure

- **Throughput at SLO compliance:** how many RPS can be sustained while meeting the SLO?
- **Tail latency degradation:** when does p99 start to spike?
- **Resource saturation:** what hits 100% first — CPU, memory, network, DB?
- **Recovery:** when the load stops, does the system return to baseline?

### Production-Like Environments

Load tests in dev rarely predict production behavior because:

- Data volumes differ (100 rows in dev vs 100M in prod).
- Concurrency patterns differ.
- Dependencies are stubbed.
- Hardware/network differs.

Either: load test in a production-equivalent staging (expensive), or shadow traffic from production to a test environment.

---

## 13. SRE Practice

Site Reliability Engineering — Google's framework for operating production systems.

### Core Ideas

- **Reliability as a product feature.** Defined by SLOs, traded off against feature work via error budgets.
- **Engineers, not operators.** SREs write code (automation, tooling), not just respond to alerts.
- **Toil reduction.** Repetitive manual work is automated.
- **Blameless postmortems.** Failures are systemic, not personal.
- **Game days and disaster recovery exercises** are normal.

### Incident Response

A common process:

```
1. Detect      alert fires
2. Triage      what's the scope? severity?
3. Mitigate    stop the bleeding (rollback, kill bad deploy, fail over, throttle)
4. Investigate find root cause
5. Resolve     restore full service
6. Postmortem  what went wrong, what to fix
```

**Mitigation comes before investigation.** Stop the impact first — even before the cause is fully understood.

### On-Call

- A rotation, not a permanent assignment.
- Pages should be rare; if they're not, fix the alerts/services.
- Compensation or time-off for on-call hours is humane and standard.
- Handoffs include current state of any open incidents.

### Postmortems

A document for every significant incident:

```
- Summary (what happened, user impact, duration)
- Timeline (detection, mitigation, resolution, with timestamps)
- Root cause analysis (often multiple contributing factors)
- What went well (detection was fast? Rollback worked?)
- What went poorly (alerts didn't fire? On-call missed it?)
- Action items (specific, owned, dated)
```

**Blameless.** "Engineer X deployed bad code" is not a useful root cause. "Our CI didn't catch this class of bug; we deployed without proper checks" is.

### Chaos Engineering

Inject failures in production (or production-like environments) to verify resilience. Netflix's Chaos Monkey kills random instances. Modern: Gremlin, Litmus, Chaos Mesh, AWS Fault Injection Service.

Game days: planned chaos exercises — for example, terminating the database in staging at a scheduled time to test whether the team detects and mitigates it.

### Production Readiness

Before launching a new service:

```
- SLOs defined
- Alerts wired up
- Dashboards built
- Runbooks written
- Capacity planning done
- Backups configured
- Failover tested
- Load tested
- Security reviewed
- On-call rotation established
```

Lists like this are checklists, but they catch real gaps.

---

## 14. Tools Reference

### Metrics

```
Prometheus + Alertmanager      open-source, pull-based, the standard
Grafana                        dashboards over many backends
Thanos / Mimir / Cortex        long-term storage for Prometheus
VictoriaMetrics                Prometheus-compatible, faster
InfluxDB                       time-series DB
Datadog, New Relic, Dynatrace  commercial all-in-ones
```

### Logs

```
Elastic / OpenSearch + Kibana    full-text indexed, expensive at scale
Loki + Grafana                   log aggregation indexed by labels (cheap)
Fluent Bit, Vector, Filebeat     log shippers
Splunk                           commercial, enterprise
Datadog Logs, Sumo Logic         SaaS
journald                         systemd's binary log
```

### Tracing

```
Jaeger, Tempo, Zipkin           open-source backends
OpenTelemetry                   instrumentation standard
Honeycomb                       commercial, great for high-cardinality analysis
AWS X-Ray, GCP Cloud Trace      cloud-native
```

### Profiling

```
perf, perf-tools, FlameGraph      Linux + Brendan Gregg toolkit
pprof                             Go's built-in profiler
async-profiler                    JVM
py-spy, scalene, memray           Python
Pyroscope, Parca, Polar Signals   continuous profiling
```

### Load Testing

```
k6, Gatling, Locust, wrk, JMeter
```

### Network Performance

```
iperf3                  bandwidth between hosts
mtr                     traceroute + ping
ping                    round-trip reachability and latency
tcpdump, wireshark      packet capture
ss, netstat             socket stats
nstat                   netstat-style counters
ethtool                 NIC stats
```

### System

```
top, htop, btop, atop          processes
vmstat, mpstat, iostat, sar    resources
free, smem                     memory
lsof, fuser                    fds
strace, ltrace                 syscalls / lib calls
bpftrace, bcc, perf            tracing / profiling
```

### A Brief Methodology

When investigating a performance problem in production:

1. **Define the symptom.** "Page load is slow" is too vague. "p99 latency on /api/orders is 2s, was 300ms last week" is specific.
2. **Find a recent change.** Most regressions correlate with deploys, config changes, or traffic changes.
3. **Use the dashboards.** RED for the affected service; USE for the affected hosts; correlations across services.
4. **Use traces.** For one slow request, where is the time going?
5. **Use profiling.** Where is the CPU going? Where is memory going?
6. **Form a hypothesis.** For example, the new DB query in the order-summary endpoint.
7. **Test the hypothesis.** Change one thing; observe whether metrics improve.
8. **Verify the fix.** Watch the SLO and confirm.
9. **Document the findings.** Even a brief written summary suffices.

The most expensive bug is the one that cannot be reproduced. Observability's purpose is to make production reproducible. Once a bottleneck is localized, the engineering fix lives in `Performance Optimization.md`.
