# Module 22: Observability Correlation (Metrics, Logs and Traces)

## Overview

This module is the payoff of the three-module observability build in Modules 19-21: Grafana Loki for logs, Grafana Tempo for traces, and Grafana Mimir for metrics are now all live in the same **observability cluster**, but so far each has only been explored on its own. This module's four lessons wire the three pillars together, so that a trace can be followed to the logs it produced, a service graph can be derived directly from trace data, and metrics, logs, and traces can all sit side by side in one Grafana dashboard. As the instructor frames it, this is a short, low-drama module — "aula mais soft" — that touches no new infrastructure at all, only Grafana data source configuration and one Helm value flip on Tempo.

All 4 lessons continue in the **same repository and cluster** as Modules 19-21 — `linuxtips-eks-observability-cluster`, branch `main` — no new repository is involved this time, and no new Terraform resource is created. Every piece of ground truth for this module lives inside `locals.tf`'s existing `grafana` and `tempo` blocks, both of which were already quoted in full in earlier modules as the repository's cumulative final state.

> **Note:** the `tempo` block's `metricsGenerator`/`overrides`/`global_overrides` keys, and the `grafana` block's `tracesToMetrics`/`serviceMap`/`nodeGraph` datasource fields, were already visible in Module 20's README, which flagged them as forward-looking scaffolding and attributed them to "Module 21." That attribution turns out to be one module early: Module 21 only builds and activates the `Mimir` data source itself. The metrics-generator setup and the trace-to-metrics/service-graph wiring that actually *use* that data source are this module's content, not Module 21's.

> **Note:** this module's lessons reference per-language code samples (in the course's own "extra materials," not part of either Git repository) for instrumenting an application to log its active trace ID. Since no such file exists in either repository, this README describes the requirement generically rather than inventing a code sample or a specific language/framework.

> **Note:** Lesson 4's dashboard is built live in Grafana's UI during the recording, and the instructor mentions sharing a dashboard JSON export as course material — but no such export exists in the `linuxtips-eks-observability-cluster` repository. That lesson's dashboard content is presented here as a reconstruction of what the transcript walks through, not a quote from a real file.

## Table of Contents

- [Lesson 1: Introduction to Observability Correlation](#lesson-1-introduction-to-observability-correlation)
  - [1. One Grafana Front End, Three Correlated Pillars](#1-one-grafana-front-end-three-correlated-pillars)
- [Lesson 2: Correlating Traces and Metrics with the Tempo Metrics Generator](#lesson-2-correlating-traces-and-metrics-with-the-tempo-metrics-generator)
  - [1. What the Metrics Generator Does](#1-what-the-metrics-generator-does)
  - [2. Enabling the Metrics Generator and Remote-Writing to Mimir](#2-enabling-the-metrics-generator-and-remote-writing-to-mimir)
  - [3. Wiring the Tempo Datasource for Service Graph and Traces-to-Metrics](#3-wiring-the-tempo-datasource-for-service-graph-and-traces-to-metrics)
  - [4. Exploring the Service Graph](#4-exploring-the-service-graph)
  - [5. Building a Service Graph Panel and an Outlier Traces Table](#5-building-a-service-graph-panel-and-an-outlier-traces-table)
- [Lesson 3: Correlating Traces and Logs](#lesson-3-correlating-traces-and-logs)
  - [1. Instrumenting Application Logs with a Trace ID](#1-instrumenting-application-logs-with-a-trace-id)
  - [2. Wiring the Loki Datasource's Derived Fields](#2-wiring-the-loki-datasources-derived-fields)
  - [3. Navigating Between Logs and Traces](#3-navigating-between-logs-and-traces)
  - [4. Adding a Logs Panel to the Dashboard](#4-adding-a-logs-panel-to-the-dashboard)
- [Lesson 4: Building a Correlated RED Dashboard](#lesson-4-building-a-correlated-red-dashboard)
  - [1. The RED Method](#1-the-red-method)
  - [2. Request Rate, Per-Cluster Breakdown](#2-request-rate-per-cluster-breakdown)
  - [3. Availability and Error Rate](#3-availability-and-error-rate)
  - [4. Response Time Percentiles](#4-response-time-percentiles)
  - [5. Single Pane of Glass](#5-single-pane-of-glass)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to Observability Correlation

## 1. One Grafana Front End, Three Correlated Pillars

Modules 19-21 built each observability pillar in isolation: traces into Tempo, logs into Loki, metrics into Mimir. This module's goal is to make those three stacks work together — to be able to start from a trace and jump to the log lines it produced, or from a trace to a service map showing what called what, all inside the same Grafana front end. The lesson frames this as the actual definition of "doing observability": not just collecting the three signals, but being able to move between them without leaving the tool.

Concretely, two correlations are built this module: **traces to a service map**, so that communication between services and where it's degrading becomes visible at a glance, and **traces to logs** (and back), so that a specific span can be traced down to the log lines it generated. Unlike the previous three modules, none of this requires new infrastructure — only Grafana data source configuration, plus one Helm value that turns on a feature already sitting inside the `tempo-distributed` chart.

---

# Lesson 2: Correlating Traces and Metrics with the Tempo Metrics Generator

## 1. What the Metrics Generator Does

Tempo ships an optional component called the **metrics generator**, disabled by default in the `tempo-distributed` chart. Instead of only storing incoming spans, the metrics generator inspects every span passing through the ingestion path and derives Prometheus-style metrics from them — essentially running a small Prometheus-like process inside Tempo itself. Those derived metrics are then remote-written out to a real metrics backend, the same remote-write mechanism used by the per-cluster Prometheus servers in Module 21.

## 2. Enabling the Metrics Generator and Remote-Writing to Mimir

The metrics generator is configured directly in `locals.tf`'s `tempo` block, alongside the storage/component settings already covered in Module 20:

```yaml
# locals.tf (tempo.values)
metricsGenerator:
  enabled: true
  registry:
    external_labels:
      source: tempo
  config: 
    storage:
      remote_write: 
      - url: "http://mimir-nginx.mimir.svc.cluster.local:80/api/v1/push"
        send_exemplars: true

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks]

global_overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks]
```

`registry.external_labels.source: tempo` tags every metric the generator produces, distinguishing it from metrics that arrive at Mimir via a cluster's own Prometheus. `config.storage.remote_write` points at the exact same `mimir-nginx.mimir.svc.cluster.local:80` Service used by the Mimir Grafana data source — but because Tempo and Mimir run in the same observability cluster, this remote write goes straight to Mimir's internal Kubernetes Service DNS. That's a direct contrast with Module 21's per-workload-cluster Prometheus servers, which had to remote-write to Mimir's *external* Route53/ALB address instead, since they run in separate clusters that can't resolve an internal Service name in the observability cluster. `send_exemplars: true` attaches exemplars to the remote-written metrics — sample-level links back to the specific trace ID that produced a given data point, which is what makes "jump from a metric spike to the trace that caused it" possible later.

The `processors` list controls which kinds of metrics the generator derives from spans:

- **`service-graphs`** — builds the edges and health/latency data behind Grafana's node/service graph visualization, based on caller→callee relationships observed in span data.
- **`span-metrics`** — derives RED-style (rate, errors, duration) metrics directly from span data, without needing a separate scrape target.
- **`local-blocks`** — powers TraceQL metrics, letting ad hoc rate/quantile queries run directly against trace data through the Tempo data source.

`overrides` and `global_overrides` set the exact same `processors` list under two different keys. The lesson explains this duplication as a backward/forward-compatibility measure: the key that controls per-tenant processor overrides was renamed relatively recently in the `tempo-distributed` chart, so keeping both keys populated with the same value covers whichever key a given chart version actually reads.

## 3. Wiring the Tempo Datasource for Service Graph and Traces-to-Metrics

With the metrics generator remote-writing into Mimir, the `Tempo` Grafana data source's `jsonData` — already quoted in full back in Module 20, where it referenced a `Mimir` data source that didn't exist yet — is now fully functional:

```yaml
# locals.tf (grafana.values → datasources, Tempo entry)
- name: Tempo
  type: tempo
  access: proxy
  url: http://tempo-gateway.tempo.svc.cluster.local
  basicAuth: false
  jsonData:
    tracesToMetrics:
      datasourceUid: 'Mimir'
    serviceMap:
      datasourceUid: 'Mimir'
    nodeGraph:
      enabled: true
    tracesToLogs:
      datasourceUid: 'Loki'
```

`serviceMap.datasourceUid: 'Mimir'` tells Grafana's Tempo data source where to query the `service-graphs` processor's metrics from, so it can render a service map. `nodeGraph.enabled: true` turns on the node graph visualization type in Explore and in dashboard panels — this is what actually renders that service map as an interactive graph. `tracesToMetrics.datasourceUid: 'Mimir'` is what lets a trace view jump to related metrics in Mimir. `tracesToLogs.datasourceUid: 'Loki'` was already functional since Module 20; it's the trace side of the correlation Lesson 3 builds the other half of.

## 4. Exploring the Service Graph

After a `terraform apply`, Tempo's pods recycle to pick up the new metrics-generator configuration, and a new `metrics-generator` Deployment appears in the `tempo` namespace. Its logs confirm it's collecting: a startup message showing it's active, followed immediately by successful remote-write pushes to Mimir.

From Grafana's Explore view, selecting the `Tempo` data source and switching to **Service Graph** now renders a live graph of the `health-api` demo's call chain: a client hitting `health-api` at roughly 0.5 requests/second and ~59ms response time, which in turn calls `recommendation-service`, which fans out to `water`, `proteins`, `calories`, `imc`, and `bmr`. The same view exposes the underlying `traces_service_graph_*` metrics that Prometheus-style querying can consume directly, plus a live, continuously-updating trace list.

## 5. Building a Service Graph Panel and an Outlier Traces Table

> **Note:** this section reconstructs a live Grafana UI walkthrough with no exported dashboard JSON in either repository. It's presented here as a description of what the lesson builds, not a quote from a real file.

The lesson creates a new dashboard and adds a **Node Graph** panel driven by the `Tempo` data source, filtered to the `health-api`/`nutrition-cal-service` span name, labeled "API Serv" — giving a permanent, always-visible view of which downstream calls are failing and when. Alongside it, a second panel queries the same span name in table format, listing `traceID`/`service` columns, filtered down to spans with a duration greater than 500ms — an "outlier traces" view.

To generate an actual outlier, the lesson deletes the `protorpc`-labeled pods in one of the workload clusters (`linuxtips-cluster-01`), forcing the `health-api` demo's gRPC calls to fail and retry while the pod restarts. That produces a real outlier trace lasting around 16 seconds, dominated by internal retries before recovering — the lesson is explicit that this is an artificially-induced outlier for demonstration, not representative of the application's normal latency. Clicking into that trace from the outlier table shows the full end-to-end trace, including exactly which downstream service failed and the resulting upstream error message. The panels are grouped under a new dashboard row labeled "Global SLIs."

---

# Lesson 3: Correlating Traces and Logs

## 1. Instrumenting Application Logs with a Trace ID

Correlating a log line back to the trace that produced it is simple to configure in Grafana, but it depends on the application itself logging its active trace ID on every log line — Grafana can't infer this correlation on its own. In the `health-api` demo, the `nutrition-cal-service` component already does this: each log entry includes a `traceID` field, populated from the active OpenTelemetry span. The lesson notes that the course's extra materials include per-language guidance for adding this instrumentation to an application that doesn't already emit it.

> **Note:** since no such per-language sample file exists in either repository, this requirement is described here generically — as "log the active trace ID as a field on every log line" — rather than naming a specific language or library.

## 2. Wiring the Loki Datasource's Derived Fields

Once an application logs its trace ID, Grafana needs to know how to find that ID inside a log line and which data source to link it to. That's configured as a **derived field** on the `Loki` data source — already quoted in full back in Module 19, where it referenced a `Tempo` data source that didn't yet exist:

```yaml
# locals.tf (grafana.values → datasources, Loki entry)
- name: Loki
  type: loki
  access: proxy
  url: http://loki-gateway.loki.svc.cluster.local
  isDefault: false
  jsonData:
    maxLines: 1000
    derivedFields:
    - datasourceName: Tempo
      datasourceUid: Tempo
      matcherRegex: '\\"traceID\\":\\"([^\\"]+)\\"'
      name: traceID
      url: $$${__value.raw}
```

`matcherRegex` runs against every log line Loki returns, capturing whatever sits inside a `"traceID":"..."` JSON field — matching the exact field name the `nutrition-cal-service` component logs. `datasourceName`/`datasourceUid: Tempo` tells Grafana which data source the captured value should link to, and `url: $${__value.raw}` passes the captured trace ID straight through as the trace lookup value, with no URL template needed since Tempo's own data source already knows how to resolve a trace ID.

## 3. Navigating Between Logs and Traces

After a `terraform apply`, filtering Loki logs by `service_name="health-api"` (or another instrumented service) and expanding a log line now shows a **Tempo** button next to the captured `traceID` field. Clicking it opens the full trace that produced that specific log line, end to end.

The correlation also works in the opposite direction: from a trace view in Tempo, a **Logs for span** button attempts the reverse lookup — finding log lines associated with a given span. In the lesson's demo this sometimes returns no results for a given span (a normal outcome when that particular service/log line wasn't captured, or the correlated logs didn't include a matching trace ID), which the instructor treats as an expected edge case, not a bug: the two-way correlation exists, but its coverage still depends on which services actually emit a `traceID` field on their logs.

## 4. Adding a Logs Panel to the Dashboard

> **Note:** continues the same reconstructed dashboard from Lesson 2 §5 — no exported JSON survives in either repository.

A new panel is added to the same example dashboard, using the `Loki` data source, filtered to `namespace="nutrition"` with `container != "istio-proxy"` (excluding the Envoy sidecar's own logs), rendered as a **Logs** visualization titled "health API logs." Combined with the service graph and outlier-traces panels from Lesson 2, the dashboard now surfaces traces, a service map, and application logs together — the first working sketch of the "single pane of glass" the module is building toward.

---

# Lesson 4: Building a Correlated RED Dashboard

> **Note:** this entire lesson reconstructs a live Grafana UI walkthrough. The instructor mentions exporting the finished dashboard as JSON for the course's extra materials, but no such file exists in either repository — everything below describes what the transcript builds, not a quote from a real file.

## 1. The RED Method

With traces, logs, and a service graph correlated, the lesson adds one more dimension: metrics, using the **RED method** (Rate, Errors, Duration) as a simple, well-known pattern for summarizing a service's health. The metrics come from Istio's own request metric, `istio_requests_total`, and its companion histogram `istio_request_duration_milliseconds_bucket`, scraped via the Envoy sidecar `PodMonitor` configuration set up back in Module 21 — the same metrics already used in the cross-pillar dashboard that closed that module.

## 2. Request Rate, Per-Cluster Breakdown

The **Rate** panel sums `istio_requests_total`, filtered by `destination_workload` to isolate the `health-api` demo traffic and by `source_workload` to exclude the Envoy sidecar's own internal requests, over a 1-minute rate window, displayed as requests per second. The panel is built twice: once as a time series (with the course's purple color palette) and once as a stacked bar chart split **by cluster**, so the contribution of `linuxtips-cluster-01` versus `linuxtips-cluster-02` to total traffic is visible side by side — the same `cluster` external label introduced by each workload cluster's Prometheus in Module 21 is what makes this split possible.

## 3. Availability and Error Rate

Two **stat** panels summarize the **Errors** dimension. Availability divides the count of non-error `istio_requests_total` responses by the total request count, displayed as a percentage — 100% during normal operation. A second panel inverts the same idea, summing requests where `response_code` falls outside the `2xx` range (again filtered by differing `source_workload`/`destination_workload` to isolate real cross-service calls) over a 1-minute range, to surface the raw error count.

## 4. Response Time Percentiles

The **Duration** dimension uses `histogram_quantile` against `istio_request_duration_milliseconds_bucket`, plotted as three separate queries for the P50, P90, and P99 percentiles, each labeled accordingly. Values are divided by 1000 and displayed in seconds rather than milliseconds, matching the convention the lesson notes is also used in Istio's own reference dashboards.

## 5. Single Pane of Glass

With rate, errors, duration, the service graph, outlier traces, and application logs all on one dashboard, the lesson closes by naming the destination pattern explicitly: a **single pane of glass**. The scenario it walks through is a response-time spike visible over a 12-hour window with no corresponding errors — which on its own tells an operator very little, until the correlated traces panel identifies the specific outlier trace, which in turn shows that `recommendation-service`'s call to `proteins` was the point of failure. That end-to-end narrowing — metric anomaly → outlier trace → failing downstream call — is presented as the actual goal of building this dashboard, and of the three-module observability effort as a whole.

---

## Key Takeaways

- This module adds no new infrastructure: every change lives inside `locals.tf`'s already-existing `grafana` and `tempo` blocks, both already fully quoted in earlier modules as the repository's cumulative final state.
- Correlation depends on two features enabled this module: Tempo's **metrics generator** (deriving service-graph and span metrics from trace data, remote-written to Mimir) and Loki's **derived fields** (regex-extracting an application-logged trace ID and linking it to Tempo).
- The metrics-generator's remote write to Mimir uses the **internal** `mimir-nginx.mimir.svc.cluster.local` Service address, since Tempo and Mimir share the observability cluster — the mirror image of Module 21's per-workload-cluster Prometheus servers, which had to use Mimir's **external** address instead.
- Log-to-trace correlation is not automatic: it depends on the application itself logging its active trace ID on every line, an application-level instrumentation step outside this repository's own Terraform/Helm scope.
- `overrides` and `global_overrides` both carry the same `metrics_generator.processors` value, a deliberate hedge against a chart-version-dependent key rename rather than redundant configuration.
- Correlation works in both directions where supported: traces link to their logs and back, and a service graph derived purely from trace data now sits alongside Loki logs and Mimir metrics in one dashboard.
- The dashboard content built in Lessons 2-4 exists only as a live Grafana UI walkthrough in the transcripts — no exported JSON survives in either repository, so it's documented here as a reconstruction rather than a quoted file.
- The "single pane of glass" built across this module is presented as the payoff of the entire Modules 19-22 observability arc: a metric anomaly can be narrowed down, through a correlated dashboard alone, to the exact trace and failing downstream call that caused it.
