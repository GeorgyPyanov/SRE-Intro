# Lab 7 — Progressive Delivery and Canary Deployments

## Environment

Work was performed on 2026-10-04 in the existing `quickticket` k3d cluster.

```text
Docker: 29.2.1 / 29.2.1
kubectl client: v1.34.1
k3d: v5.9.0 (k3s v1.35.5-k3s1)
Helm: v4.3.0
Argo CD CLI: v3.5.3
kubectl-argo-rollouts: v1.10.0+d90700a (windows/amd64)
```

The Argo Rollouts controller was installed in `argo-rollouts` and was Running. The CRDs `rollouts.argoproj.io`, `analysisruns.argoproj.io`, and `analysistemplates.argoproj.io` were present. The initial official manifest hit the Kubernetes annotation-size limit for two CRDs; a server-side apply created them safely.

## Rollout Manifest and Baseline

`k8s/gateway.yaml` changes the gateway from a Deployment to an `argoproj.io/v1alpha1` Rollout. It keeps the gateway Service selector, GHCR image, pull secret, dependencies, resources, and `/health` probes. It uses five replicas; `APP_VERSION` changes pod-template hash without rebuilding the image.

Argo CD was switched from `feature/lab5` to `feature/lab7` after the first branch push. It synced revision `bd9c2ffed9019852c5ed62507c30bb4324e9954c`; the old Deployment was absent and the Rollout had five Ready pods.

## Manual Canary

Revision 2 (`APP_VERSION=v2-canary`) paused at 20%:

```text
Status: Paused / CanaryPauseStep
SetWeight: 20; ActualWeight: 20
Updated: 1; Ready: 5; Available: 5
stable RS gateway-669f94f69d: 4 pods
canary RS gateway-cbd89f6c8: 1 pod
```

The provided in-cluster load generator was applied. Logs were counted only for the same 30-second interval:

| Pod | Hash | Role | Requests | Share |
|---|---|---|---:|---:|
| gateway-669f94f69d-jr6zj | 669f94f69d | stable | 56 | 18.42% |
| gateway-669f94f69d-lz8xv | 669f94f69d | stable | 60 | 19.74% |
| gateway-669f94f69d-v7m6j | 669f94f69d | stable | 52 | 17.11% |
| gateway-669f94f69d-vtwgj | 669f94f69d | stable | 65 | 21.38% |
| gateway-cbd89f6c8-mld5r | cbd89f6c8 | canary | 71 | 23.36% |

`kubectl argo rollouts promote gateway` promoted the revision. The final result was Healthy with five Ready pods and ActualWeight 100.

## Manual Abort

Candidate revision 3 used only `APP_VERSION=v3-bad`; no application defect was claimed. It paused at 20% with one candidate and four stable pods. `kubectl argo rollouts abort gateway` was run at `2026-10-04T11:41:41.4262349Z`.

```text
11:41:43.1927438Z canary row still present, 4 Ready stable rows
11:41:44.3704959Z candidate row absent, 4 Ready stable rows
11:41:50.2051164Z candidate absent, 5 Ready stable rows
observable abort-to-five-Ready-stable time: 8.779 seconds
```

The Rollout correctly showed `Degraded / RolloutAborted` while the stable revision served all traffic. This is faster than the Lab 5 Git revert observation of about 18.6 seconds because Rollouts scales down the candidate in-cluster, while Git revert also waits for push, Argo CD detection, synchronization, and a replacement pod.

## Multi-Step Canary

The strategy in the final manifest is 20%/60s, 40%/60s, 60%/60s, 80%/30s, then 100%.

| UTC timestamp | Weight | Stable replicas | Canary replicas | Ready pods | Result |
|---|---:|---:|---:|---:|---|
| 11:42:42.941661Z | 20 | 4 | 1 | 5 | Paused, all Ready |
| 11:44:26.003437Z | 40 | 3 | 2 | 5 | Paused, all Ready |
| 11:45:13.691852Z | 60 | 2 | 3 | 5 | Paused, all Ready |
| 11:56:11.070166Z | 80 | 1 | 4 | 5 | Paused, all Ready |

The in-cluster load generator remained active, so request traffic continued while Rollout changed ReplicaSet sizes. I would run an automated abort at the first 20% analysis stage: it limits blast radius to one pod while still providing enough traffic for a measurement.

## Prometheus Analysis Bonus

The supplied `labs/lab7/prometheus.yaml` and `analysis-template.yaml` were applied. In-cluster Prometheus was Running and discovered each gateway pod with the relabeled `rs_hash`:

```text
gateway-7cff4c48b9-lj9fl  rs_hash=7cff4c48b9  health=up
gateway-7cff4c48b9-zr7cs  rs_hash=7cff4c48b9  health=up
gateway-7cff4c48b9-jvgqd  rs_hash=7cff4c48b9  health=up
gateway-7cff4c48b9-zwj7v  rs_hash=7cff4c48b9  health=up
gateway-7cff4c48b9-29qbk  rs_hash=7cff4c48b9  health=up
sum(rate(gateway_requests_total[30s])) = 0.6388933333333333
```

The AnalysisTemplate `gateway-error-rate` filters the canary by `{{args.canary-hash}}`, has `initialDelay: 60s`, three 20-second measurements, success condition `< 0.05`, and failure limit 1.

### Successful AnalysisRun

The first attempted good analysis failed because the cluster restart had left PostgreSQL without the `events` table; logs showed `relation "events" does not exist`. I seeded the existing database using `app/seed.sql` (`INSERT 0 5`) and retried the same good revision. The retried AnalysisRun was successful:

```text
name: gateway-74d8846df-5-2.1
startedAt: 2026-10-04T11:51:52Z
completedAt: 2026-10-04T11:53:32Z
phase: Successful
measurements: [0] at 11:52:52Z, [0] at 11:53:12Z, [0] at 11:53:32Z
```

No manual promotion was used. It automatically advanced through 40%, 60%, 80%, and finally reached Healthy at 100% with five Ready pods.

### Failed AnalysisRun and Automatic Abort

Bad revision 6 used `EVENTS_URL=http://broken-on-purpose:8081` and timeout 2000ms. Only for this controlled candidate, liveness and readiness used `/metrics`; otherwise `/health` would prevent it becoming Ready and Prometheus could not measure it. The stable revision retained normal settings.

The canary received real loadgen traffic and logged real failures:

```text
events service error: [Errno -2] Name or service not known
GET /events HTTP/1.1 502 Bad Gateway
```

```text
name: gateway-69957fc8bb-6-2
startedAt: 2026-10-04T11:58:37Z
completedAt: 2026-10-04T11:59:57Z
phase: Failed
measurements: [1] at 11:59:37Z, [1] at 11:59:57Z
message: failed (2) > failureLimit (1)
```

No manual abort command was issued. Rollout automatically changed to `Degraded / RolloutAborted`, `ActualWeight: 0`, and the bad ReplicaSet scaled down.

Beyond error rate, latency p95 and dependency-health availability would make the analysis more complete.

## Recovery

Git was restored to `APP_VERSION=v7-recovered`, `EVENTS_URL=http://events:8081`, `PAYMENTS_URL=http://payments:8082`, timeout 5000, and `/health` for both probes. The recovery AnalysisRun `gateway-6456b98586-7-2` completed Successfully at `2026-10-04T12:04:23Z` with three `[0]` measurements. It was then fully promoted; the Rollout was Healthy at ActualWeight 100 with exactly five Ready gateway pods, and Argo CD was `Synced Healthy` at revision `c0f161e066c70c7192de7eddd798eb59915d732f`.

The load generator was deleted. In-cluster Prometheus remains Running for Lab 8. Final gateway checks through a temporary port-forward were real:

```text
health={"status":"healthy","checks":{"events":"ok","payments":"ok","circuit_payments":"CLOSED"}}
events_count=5
reservation_id=5249d48a-94a9-4662-9d3e-ea0ff010f6bc
payment={"order_id":"5249d48a-94a9-4662-9d3e-ea0ff010f6bc","event_id":3,"quantity":1,"total_cents":15000,"status":"confirmed"}
```

## Acceptance Checklist

- [x] Argo Rollouts controller and Windows plugin installed
- [x] Gateway converted to a five-replica canary Rollout
- [x] Manual 20% canary, traffic evidence, promotion, and abort completed
- [x] Multi-step 20/40/60/80/100 strategy observed
- [x] Prometheus AnalysisTemplate installed and queried
- [x] Good AnalysisRun succeeded and auto-promoted
- [x] Bad AnalysisRun failed and automatically aborted
