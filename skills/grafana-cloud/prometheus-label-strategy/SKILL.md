---
name: prometheus-label-strategy
license: Apache-2.0
description: >
  Expert evaluator for Prometheus label strategy. Audits, designs, and improves label
  schemas using cardinality scoring, access-pattern alignment, static vs. dynamic label
  rules, histogram bucket discipline, instrumentation hygiene, and source-side prevention
  via relabel_config / metric_relabel_configs. Use when the user asks to evaluate, audit,
  design, or improve Prometheus labels — or asks how to prevent high cardinality at the
  source. For post-ingest aggregation, see the adaptive-metrics skill. For "why is my
  Prometheus slow / expensive right now" triage, see prometheus-cardinality-troubleshooter.
---

# Prometheus Label Strategy Evaluator

You are an expert in Prometheus label strategy. When asked to evaluate, audit, design, or improve a Prometheus label schema — or when a user asks how to prevent high cardinality at the source — use this guide to provide structured, actionable advice.

This skill is about **preventing bad labels at ingest** (instrumentation, scrape configuration, relabeling). For post-ingest cost reduction via aggregation rules, route the user to the `adaptive-metrics` skill. For diagnosing an active cardinality fire, route to `prometheus-cardinality-troubleshooter`.

---

## Core Concepts

**Series** are the fundamental unit in Prometheus. Each unique combination of metric name plus label key-value pairs creates a new active series. Too many series = memory pressure, slow queries, ingest pressure, high bill.

**Cardinality** = the number of unique values a label can have. Total series for a metric ≈ the *product* of cardinalities across its labels. A metric with `path` (100 values), `status_code` (10 values), `method` (5 values), and `instance` (50 values) = **250,000 series per metric**. Adding one more high-cardinality label often 10–100×s the count.

**The dual impact rule**: High-cardinality labels hurt on both paths:
- **Ingestion path**: More active series → larger head block, larger WAL, more memory, larger remote_write payloads, higher Grafana Cloud bill (Active Series + DPM)
- **Query path**: PromQL operators (`sum by`, `rate`, joins) must materialize matching series in memory. High cardinality balloons query memory and latency

**Series churn** is the silent killer. If a label value changes frequently (deploy version, pod name, ephemeral IDs), every change creates a *new* series while the old one continues to age out. Daily churn of 100% means you carry roughly 2× the steady-state series count for retention purposes.

**The key question for any proposed label**: "Will queries that use this metric reliably specify or aggregate on this label?" If no → it should NOT be a label.

---

## Label Evaluation Framework

When auditing a label set, assess each label against these criteria.

### Cardinality Scoring

| Label Example | Cardinality | Verdict |
|---|---|---|
| `env` (prod/staging/dev) | 2–5 values | ✅ Good |
| `job` (Prometheus scrape job) | 5–50 values | ✅ Good |
| `cluster`, `region` | Tens | ✅ Good |
| `namespace` (K8s) | Tens–low hundreds | ✅ Acceptable |
| `service`, `workload`, `container` | Tens–hundreds | ✅ Acceptable |
| `instance` (host:port) | Hundreds–low thousands | ⚠️ Evaluate — fine on per-instance metrics, risky on aggregated ones |
| `pod` (K8s) | Thousands + transient = high churn | ❌ Drop at scrape unless required |
| `path` / `route` (HTTP) | Bounded if templated; unbounded if raw URLs | ⚠️ Only with templated values (`/users/:id`) |
| `version`, `image_tag`, `git_sha` | Grows on every deploy → churn | ⚠️ Use sparingly; consider info-metric pattern |
| `user_id`, `request_id`, `trace_id` | Unbounded | ❌ Never as label — use exemplars |
| `customer_id`, `tenant_id` | Often unbounded | ❌ Only acceptable for small fixed tenant counts |
| `error_message`, `query`, `sql` | Unbounded text | ❌ Never |

### Access Pattern Alignment

For each label, ask:
- Do queries on this metric reliably aggregate by or filter on this label?
- Does this label logically segment the metric the way users think about it?
- Would removing this label force users to use exemplars, logs, or traces instead — and would that be acceptable for the rare lookup case?

### Static vs. Dynamic Label Values

- **Static / target labels** (set once per scrape target via `relabel_configs`, e.g., `env=prod`, `cluster=us-east`, `team=payments`) add cardinality proportional to *targets*, not requests. Cheap and high-value. Use freely.
- **Dynamic / sample labels** (emitted by the application per measurement, e.g., `status_code`, `method`, `cache_hit`) multiply cardinality by *value count*. Keep possible values in the single digits or low tens. **The application code is the source of truth — fix it there, not in Prometheus.**

### Consistency Check

- Label *names* consistent across services? (`status` vs `status_code` vs `http_status` produces three separate label families — joins break)
- Label *values* normalized? (`200` vs `"200"`, `GET` vs `get`, `Error` vs `error`)
- Naming convention consistent? Prometheus convention is `snake_case` for both metric and label names
- Same concept, same name across services? (`service` vs `svc` vs `app_name`)

### Histogram Bucket Discipline (critical, often missed)

Every histogram metric multiplies its base cardinality by **(bucket count + 3)** — buckets via `_bucket{le="..."}` plus `_sum`, `_count`, and `_created` (Prometheus 2.39+).

- Default `prometheus.DefBuckets` has 11 buckets → **14× multiplier**
- A histogram with `method`, `path`, `status` already at 1,000 series becomes **14,000 series** after adding histogram cardinality
- **Always trim histogram label cardinality first** — labels matter 14× more on histograms than on counters/gauges
- Consider native histograms (Prometheus 2.40+) which use a single sparse series instead of one-per-bucket — major cardinality reduction for high-resolution latency tracking

### Info-Metric Pattern (for high-churn metadata)

When you want to *know* about a label (e.g., `version`, `git_sha`, `image_tag`) without paying for it on every metric, use an info metric:

```
# A single low-cardinality counter/gauge of value 1, with the metadata attached
app_build_info{app="payment-api", version="2.4.1", git_sha="a1b2c3"} 1
```

Then join at query time:
```promql
sum by (version) (
  rate(http_requests_total{app="payment-api"}[5m])
  * on (app) group_left (version) app_build_info
)
```

The `version` label lives on exactly one series per build, not on every metric.

---

## Evaluation Output Format

When auditing a label set, produce a report in this structure:

```
## Prometheus Label Strategy Audit

### Summary
[1-2 sentence overall assessment — total estimated active series, biggest risks]

### Per-Label Analysis
| Metric Family | Label | Cardinality | Used in Queries? | Verdict | Action |
|---|---|---|---|---|---|
| http_requests_total | path | Unbounded (raw URLs) | Sometimes | ❌ Remove | Template in code: `/users/:id` not `/users/12345` |
| http_requests_total | pod | High + churn | Rarely | ❌ Drop via metric_relabel_configs | Already in target metadata |

### Histogram-Specific Findings
[Highlight any histograms with high label cardinality — these are 14×+ amplified]

### Estimated Impact
- Active series reduction: [X series → Y series]
- DPM reduction: [X DPM → Y DPM]  (samples-per-minute = series × ~6 at 10s scrape)
- Memory impact: [if measurable]

### Recommended Label Set
[Final recommended labels per metric family]

### Implementation Plan
1. [Code changes — instrumentation hygiene]
2. [Scrape config changes — relabel_configs]
3. [Drop-at-scrape changes — metric_relabel_configs]
4. [Recording rules to materialize useful aggregates]
```

---

## Recommended Common Target Labels

These should be set as **target labels** (via `relabel_configs` on the scrape job, NOT emitted by the app) — they're per-target, low cardinality, high query value:

| Label | Purpose | Notes |
|---|---|---|
| `job` | Prometheus scrape job name | Set automatically by Prometheus |
| `instance` | Target endpoint (`host:port`) | Set automatically; rename via `relabel_configs` to a friendlier value if needed |
| `env` | Environment (`prod`, `staging`, `dev`) | Set via static_configs labels or service discovery |
| `cluster` | Multi-cluster differentiation | Critical for federation/Mimir multi-tenant |
| `region` | Geographic region | |
| `team` / `squad` | Ownership — also useful for access control | |
| `service` | Logical service identity | One service may span multiple jobs |

These should **NOT** be re-emitted by the application. If the app emits a `cluster` label, it duplicates the target label and creates collisions / `honor_labels` decisions you don't want to make.

---

## Kubernetes Patterns

### Recommended Labels (from kubernetes_sd_configs)

| Label | Source | Notes |
|---|---|---|
| `namespace` | Pod metadata | Always keep |
| `container` | Pod spec | Low cardinality, useful for multi-container pods |
| `workload` | Derived: `{controller_kind}/{controller_name}` | **Strongly preferred over `pod`** — static, predictable |
| `service` | K8s Service | If scraping via Service |

### Labels to AVOID by Default in Kubernetes

**`pod` label** ⚠️
- Highly transient: rolls every deploy and on every restart
- High cardinality: 100 pods × N metrics = N × 100 series, but on rollouts you carry both old and new pods until they age out
- Almost never the right query dimension — users want *workload*, not *pod instance*
- **Solution**: Keep `workload` as a label; drop `pod` via `metric_relabel_configs`; use exemplars or kube-state-metrics for pod-specific lookups

```yaml
# Drop the pod label from application metrics at scrape time
metric_relabel_configs:
  - regex: pod
    action: labeldrop
```

**`uid` label** ❌
- Completely unbounded (regenerates on every pod recreation)
- No legitimate query use — kept only by accident in default kubernetes_sd configs

**Application-emitted `instance` / `pod` / `node`** ❌
- These should come from target labels, not from the app code
- Drop them at scrape with `metric_relabel_configs` or fix in code

**kube-state-metrics annotation / label propagation** ⚠️
- `kube_pod_labels{label_app_kubernetes_io_*=...}` can carry dozens of metadata labels
- Each unique pod label combination is a new series
- Use kube-state-metrics' `--metric-labels-allowlist` to restrict to the labels you actually query on

---

## Source-Side Prevention: Where to Fix What

There are four levers, in **order of preference**:

### 1. Fix in the Application (best)

Bad labels emitted by the app are the root cause. Examples:
- HTTP paths: use templated routes (`/users/:id`) not raw paths
- Error metrics: use a small enum (`error_type="timeout"`) not the error message string
- User-scoped metrics: don't include `user_id` — use exemplars to point to logs/traces
- Free-form input: never emit user-supplied strings as label values

If you control the code, this is always the right fix. It saves cost on every downstream system (Prometheus, remote_write, Mimir, Grafana Cloud).

### 2. `relabel_configs` (target-time relabeling)

Runs *before* the scrape. Used to:
- Set target labels (`env`, `cluster`, `team`) on discovered targets
- Drop entire targets you don't want to scrape
- Rewrite `instance` to a friendly value
- Add identity from service discovery metadata

```yaml
scrape_configs:
  - job_name: my-app
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Set workload from controller metadata
      - source_labels: [__meta_kubernetes_pod_controller_kind, __meta_kubernetes_pod_controller_name]
        target_label: workload
        separator: /
      # Set env from a pod label
      - source_labels: [__meta_kubernetes_pod_label_env]
        target_label: env
      # Only scrape pods explicitly opted in
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
```

### 3. `metric_relabel_configs` (scrape-time relabeling)

Runs *after* the scrape, *before* storage. Used to:
- Drop high-cardinality labels the app shouldn't have emitted
- Drop entire metrics you don't want
- Rewrite label values for normalization

```yaml
scrape_configs:
  - job_name: my-app
    metric_relabel_configs:
      # Drop the pod label from every metric
      - regex: pod
        action: labeldrop

      # Drop a specific high-cardinality metric entirely
      - source_labels: [__name__]
        regex: my_app_request_details
        action: drop

      # Normalize status_code to a class (200, 300, 400, 500)
      - source_labels: [status_code]
        regex: (\d)\d\d
        target_label: status_code
        replacement: ${1}xx
```

This is the **emergency stop** for bad labels. Use when you can't fix the app immediately.

### 4. Recording Rules (query-time cardinality reduction)

Pre-aggregate expensive series into a lower-cardinality recorded series. Stored at the same data point density but with far fewer series.

```yaml
groups:
  - name: http-requests-aggregates
    interval: 30s
    rules:
      # Drop pod/instance dimension; keep only service-level rollup
      - record: service:http_requests:rate5m
        expr: sum by (service, env, cluster, status_code) (rate(http_requests_total[5m]))
```

Queries that target the rollup are dramatically cheaper. The raw series still exist — recording rules don't reduce ingest cost (use Adaptive Metrics or `metric_relabel_configs` for that). They reduce query cost.

---

## Instrumentation Hygiene (for app developers)

If the user is *writing* instrumentation code, these are the rules:

| Rule | Why |
|---|---|
| Never use unbounded user input as a label value | `email`, `user_id`, `query string`, `error message` — they're the #1 cardinality bug |
| Template HTTP paths before recording | `/users/{id}` not `/users/12345`. Most frameworks do this via routing metadata |
| Bound error labels via small enums | `error_type="timeout"` not `error="connection to db-shard-7 timed out at 14:32:09"` |
| Don't put `version` / `git_sha` / `build_id` on every metric | Use an info metric and join at query time |
| Don't emit `pod` / `node` / `host` from code | Comes from scrape targets — duplicating creates collisions |
| Avoid dynamically constructed label *names* (keys) | `metric{[user]=1}` cannot be bounded — use a fixed key |
| Use histograms sparingly and trim labels first | 14× cardinality amplification |
| Prefer exemplars over labels for trace correlation | Exemplars carry `trace_id` without inflating cardinality |

### Exemplars (the escape hatch)

Exemplars attach a `trace_id` (or any key-value pair) to specific samples *without* making it a label dimension. The ideal home for high-cardinality correlation data.

Requires OpenMetrics format, Prometheus 2.26+, scrape config:
```yaml
scrape_configs:
  - job_name: my-app
    enable_protobuf_negotiation: true
    # Or for text-format:
    follow_redirects: true
```

And on the Prometheus server:
```yaml
storage:
  exemplars:
    max_exemplars: 100000
```

Use exemplars for:
- `trace_id` correlation (Tempo, Jaeger)
- `request_id` for specific debug lookups
- Any sparse "useful when you need it" key

Query exemplars via Grafana's exemplars-on-graph feature, not via PromQL aggregation.

---

## The 80/20 Rule

The most impactful improvements almost always come from these five changes:

1. **Drop unbounded labels at the app layer** — `path` (untemplated), `user_id`, `error_message`. Single biggest win.
2. **Trim histogram label cardinality before anything else** — 14× amplification on every histogram.
3. **Drop `pod` from application metrics** — keep `workload` instead. Eliminates churn, big stream-count reduction.
4. **Use info metrics for `version` / `git_sha` / `image_tag`** — eliminates deploy-driven churn.
5. **Set target labels via `relabel_configs`, not app code** — `env`, `cluster`, `team`, `service` should never be emitted by the application.

Focus on these before anything else.

---

## Labels to Avoid — Quick Reference

| Label | Why | Alternative |
|---|---|---|
| `user_id`, `customer_id` (large tenant base) | Unbounded | Exemplars; aggregate by `tenant_tier` |
| `request_id`, `trace_id` | Unbounded | Exemplars |
| `path` / `route` (raw URLs) | Unbounded | Template in code: `/users/:id` |
| `error_message`, `query`, `sql` | Unbounded text | Bounded `error_type` enum |
| `version`, `git_sha`, `image_tag` (on every metric) | Churn on every deploy | Info metric pattern |
| `pod` (on app metrics) | Transient + high cardinality | `workload`; exemplars for pod-specific debug |
| `uid` (K8s) | Unbounded; regenerates on restart | Drop entirely |
| Application-emitted `instance`, `node`, `host` | Should come from scrape target | Drop via `metric_relabel_configs` |
| Dynamically-named label keys | Cannot be bounded | Use fixed keys with bounded values |
| Raw `status_code` on histograms | 14× amplification | Bucket to `status_class` (`2xx`, `4xx`, `5xx`) |

---

## When to Route Elsewhere

- **"Reduce my Grafana Cloud bill"** → also engage `adaptive-metrics` skill (post-ingest aggregation rules)
- **"Which metrics are driving my DPM?"** → engage `dpm-finder` skill
- **"My Prometheus is OOMing / scraping is failing right now"** → engage `prometheus-cardinality-troubleshooter` skill
- **"How do I write the query to find the bad metric?"** → engage `promql` skill
- **"How do I configure relabel rules in Alloy?"** → engage `alloy` skill

This skill's lane is **strategy and design**. Other skills own **diagnosis** and **operational remediation**.
