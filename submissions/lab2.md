# Lab 2 — Containerization

## Image Inspection

Before changes: `app-gateway 226MB`, `app-events 245MB`, `app-payments 223MB`. Gateway had the pip-install layer at 29.6MB; a Python base-image layer was larger at 87.5MB, so pip was not the largest layer. The gateway image had 13 history entries in this environment.

## Container Inspection

```text
/app-gateway-1 172.27.0.6
/app-events-1 172.27.0.5
/app-payments-1 172.27.0.2
PAYMENT_FAILURE_RATE=0.0
PAYMENT_LATENCY_MS=0
```

Before the change, gateway ran as:

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

## Service Discovery

Inside gateway, `events` resolved to `172.27.0.5`; `/etc/resolv.conf` used `nameserver 127.0.0.11`. Compose created the `app_default` bridge network and its embedded DNS resolves the service hostname to that dynamic container IP.

```text
GET http://events:8081/health -> {"status":"healthy","checks":{"postgres":"ok","redis":"ok"}}
GET http://payments:8082/health -> {"status":"healthy","failure_rate":0.0,"latency_ms":0}
```

## Live Debugging

The Python health calls above succeeded from the gateway container. `docker network inspect app_default` showed gateway `172.27.0.6/16`, events `172.27.0.5/16`, payments `172.27.0.2/16`, postgres `172.27.0.4/16`, and redis `172.27.0.3/16`.

## Logs Analysis

Gateway and Events logs contain endpoint requests at close timestamps, but ordinary list/reserve requests have no request ID. That is only temporal/endpoint correlation and is not reliable distributed tracing.

## Network Inspection

All five containers were attached to the Compose-created `app_default` network. Service names, not hard-coded IPs, are used by gateway (`http://events:8081` and `http://payments:8082`).

## Dockerfile Optimization

Added the required `.dockerignore` to each Python service and added a system `app` user before each `CMD`. No `chown` was needed: the applications ran successfully as the non-root user.

```text
__pycache__
*.pyc
.git
.env
*.md
.vscode
```

## Before/After Comparison

After a no-cache rebuild: gateway `226MB`, events `245MB`, payments `223MB`; image sizes did not materially change. The contexts are already small and Dockerfiles copy only requirements and main.py, so `.dockerignore` mainly protects future build contexts and secrets.

```text
docker exec app-gateway-1 whoami -> app
uid=100(app) gid=101(app) groups=101(app)
GET /health -> {"status":"healthy","checks":{"events":"ok","payments":"ok","circuit_payments":"CLOSED"}}
POST full purchase -> {"order_id":"49780e41-7d1a-4f55-b423-86b9eb06c2a3",...,"status":"confirmed"}
```

## Request Tracing Bonus

For reservation `ef1d0627-6bfa-4516-97a7-bbe6547f5bc9`, timestamped logs showed Events reserve at `22:14:31.328`, Payments success (`PAY-6EE0D77F`) at `22:14:31.403`, Events confirmation at `22:14:31.410`, and gateway’s 200 response at `22:14:31.412`. The observable reserve-to-pay-response interval was about 84 ms; gateway does not log receipt of the reserve request, so an exact whole-flow duration cannot be calculated from logs alone.

## Conclusions

The services communicate through Compose DNS on a bridge network. Running as `app` reduces container privilege without changing application behavior.
