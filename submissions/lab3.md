# Lab 3 — Monitoring, Observability and SLOs

## Environment

Docker 29.2.1 and Docker Compose v5.0.2 were used on Windows with Docker Desktop.

## Monitoring Configuration

Prometheus uses a 15-second scrape and evaluation interval. It scrapes `gateway:8080`, `events:8081`, and `payments:8082`; `rules.yml` is mounted read-only.

## Running Services

```text
app-events-1 Up
app-gateway-1 Up
app-grafana-1 Up
app-payments-1 Up
app-postgres-1 Up (healthy)
app-prometheus-1 Up
app-redis-1 Up (healthy)
```

## Prometheus Targets

```text
events    up  http://events:8081/metrics
gateway   up  http://gateway:8080/metrics
payments  up  http://payments:8082/metrics
```

## Golden Signals Dashboard

The committed dashboard contains p50, p95, and p99 latency queries using `histogram_quantile()`, a gauge for `events_db_pool_size`, and an SLO availability gauge. Latency is displayed in seconds; saturation is configured for 0–10 with yellow at 7 and red at 9.

## SLI, SLO and Error Budget

Availability SLI is the proportion of gateway responses that are not 5xx; its SLO is 99.5% for seven days. Latency SLI is the proportion of gateway requests below 500 ms; its SLO is 95%.

`1000 requests/day × 7 days = 7000 requests/week`; `7000 × 0.5% = 35` allowed failed requests per week.

## Recording Rules

```text
gateway:sli_availability:ratio_rate5m = ok
gateway:sli_latency_500ms:ratio_rate5m = ok
gateway:error_budget_burn_rate:ratio_rate5m = ok
```

The availability and latency rules aggregate numerator and denominator labels consistently before division. Burn rate is `(1 - availability) / (1 - 0.995)`.

## Conclusions

Prometheus successfully scrapes all application services and evaluates all three recording rules. The dashboard is provisioned from the committed JSON file rather than a UI-only change.
