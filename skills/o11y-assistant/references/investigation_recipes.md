# Investigation Recipe Patterns

> Loaded by SKILL.md at Step 4 when constructing queries for common incident types.
> These are starting patterns — always discover actual label names before using.
> Replace `<svc>` with the service name from Session State's service mapping.

---

## Error Rate Investigation

| Backend | Query | Notes |
|---------|-------|-------|
| **Prometheus** | `rate(http_requests_total{job="<svc>", status=~"5.."}[5m]) / rate(http_requests_total{job="<svc>"}[5m])` | Error ratio. If `http_requests_total` doesn't exist, try `http_server_requests_seconds_count` |
| **Prometheus** | `sum(increase(http_requests_total{job="<svc>", status=~"5.."}[15m]))` | Absolute error count in window |
| **Tempo** | `{ resource.service.name="<svc>" && status=error }` (search) | Find individual error traces |
| **Tempo** | `{ resource.service.name="<svc>" } | rate()` + `{ resource.service.name="<svc>" && status=error } | rate()` (metrics) | Compute error rate from traces |
| **Loki** | `{service_name="<svc>"} \|= "error" \| logfmt \| level="error"` | Error logs. Adapt parser to log format (logfmt/json/pattern) |

---

## Latency Investigation

| Backend | Query | Notes |
|---------|-------|-------|
| **Prometheus** | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="<svc>"}[5m])) by (le))` | p99 latency. Check for `_bucket` suffix. Also try `_duration_milliseconds_bucket` |
| **Prometheus** | `histogram_quantile(0.95, ...)` vs `histogram_quantile(0.50, ...)` | Compare p95 to p50. If p99 ≫ p50, long tail problem |
| **Tempo** | `{ resource.service.name="<svc>" && duration > 1s }` (search) | Find slow traces. Adjust threshold to symptom |
| **Tempo** | `{ resource.service.name="<svc>" } \| quantile_over_time(duration, 0.99) with(sample=true)` (metrics-instant) | p99 duration from traces |
| **Loki** | `{service_name="<svc>"} \| json \| latency > 1000` | If app logs latency. Adapt field name |

---

## Resource Exhaustion (CPU / Memory / Disk)

| Backend | Query | Notes |
|---------|-------|-------|
| **Prometheus** | `container_memory_working_set_bytes{pod=~"<svc>.*"} / container_spec_memory_limit_bytes{pod=~"<svc>.*"}` | Memory utilization ratio. >85% = pressure |
| **Prometheus** | `rate(container_cpu_usage_seconds_total{pod=~"<svc>.*"}[5m])` | CPU usage in cores |
| **Prometheus** | `node_filesystem_avail_bytes{mountpoint="/"}` | Disk space. Check all mountpoints |
| **Prometheus** | `kube_pod_container_status_restarts_total{pod=~"<svc>.*"}` | OOM restarts. Combine with `kube_pod_container_status_last_terminated_reason` |
| **Loki** | `{pod=~"<svc>.*"} \|= "OOMKilled" or \|= "Out of memory"` | OOM evidence in logs |

---

## Connection / Dependency Failures

| Backend | Query | Notes |
|---------|-------|-------|
| **Prometheus** | `sum(rate(http_client_requests_total{job="<svc>", status=~"5.."}[5m])) by (target)` | Outbound errors by downstream dependency |
| **Prometheus** | `<svc>_pool_active_connections / <svc>_pool_max_connections` | Connection pool saturation. Metric name varies |
| **Tempo** | `{ resource.service.name="<svc>" && status=error && span.http.status_code >= 500 }` (search) | Error spans with HTTP status |
| **Tempo** | `{ resource.service.name="<svc>" } >> { status=error }` (search) | Find downstream error spans (⚠️ expensive structural operator) |
| **Loki** | `{service_name="<svc>"} \|= "connection refused" or \|= "timeout" or \|= "pool exhausted"` | Connection failure logs |

---

## Deployment Correlation

| Backend | Query | Notes |
|---------|-------|-------|
| **Grafana** | `get_annotations(From=<start_ms>, To=<end_ms>)` | Find deployment markers in investigation window |
| **Prometheus** | `changes(kube_deployment_status_observed_generation{deployment="<svc>"}[1h])` | Detect rollout events |
| **Prometheus** | Error rate BEFORE vs AFTER annotation timestamp | Compare: `rate(...[5m] offset 30m)` vs `rate(...[5m])` |
| **Loki** | `{namespace="argocd"} \|= "<svc>"` or `{job="deploy-bot"} \|= "<svc>"` | Deployment pipeline logs |

---

## Saturation / Traffic Spike

| Backend | Query | Notes |
|---------|-------|-------|
| **Prometheus** | `sum(rate(http_requests_total{job="<svc>"}[5m]))` | Current RPS. Compare to baseline |
| **Prometheus** | `sum(rate(http_requests_total{job="<svc>"}[5m])) / sum(rate(http_requests_total{job="<svc>"}[5m] offset 1d))` | Today vs yesterday ratio. >2x = spike |
| **Prometheus** | `avg(kube_deployment_status_replicas{deployment="<svc>"})` vs `avg(kube_hpa_status_desired_replicas{hpa="<svc>"})` | Autoscaling saturation |
| **Tempo** | `{ resource.service.name="<svc>" } \| rate()` (metrics-range) | RPS from traces |
