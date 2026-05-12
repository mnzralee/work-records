# Track 1 — Observability and Debugging: Industry Practice

> Senior-architect research study, May 2026.
> Scope: the three pillars (logs, metrics, traces) plus continuous profiling, exemplars, and the canonical mature-team setups behind them. No GX-specific gap analysis — that is the orchestrator's job. Every claim is anchored to a named tool, company, or source.

---

## 1. Canonical Models

### 1.1 Charity Majors / Honeycomb — Observability vs Monitoring

Charity Majors (CTO, Honeycomb) has spent eight years drawing a sharp operational distinction: **monitoring handles known-unknowns, observability handles unknown-unknowns**. Monitoring assumes you know in advance which threshold will break and writes a check for it; observability instruments the system so richly that any new question can be answered against existing data without shipping new code. The operational implication is that the dashboard-and-alert paradigm of Nagios-era monitoring is structurally incapable of debugging modern distributed systems where every outage is novel. Honeycomb's product thesis is that you achieve this by recording **wide, structured events with high cardinality and high dimensionality** — one event per unit of work, with every interesting attribute attached.

**"High cardinality"** means a field has many unique values (e.g., `user_id`, `request_id`, `build_sha`, `shopping_cart_id` — fields with millions of distinct values). **"High dimensionality"** means an event has many such fields (often 100+ per span). The two together let an engineer drill in like: "show me traces from Canadian iOS users on build 4.7.1 hitting `/checkout` with status 500 in the last 10 minutes" — a query no time-series metric system can answer because pre-aggregation throws cardinality away.

**Tools**: Honeycomb (the canonical implementation), Lightstep (now ServiceNow Cloud Observability — being EOL'd March 2026), Grafana Tempo + TraceQL (open-source approximation).

**Read**:
- [How Observability Differs from Traditional Monitoring — Honeycomb](https://www.honeycomb.io/blog/observability-differs-traditional-monitoring)
- [Understanding High Cardinality and Its Role in Observability — Honeycomb](https://www.honeycomb.io/resources/getting-started/understanding-high-cardinality-role-observability)
- [Charity Majors on Observability — InfoQ](https://www.infoq.com/articles/charity-majors-observability-failure/)

### 1.2 Google SRE — The Four Golden Signals

Chapter 6 of Google's *Site Reliability Engineering* book (2016) reduces user-facing service health to four signals: **latency** (time to service a request, measured separately for successes vs failures), **traffic** (demand on the system — RPS, sessions, bytes), **errors** (rate of failed requests), and **saturation** (how full the service is — CPU, memory, queue depth). The argument is empirical: across hundreds of Google services, these four catch the overwhelming majority of user-impacting incidents, and any team with limited monitoring budget should instrument them first before adding anything else. They map cleanly to the RED method (Rate, Errors, Duration) for request-driven services and the USE method (Utilization, Saturation, Errors) for resources, both of which are simplifications of the same underlying observation.

**Tools**: Prometheus + Grafana dashboards keyed on these four; Google Cloud Operations (formerly Stackdriver) bakes them in as default templates.

**Read**:
- [Monitoring Distributed Systems — Google SRE Book Ch. 6](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Four Golden Signals — Splunk](https://www.splunk.com/en_us/blog/learn/sre-metrics-four-golden-signals-of-monitoring.html)

### 1.3 Three Pillars and Cindy Sridharan's Critique

Cindy Sridharan's 2018 O'Reilly book *Distributed Systems Observability* popularised the **logs / metrics / traces** triad, but the book itself argues that the framing is reductive. Her critique, repeated in talks and her widely-cited Medium essays, is that the three-pillars model has been twisted into a procurement checklist — "we bought a tool for each pillar, we are now observable" — when in practice observability emerges only when the three are **correlated** through shared context (trace IDs, request IDs, tenant IDs) and the underlying telemetry is **wide enough** to answer questions you didn't plan for. Logs without trace IDs are search-only; metrics without exemplars are summary-only; traces without sampling discipline are storage-bankruptcy. The pillars are the raw materials; the architecture is what makes them observable.

**Tools**: OpenTelemetry (the modern answer — unifies the three pillars under one SDK and protocol).

**Read**:
- [Distributed Systems Observability — Cindy Sridharan, O'Reilly (free PDF)](https://unlimited.humio.com/rs/756-LMY-106/images/Distributed-Systems-Observability-eBook.pdf)
- [Ch. 4 — The Three Pillars of Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)

---

## 2. Logs

### 2.1 Structured Logging (JSON) — Why Universal

Structured JSON logs are the floor for any team above ~10 services. The reason is mechanical: free-form text logs are searchable but not aggregatable — you cannot compute "p99 latency by tenant" from `2026-05-01 12:34:56 user fred bought 3 widgets` without writing a regex parser per log line, and the parser breaks every time a developer reformats the message. JSON logs make every field a first-class column in the downstream warehouse, queryable in SQL or LogQL without text-mining. Mature teams converge on a small set of mandatory fields — `timestamp`, `level`, `service`, `trace_id`, `span_id`, `request_id`, `tenant_id`, `user_id` — because those are the join keys that connect a log line back to the trace, the metric exemplar, and the user-reported incident.

**Tools**: Pino (Node), Zap (Go), Logback + Logstash JSON encoder (Java), structlog (Python), tracing crate (Rust).

**Read**:
- [Logging in Node.js: Comparison of Top 8 Libraries — Better Stack](https://betterstack.com/community/guides/logging/best-nodejs-logging-libraries/)

### 2.2 Pino vs Winston vs Bunyan

In Node.js, the contest is functionally over: **Pino has won the new-codebase market, Winston has won the legacy installed base**. Pino is the default logger for Fastify and is benchmarked at 5–10× Winston's throughput, with weekly downloads growing from ~3M (2022) to ~9M (2026); Winston's 15M downloads reflect a large but ageing footprint. Pino's architectural advantage is that it serialises a minimal JSON string to stdout asynchronously and offloads transports to a worker thread, keeping the event loop free — Winston runs its formatter pipeline on the main loop and adds ~47ms per 1k messages. Bunyan introduced the JSON-first idea in 2014 but has effectively stopped active development; it is no longer recommended for greenfield work.

**Pattern**: Pino + `pino-http` (request logger middleware) + `pino-pretty` only in dev + `pino-otlp-transport` to ship to an OpenTelemetry collector in prod.

**Read**:
- [Pino vs Winston in 2026 — PkgPulse](https://www.pkgpulse.com/guides/pino-vs-winston-2026)
- [Benchmark: Winston 3.11 vs Pino 8.0 on Node 24](https://johal.in/benchmark-winston-311-vs-pino-80-nodejs-24/)

### 2.3 Sampling vs Full Ingestion at Scale

At Cloudflare scale (millions of HTTP requests per second), full log ingestion into Elasticsearch broke down — ingestion bottlenecks and storage cost made it untenable. Their answer was a migration to **ClickHouse**, a column-store that gives 10–20× compression, lets compute scale separately from cheap object storage, and ingests ~90M rows/sec across a 36-node, 3×-replicated cluster. Crucially, they **kept full ingestion** (no head sampling) and used ClickHouse's columnar economics to make it affordable; a single query can scan 96 trillion events in under two seconds. The lesson for mid-size teams: sampling is a structural admission that your storage tier is wrong; before sampling, audit whether moving from Elastic/Splunk to ClickHouse-backed Loki/SigNoz/ClickStack would solve the cost problem at full fidelity.

When sampling is unavoidable (large free-text logs, debug verbosity), tiered approaches dominate: keep 100% of error/warn lines, sample info logs at 1–10%, drop debug in prod entirely. Lyft and similar microservice shops route by severity through a Fluent Bit / Vector pipeline before ingestion.

**Tools**: ClickHouse (Cloudflare, Uber, Netflix logging), Vector (Datadog-acquired log router), Grafana Loki (label-indexed, body-not-indexed — cheap by design).

**Read**:
- [An Overview of Cloudflare's Logging Pipeline](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/)
- [HTTP Analytics for 6M req/s using ClickHouse — Cloudflare](https://blog.cloudflare.com/http-analytics-for-6m-requests-per-second-using-clickhouse/)
- [Beyond Sampling: Petabyte-scale logs — ClickHouse](https://clickhouse.com/resources/engineering/managing-petabyte-scale-logs-without-sampling)

### 2.4 Correlation Fields — The Mandatory Schema

Every log line in a mature stack carries: `trace_id` (32-hex from W3C trace-context), `span_id` (16-hex), `service.name`, `deployment.environment`, `tenant_id`, and a per-request `request_id` (UUID generated at the ingress). The trace ID is the master key — it joins logs, metrics exemplars, and the trace itself. Without it, "go from a metric spike to the failing request" requires a human grepping by timestamp. With it, the dashboard hyperlinks directly to the offending span. OpenTelemetry's logging SDK now injects these fields automatically when a log call happens inside an active span, removing the discipline burden from developers.

### 2.5 Tools — What Mid-Size Orgs Actually Run

| Stack | Typical Org | Strength | Cost Profile |
|---|---|---|---|
| **Grafana Loki + Tempo + Mimir** | OSS-leaning, Kubernetes-native, $50M-$500M ARR | Cheapest, label-indexed, single pane in Grafana | $19/mo Cloud Pro entry; self-host trivial |
| **Elastic / OpenSearch** | Legacy enterprises, security teams | Best free-text search, mature alerting | Storage-heavy; row format doesn't compress |
| **Splunk** | Fortune-500 finance/regulated | Enterprise SIEM dominance | $15k/yr at 5GB/day; >$1M at 600GB/day |
| **Datadog Logs** | SaaS unicorns, fast-moving teams | Best-in-class UX, single vendor for logs+APM+metrics | Notorious super-linear scaling; six-figure surprise bills |
| **ClickHouse-backed (SigNoz, ClickStack, Parseable)** | Cost-conscious mid-size, 2024–26 wave | OTel-native, columnar economics, SQL queries | 10–20× cheaper than Elastic at scale |

The 2024–2026 trend is unmistakable: teams under cost pressure are migrating off Datadog Logs / Splunk / Elastic onto Loki or ClickHouse-based stacks, accepting some UX downgrade for 5–20× cost reduction.

**Read**:
- [Why Orgs Use Grafana + Loki to Replace Datadog — ChaosSearch](https://www.chaossearch.io/blog/why-organizations-use-grafana-loki-to-replace-datadog)
- [Datadog vs Grafana Pricing — Vantage](https://www.vantage.sh/blog/datadog-vs-grafana-cost)

### 2.6 Stripe's Canonical Log Lines — The Pattern to Steal

Stripe's `canonical-log-lines` pattern, documented on the Stripe engineering blog, is the highest-leverage logging pattern for any service-oriented backend: **emit exactly one structured log line per request at the end of the request**, containing every important field (HTTP method, path, status, duration, user_id, account_id, request_id, DB query count, DB time, cache hits, feature flags evaluated, error class). Conventional debug logs continue to fire, but the canonical line is the queryable record. This collapses "what happened on this request" into one row, making the log warehouse function as a wide-event store — the same model Honeycomb sells as a product, achievable with Pino + a request-scoped context object. Emit it from a Ruby `ensure` block (or Node `finally`, Express error-handler) so it fires even when the request crashes.

**Read**:
- [Fast and Flexible Observability with Canonical Log Lines — Stripe](https://stripe.com/blog/canonical-log-lines)
- [Brandur Leach — Canonical Log Lines](https://brandur.org/canonical-log-lines)

---

## 3. Metrics

### 3.1 Prometheus + Grafana — The De Facto Standard

Prometheus graduated CNCF in August 2018 and is reported by **77% of CNCF Annual Survey 2023 respondents** as in production use; ~63% of Kubernetes-running orgs run the Prometheus + Grafana pair. Its dominance is not because it is the best metrics database (it isn't — VictoriaMetrics, Mimir, and Thanos all out-scale it on raw cardinality) but because it is the **interface contract** the cloud-native ecosystem has converged on: every Kubernetes operator, every sidecar, every CNCF project ships a `/metrics` endpoint in the Prometheus exposition format. You don't choose Prometheus — you inherit it.

**Tools**: Prometheus (scrape + storage), Grafana (visualisation), Alertmanager (routing), Grafana Mimir / Thanos / VictoriaMetrics (long-term storage at scale).

**Read**:
- [Prometheus Project — CNCF](https://www.cncf.io/projects/prometheus/)
- [Grafana Observability Survey 2024](https://grafana.com/observability-survey/2024/)

### 3.2 Counter, Gauge, Histogram, Summary — When Each Is Right

- **Counter** — monotonically increasing (resets to zero on process restart). Use for `requests_total`, `errors_total`, `bytes_sent_total`. Always query with `rate()` or `increase()`, never the raw value.
- **Gauge** — value that can go up or down. Use for `queue_depth`, `memory_bytes`, `connections_active`, `temperature_celsius`.
- **Histogram** — pre-bucketed distribution counts. Use for **latency, payload size, anything where the average lies**. Computed server-side via `histogram_quantile()`. Aggregatable across instances. **Default choice for latency.**
- **Summary** — quantiles computed client-side. Cheaper to query, **NOT aggregatable across instances** (you cannot meaningfully average two p99s). Use only when you have a single instance or genuinely don't need cross-instance aggregation. Stripe's Veneur exists explicitly because summaries don't aggregate — they had to build a global aggregation tier to compute true p99s across hundreds of API hosts.

**Read**:
- [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)
- [Histograms and Summaries — Prometheus](https://prometheus.io/docs/practices/histograms/)
- [Stripe Veneur — Global Percentile Aggregation](https://stripe.com/blog/introducing-veneur-high-performance-and-global-aggregation-for-datadog)

### 3.3 SLI / SLO / SLA — The Practitioner's Recipe

**SLI** — a measurement of one user-perceived dimension (e.g., `successful_requests / total_requests`). **SLO** — a target on that SLI over a window (e.g., `99.9% over 28 days`). **SLA** — the contractual version with financial consequences, always set looser than the SLO so the SLO trips first. The **error budget** is `1 - SLO`: a 99.9% SLO over 28 days × 3M requests = 3,000 error budget; once consumed, by Google's policy, the team **freezes feature releases** and shifts to reliability work until the next window. This is the operational mechanism that gives reliability teeth — without an enforced budget, "reliability" is aspirational.

The canonical recipe:
1. Define 2–4 SLIs per user journey (latency p99, success ratio, freshness).
2. Set SLOs from observed-baseline-minus-realistic (don't pick 99.99% if you currently run 99.5%).
3. Burn-rate alerts (multi-window: 1h / 6h / 3d) — alert when budget consumption rate would exhaust the window.
4. Error-budget policy in writing, signed by product + engineering.

**Tools**: Sloth (OSS SLO generator → Prometheus rules), Nobl9 (SaaS SLO platform), Grafana SLO, Datadog SLOs.

**Read**:
- [Implementing SLOs — Google SRE Workbook](https://sre.google/workbook/implementing-slos/)
- [Alerting on SLOs (burn-rate) — Google SRE Workbook](https://sre.google/workbook/alerting-on-slos/)
- [Error Budget Policy — Google SRE Workbook](https://sre.google/workbook/error-budget-policy/)

### 3.4 Cardinality Explosions

Every label value combination on a Prometheus metric creates a separate time series. Add `user_id` as a label on a 10M-user system and you get 10M time series per metric — Prometheus runs out of RAM and dies. The rule: **labels must have bounded, low-cardinality values**. `http_method` (5 values), `status_class` (5 values), `endpoint` (50 templated routes — never raw paths) — fine. `user_id`, `request_id`, `path` (with parameters interpolated), `error_message` (free text) — banned. Trace IDs and user IDs belong on the trace/log side, not the metric side; **exemplars** (next section) bridge them.

**Read**:
- [Prometheus Metrics Types: A Deep Dive — Last9](https://last9.io/blog/prometheus-metrics-types-a-deep-dive/)

---

## 4. Distributed Tracing

### 4.1 OpenTelemetry — The Industry Shim

OpenTelemetry (OTel) is the CNCF-incubating successor to OpenTracing and OpenCensus, and as of the 2025 follow-up survey is firmly **production infrastructure**: 65% of orgs run >10 collectors, 81% on Kubernetes, with 83% collecting metrics, 61% logs, 25% traces through OTel. The collector is a vendor-neutral pipeline with three stages — **receivers** (OTLP, Prometheus scrape, Jaeger, Zipkin, Fluent Forward), **processors** (batching, filtering, attribute enrichment, tail sampling), **exporters** (to Jaeger, Tempo, Honeycomb, Datadog, Splunk, Loki, etc.). Architecturally, the collector lets you instrument once with the OTel SDK and switch backends without code change — the practical end of vendor lock-in for telemetry.

**Tools**: OpenTelemetry SDK (per-language), OTel Collector (Go binary, deploy as agent DaemonSet + gateway StatefulSet).

**Read**:
- [OpenTelemetry Collector Architecture](https://opentelemetry.io/docs/collector/architecture/)
- [OTel Collector Follow-up Survey 2026](https://opentelemetry.io/blog/2026/otel-collector-follow-up-survey-analysis/)

### 4.2 W3C Trace Context — `traceparent` / `tracestate`

The W3C Trace Context Recommendation (Level 1 in 2020, Level 2 in 2024) defines two HTTP headers that every modern tracing library propagates by default. **`traceparent`** carries `version-traceid-parentid-flags` (e.g., `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`) — the trace ID is the same 32-hex string from request ingress to terminal database call, regardless of how many service hops occurred. **`tracestate`** carries vendor-specific extensions as comma-separated key=value pairs. The standardisation matters because before W3C, every vendor (Zipkin's `B3-*`, Jaeger's `uber-trace-id`, AWS X-Ray's `X-Amzn-Trace-Id`, Datadog's `x-datadog-*`) used their own headers, and a request crossing three vendors lost the trace at every boundary. W3C means a Stripe webhook hitting a customer's Express service hitting their Postgres can produce one continuous trace.

**Read**:
- [W3C Trace Context Recommendation](https://www.w3.org/TR/trace-context/)
- [W3C Trace Context Level 2](https://www.w3.org/TR/trace-context-2/)

### 4.3 Sampling Strategies

- **Head-based (probabilistic)** — decision made at the trace's first span. Cheap, simple, deterministic across services (if all services see the same sampling flag). **Loses every interesting rare trace** because it can't know which traces will turn out to be slow or failed.
- **Tail-based** — buffer the entire trace, then decide. Honeycomb's **Refinery** is the canonical implementation: it ingests all spans, buffers per trace ID, and on trace completion applies rules — keep all 5xx traces, keep all p99-latency traces, sample 1% of healthy traces. The OTel Collector ships a `tailsamplingprocessor` with similar capability. Tail sampling is operationally expensive (you pay buffer memory, you can't shed load by dropping early) but it's the only way to keep all the interesting traces at low total volume.
- **Dynamic / adaptive sampling** — Refinery's signature feature: per-trace-key (e.g., per-endpoint, per-status-code), increase sample rate for rare traffic and decrease for noisy traffic, so the budget is spent on diversity rather than the most common path.

For most teams: head-sample at the gateway (1–10%), then tail-sample errors and slow traces at 100% via OTel collector or Refinery. This is roughly Honeycomb's recommended starting topology.

**Read**:
- [Refinery — Honeycomb](https://github.com/honeycombio/refinery)
- [Tuning Refinery Dynamic Sampling — Honeycomb](https://www.honeycomb.io/blog/tuning-refinery-dynamic-sampling)

### 4.4 Backends

| Backend | Origin | Strength | OSS / SaaS |
|---|---|---|---|
| **Jaeger** | Uber, donated to CNCF (graduated) | Mature, OTel-native, Kubernetes-native | OSS |
| **Grafana Tempo** | Grafana Labs | Object-storage-backed (S3/GCS), cheap, integrates with Loki/Mimir | OSS + Cloud |
| **Honeycomb** | Christine Yen / Charity Majors | Wide events, BubbleUp, Refinery, the high-cardinality leader | SaaS |
| **Lightstep / ServiceNow Cloud Observability** | Ben Sigelman | Was the OTel-native leader; **EOL March 2026** | SaaS (dying) |
| **Datadog APM** | Datadog | Single-vendor across logs/metrics/APM, tight UX | SaaS (expensive) |
| **Tempo + Grafana** | Grafana Labs | Composable open-source LGTM stack | OSS |

The Lightstep EOL is a watershed: a generation of OTel-mature teams are migrating to Honeycomb, SigNoz, or Grafana Tempo through 2026.

**Read**:
- [Evolving Distributed Tracing at Uber Engineering](https://www.uber.com/blog/distributed-tracing/)
- [Top Lightstep Alternatives 2026 — SigNoz](https://signoz.io/comparisons/lightstep-alternatives/)

### 4.5 Trace-Driven / Observability-Driven Development

Coined by Charity Majors, **observability-driven development (ODD)** asks engineers to think about what spans, attributes, and traces a feature should produce **before** they write the code, then verify the feature in production by querying its traces — not by asserting on test fixtures. The Honeycomb + Tracetest integration operationalises this: write tests that assert on real OTel spans collected from a live service. In practice, ODD looks like (a) every PR adds explicit `span.setAttribute` calls for the new code path's interesting fields, (b) reviewers ask "what trace would prove this works?" instead of only "what unit test covers this?", (c) post-deploy, the engineer opens the Honeycomb trace view filtered to the new attribute and watches real traffic. It is closer to the Erlang/observability-built-in tradition than to the test-pyramid-and-pray tradition.

**Read**:
- [What Observability-Driven Development Is (and Isn't) — Honeycomb](https://www.honeycomb.io/blog/observability-driven-development)
- [Observability-Driven Development with Honeycomb + Tracetest](https://www.honeycomb.io/blog/honeycomb-tracetest-observability-driven-development)

---

## 5. Continuous Profiling — The Fourth Pillar

Always-on production profiling captures CPU stack samples and memory allocations at low frequency (typically 100Hz) with **<1% overhead** thanks to eBPF and async-profiler-class techniques. Where a metric tells you "p99 latency rose at 14:03" and a trace tells you "the slow span is `OrderService.charge`", a profile tells you **which line of code inside that span was burning CPU**. Profiling closes the loop from "where is the latency" to "what to change in the code".

| Tool | Origin | Notes |
|---|---|---|
| **Grafana Pyroscope** | Acquired by Grafana 2023; Pyroscope 2.0 (2024) at scale | OSS, Go/Python/Ruby/eBPF/Java/.NET/PHP/Rust agents, integrates with Tempo/Loki for "click from metric to flamegraph" |
| **Polar Signals Parca** | Standalone, eBPF-first | Infrastructure-wide, language-agnostic via eBPF, low overhead |
| **Datadog Continuous Profiler** | Datadog | Same-pane integration with APM traces; "Profile-driven flame graphs in the trace view" |
| **Pixie** | New Relic, eBPF | K8s-native auto-instrumentation, traces + profiles from kernel |

Adoption is led by hyperscalers (Google's `gprofiler` predates the OSS wave) and infrastructure-cost-sensitive shops; mid-size teams typically adopt continuous profiling **after** they have logs / metrics / traces working, when the bottleneck shifts from "where is the bug" to "which 5% of CPU costs $40k/mo".

**Read**:
- [Pyroscope 2.0 — Grafana](https://grafana.com/blog/pyroscope-2-0-release/)
- [Profiles, the Missing Pillar — InfoQ](https://www.infoq.com/presentations/profiles-continuous-profiling-observability/)
- [Continuous Profiling in Kubernetes — CNCF](https://www.cncf.io/blog/2022/04/15/continuous-profiling-in-kubernetes-using-pyroscope/)

---

## 6. Exemplars — The Bridge

A Prometheus / OpenMetrics **exemplar** is a single sample attached to a histogram bucket or counter increment that includes external labels (typically `trace_id` and `span_id`). When a request hits the latency histogram and lands in the `>1s` bucket, the SDK attaches the active OTel trace ID as an exemplar. Grafana renders exemplars as dots on the metric graph; clicking a dot opens the trace in Tempo / Jaeger / Honeycomb. Operationally this is the single most important UX feature in modern observability — it removes the "now grep the logs by timestamp" step from every incident. Prometheus 2.26+ supports exemplar storage; OTel SDK ≥1.5 emits them automatically when you scrape via the OTLP/OpenMetrics protocol with the right `Accept` header (`application/openmetrics-text`).

**Read**:
- [Using Prometheus Exemplars to Jump from Metrics to Traces in Grafana](https://vbehar.medium.com/using-prometheus-exemplars-to-jump-from-metrics-to-traces-in-grafana-249e721d4192)
- [Linking Metrics and Traces with Exemplars — Lunatech](https://blog.lunatech.com/posts/2022-01-21-linking-metrics-and-traces-with-exemplars)
- [OpenMetrics 1.0 Spec — Exemplars](https://prometheus.io/docs/specs/om/open_metrics_spec/)

---

## 7. Mature Setups — Concrete Examples

### 7.1 Stripe

Stripe's stack centres on **canonical log lines** (Section 2.6) emitted from every API request, shipped via Kafka into a data warehouse (Redshift/BigQuery class) for ad-hoc analytics, and **Veneur** — their open-source DogStatsD pipeline — for global percentile aggregation across hundreds of API hosts. Veneur exists because StatsD summaries can't be averaged across instances, so Stripe built a tier that aggregates raw observations globally before computing the p99. The combination — wide structured events + globally-aggregated metrics — gives them ad-hoc queryability of every request and accurate fleet-wide percentiles, both with one source of truth.

**Read**:
- [Fast and Flexible Observability with Canonical Log Lines — Stripe](https://stripe.com/blog/canonical-log-lines)
- [Introducing Veneur — Stripe](https://stripe.com/blog/introducing-veneur-high-performance-and-global-aggregation-for-datadog)
- [How Stripe Architected Massive Scale Observability on AWS — AWS Blog](https://aws.amazon.com/blogs/mt/how-stripe-architected-massive-scale-observability-solution-on-aws/)

### 7.2 Uber — Jaeger

Jaeger originated at Uber Engineering in 2015 and was donated to CNCF in 2017 (graduated 2019). Uber moved from a Zipkin-based prototype to a push-based architecture handling thousands of traces per second across hundreds of microservices; the design decisions — UDP agent for low-latency local emission, Kafka buffer, Cassandra/Elasticsearch storage, head sampling at the gateway — are documented in the canonical Uber Engineering blog post and remain the reference architecture for self-hosted tracing.

**Read**:
- [Evolving Distributed Tracing at Uber Engineering](https://www.uber.com/blog/distributed-tracing/)
- [Optimizing Observability with Jaeger, M3, and XYS at Uber](https://www.uber.com/blog/optimizing-observability/)

### 7.3 Netflix — Atlas + Edgar

Netflix's metric system, **Atlas**, is in-memory time-series, multi-dimensional, optimised for streaming alerting (`Atlas Streaming Eval`) — they explicitly engineered against Prometheus-class storage costs at their scale (millions of metrics/sec). Layered on top is **Edgar**, a self-service request-tracing UI that fuses traces, logs, metadata, and Atlas time series into a single "what happened to this request" view. The Netflix lesson, repeatedly stated in their tech blog, is that **distributed tracing alone is not enough**: traces must be augmented with application context (the user's device, the A/B test cohort, the upstream feature flag) and correlated to time-series metrics from Atlas to be useful in incidents. Edgar is essentially Honeycomb-built-in-house, predating Honeycomb's commercial product.

**Read**:
- [Edgar: Solving Mysteries Faster with Observability — Netflix TechBlog](https://netflixtechblog.com/edgar-solving-mysteries-faster-with-observability-e1a76302c71f)
- [Building Netflix's Distributed Tracing Infrastructure](https://netflixtechblog.com/building-netflixs-distributed-tracing-infrastructure-bb856c319304)
- [Lessons from Building Observability Tools at Netflix](https://netflixtechblog.com/lessons-from-building-observability-tools-at-netflix-7cfafed6ab17)

### 7.4 Cloudflare — ClickHouse Logging

Cloudflare's logging pipeline ingests the access logs of the public internet's HTTP/3 edge — millions of requests per second. Their architecture — Logpush → S3 → ClickPipes → ClickHouse cluster (36 nodes, 3× replication, 90M rows/sec ingest) — is the reference for "full-fidelity logging at internet scale, on a budget". A single ClickHouse query across 1.6 quadrillion events returns in <2 seconds. The strategic lesson: **column-stores and cheap object storage have killed the "you must sample" assumption** for any team willing to do infrastructure work. SigNoz, ClickStack, Parseable, and OpenObserve all bring this architecture to mid-size teams as turnkey OSS / SaaS.

**Read**:
- [An Overview of Cloudflare's Logging Pipeline](https://blog.cloudflare.com/an-overview-of-cloudflares-logging-pipeline/)
- [Log Analytics using ClickHouse — Cloudflare](https://blog.cloudflare.com/log-analytics-using-clickhouse/)
- [Cloudflare Quadrillion-Row Scale — ClickHouse Blog](https://clickhouse.com/blog/cloudflare)

---

## Convergent Architecture (May 2026)

Across the studied organisations, the converged "good enough for a 22-service backend" stack is:

1. **Pino (Node) / Zap (Go) → JSON logs to stdout** with mandatory `trace_id`, `span_id`, `service.name`, `tenant_id` fields, plus a Stripe-style **canonical log line** per request.
2. **OpenTelemetry SDK** in every service, auto-instrumenting HTTP / DB / queue clients, propagating **W3C `traceparent`**.
3. **OTel Collector** as DaemonSet (agent) + gateway (StatefulSet), with **tail sampling** keeping 100% of errors and slow traces, sampling 1–10% of the rest.
4. **Prometheus** (or VictoriaMetrics / Mimir) scraping `/metrics`, with **histograms** for latency, **exemplars** linking metric spikes to trace IDs.
5. **Grafana** as the single-pane UI, with Loki (logs), Tempo (traces), Mimir (metrics), Pyroscope (profiles) as the LGTM(P) backend OR Honeycomb / SigNoz as the SaaS alternative.
6. **SLOs codified** with burn-rate alerts, error-budget policy in Markdown, owned by service teams.
7. **Pyroscope** added once the stack is stable, for "which line of code is this regression in".

The frontier of 2025–26 is consolidation: OTel as the universal telemetry pipe, ClickHouse-class column-stores killing the sampling-vs-cost tradeoff, and Honeycomb-style wide-event observability supplanting the strict three-pillar separation.
