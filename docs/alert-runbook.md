# Alert runbook

One page per alert: what fired, what to check, what it usually is.

**First move for almost everything: open Service Health Overview and look at the error attribution panel.** It tells you whether Kong or the upstream is at fault, which halves the search space before you do anything else.

| # | Alert | Severity |
|---|---|---|
| [1](#1--service-5xx-rate-breach) | Service 5xx Rate Breach | High |
| [2](#2--upstream-unreachable) | Upstream Unreachable | Critical |
| [3](#3--route-traffic-stopped) | Route Traffic Stopped | High |
| [4](#4--upstream-latency-p95-breach) | Upstream Latency p95 Breach | Medium |
| [5](#5--gateway-self-latency-breach) | Gateway Self-Latency Breach | High |
| [6](#6--balancer-retry-storm) | Balancer Retry Storm | Medium |
| [7](#7--upstream-target-dropped-from-rotation) | Upstream Target Dropped | Medium |
| [8](#8--approaching-upstream-read-timeout) | Approaching Read Timeout | Medium |
| [9](#9--rate-limit-rejections-spike) | Rate Limit Rejections Spike | Low |
| [10](#10--authentication-failure-spike) | Authentication Failure Spike | High |
| [11](#11--admin-api-unexpected-activity) | Admin API Unexpected Activity | Critical |
| [12](#12--log-ingestion-stalled) | Log Ingestion Stalled | High |

---

## 1 — Service 5xx Rate Breach

**Means:** a service's 5xx rate exceeded the configured percentage (default 2%) over 5 minutes, across at least `kong_min_requests` requests.

**Read the email first.** `caused_by_kong` vs `caused_by_upstream` decides who owns this:

- **`caused_by_upstream` dominant** — Vault or the admin API is returning errors and Kong is faithfully passing them through. Gateway is healthy. Escalate to the upstream owner.
- **`caused_by_kong` dominant** — Kong generated the errors itself. Check alerts 2 and 5, then the Upstream and Balancer Health dashboard.

**Then:**

```spl
`kong_base` | where status>=500
| stats count by service, route, status, error_source, upstream_status
| sort - count
```

**Usual causes:** an upstream deployment gone bad; a pod failing readiness while still in rotation; upstream resource exhaustion; a Kong plugin misbehaving (cross-check alert 5).

**Tuning:** if this fires without anyone caring, the threshold is below your p99. See operations-guide.md section 7.

---

## 2 — Upstream Unreachable

**Means:** Kong returned 5xx **without ever reaching the upstream** (`latencies.proxy = -1`). This is not an application error. Kong could not establish a connection at all.

**Check, in order:**

1. Pods running and passing readiness? `oc get pods -l app=vault`
2. Does the service DNS name resolve from a Kong pod? (`vault.internal.com`)
3. Network policy or service mesh change in the last hour?
4. Kong upstream/target health via the Admin API

```spl
`kong_base` | where short_circuited==1 AND status>=500
| stats count, values(status) as codes, dc(client_ip) as clients by service, route, upstream_host
```

**Usual causes:** all pods down or unready; DNS failure; network policy change; TLS handshake failure to the upstream (port 443 here).

**Note:** if `tries` is empty on these events, Kong never even selected a target — that points at DNS or discovery rather than the pods themselves.

---

## 3 — Route Traffic Stopped

**Means:** a route that normally averages 20+ requests per 15 minutes received **zero** in the last 15 minutes.

This is the alert that catches what nothing else catches. There is no error to see — traffic simply stopped.

**Check:**

1. Does the route still exist in Kong? `GET /routes/<name>` on the Admin API. A config push can delete a route silently.
2. Is the **client** down, rather than Kong? Traffic stopping is symmetrical — it looks identical from the gateway either way.
3. Was there a Kong config deployment in the window?
4. DNS or ingress change upstream of Kong?

```spl
`kong_base` | where route=="<route>" | timechart span=5m count
```

Widen the time range to see whether it stopped abruptly or tapered off. Abrupt usually means config; taper usually means clients.

**Expected during:** planned client maintenance. Suppress before the window rather than after the page.

---

## 4 — Upstream Latency p95 Breach

**Means:** upstream response time p95 exceeded the configured p95 threshold (default 2000 ms) over 10 minutes. Measured on `kong_proxied`, so short-circuited requests are excluded.

**Check:** open Latency Analysis. The key comparison is **Kong overhead vs upstream wait** — if Kong overhead is flat and upstream wait is climbing, the backend is genuinely slow.

```spl
`kong_proxied` | where service=="<service>"
| stats count, p50(proxy_ms), p95(proxy_ms), p99(proxy_ms), max(proxy_ms) by route, upstream_path
| sort - "p95(proxy_ms)"
```

**Usual causes:** upstream resource pressure; a slow dependency behind Vault (storage backend, HSM); one degraded pod dragging the distribution — check per-target latency on Upstream and Balancer Health; a genuine traffic increase.

**If it is one target:** that is alert 7's territory, and the pod should be cordoned.

---

## 5 — Gateway Self-Latency Breach

**Means:** Kong's own processing time p95 exceeded the configured threshold (default 100 ms). **The problem is inside the gateway, not the backend.**

The email includes `p95_upstream_ms` for contrast. Normal upstream latency alongside high Kong overhead confirms it.

**Check:**

1. **Which routes?** If only `ee-smp-functests`, suspect the rate-limiting plugin's datastore (Redis or the Kong DB, depending on `config.policy`).
2. Kong pod CPU throttling or memory pressure.
3. Kong datastore latency — the `cluster` rate-limit policy forces a read and a write per request.
4. DNS resolution time inside Kong.

```spl
`kong_proxied`
| stats count, p50(kong_ms), p95(kong_ms), max(kong_ms), p95(proxy_ms) by route
| sort - "p95(kong_ms)"
```

**On Kong 3.4 specifically:** `latencies.kong` includes request *receive* time. A client uploading a large body over a slow link inflates this without any gateway problem. Check `request.size` — except it is not currently emitted (see log-format-validation.md finding 1), so correlate with `response.size` and the client IP instead.

---

## 6 — Balancer Retry Storm

**Means:** more than 5% of requests needed a balancer retry. Kong is masking an unhealthy target by retrying against a different one.

**Why this matters:** users are still getting 200s. This will not appear in the error rate at all — until retries are exhausted, and then it appears all at once. This is a leading indicator, and acting on it is cheap.

**Find the culprit:**

```spl
`kong_base` | where retried==1
| eval failed_first_try = mvindex('tries{}.ip', 0)
| stats count as retry_events, max(try_count) as max_tries, values(status) as outcomes
        by service, failed_first_try
| sort - retry_events
```

A pod appearing repeatedly as `failed_first_try` while rarely appearing as `upstream_ip` is the sick one. Cordon or restart it.

**Note:** the service is configured with `retries = 5`, so there is real headroom being consumed silently here.

---

## 7 — Upstream Target Dropped From Rotation

**Means:** fewer distinct upstream targets are serving traffic than during the 24-hour baseline.

**Compare `current_list` against `baseline_list` in the email** — that identifies the missing pod directly.

**Check:**

1. Deliberate scale-down or rolling deploy? If so, expected. Suppress during planned changes.
2. `oc get pods` — is the pod gone, or running but not receiving traffic?
3. If running but idle: failing a Kong health check, or removed from the upstream target list.

```spl
`kong_proxied` | where isnotnull(upstream_ip)
| timechart span=5m dc(upstream_ip) as targets, count by service
```

**Escalate immediately if `current_targets` is 1.** Every request now depends on a single pod.

**False positives:** rolling deployments trip this by design. Either suppress during deploy windows or accept it as deployment confirmation.

---

## 8 — Approaching Upstream Read Timeout

**Means:** requests are taking more than 80% of the 60-second `read_timeout`. They have **not** failed yet.

This is the cheapest alert to act on, because nothing is broken for users yet.

```spl
`kong_proxied`
| where proxy_ms >= (`kong_read_timeout_ms` * 0.8)
| table _time, service, route, upstream_path, proxy_ms, total_ms, client_ip, upstream_ip
| sort - proxy_ms
```

**Check:** is it one path (a specific slow Vault operation), one client (a large or pathological request), or one target (a degraded pod)? The three have completely different fixes.

**Usual causes:** an expensive Vault operation such as a large list or a slow storage backend; the upstream nearly deadlocked; a genuinely long-running request that should be async.

**Do not "fix" this by raising `read_timeout`.** That converts a timeout into a connection pile-up.

---

## 9 — Rate Limit Rejections Spike

**Means:** at least 20 429s in 10 minutes.

**Read `enforced_by` first:**

- **`kong-plugin`** — Kong's rate-limiting plugin rejected the request before it reached the upstream. Working as designed; the question is whether the limit or the client is wrong.
- **`upstream-passthrough`** — Vault returned the 429 itself. This is Vault's quota, not Kong's. Different owner.

**Important:** the rate-limiting plugin is configured on `ee-smp-functests`. A `kong-plugin` 429 on any other route means the plugin's scope is broader than documented — worth confirming against the actual Kong config.

```spl
`kong_base` | where status==429
| eval enforced_by=if(short_circuited==1,"kong-plugin","upstream-passthrough")
| stats count by route, client_ip, enforced_by
| sort - count
```

**Do not use the `x-ratelimit-*` headers to investigate this.** They are Vault's quota headers, not Kong's — see log-format-validation.md finding 5.

**Usual causes:** a functional test run (expected on `ee-smp-functests`); a client retry loop without backoff; a limit set too low after a legitimate traffic increase.

---

## 10 — Authentication Failure Spike

**Means:** at least 25 401/403 responses on `auth-route` in 10 minutes.

**`distinct_clients` decides the triage path:**

- **One or two clients** — almost always a misconfigured workload with a stale or wrong service-account token. Operational, not security.
- **Many distinct clients at once** — a Vault-side auth backend problem, or a Kubernetes token issuer change. Check Vault before treating it as an attack.
- **Many clients, spread over time, varied namespaces** — now consider credential stuffing.

```spl
`kong_base` | where route=="auth-route" AND (status==401 OR status==403)
| eval auth_op=case(match(upstream_path,"/auth/kubernetes/login$"),"k8s-login",
                    match(upstream_path,"/auth/token/renew-self$"),"token-renew", true(),"other")
| stats count, values(status) as codes, values(vault_ns) as namespaces by client_ip, auth_op
| sort - count
```

**`token-renew` failing specifically** is its own signal: tokens are expiring rather than being refreshed, so every affected client will fall back to full re-authentication. Expect a login surge to follow.

---

## 11 — Admin API Unexpected Activity

**Means:** a caller reaching Kong's Admin API through `admin-route` was denied, or generated repeated errors.

**Treat as a security event until attributed.** `admin-route` exposes the control plane through the data plane — anyone who reaches it can read or rewrite routes, services and plugins.

**Check, in order:**

1. **Is `client_ip` a known workload?** CI/CD, an operator, a config-management job. If you cannot attribute it, escalate now rather than after investigating.
2. `first_seen` — is this a new caller, or an existing one that started failing?
3. `paths_touched` — read operations (`GET /services`) differ sharply from writes (`POST /routes`).
4. `auth_denied > 0` — someone tried to reach the control plane and was rejected. That is the finding, regardless of intent.

```spl
`kong_base` | where service=="admin-api"
| table _time, client_ip, status, upstream_path, proxy_ms, error_source
| sort - _time
```

**To reduce noise once callers are attributed,** add an allowlist to the alert search:

```spl
| where NOT (client_ip IN ("10.0.0.1","10.0.0.2"))
```

Keep the list short and reviewed. An allowlist that grows unchecked defeats the alert.

---

## 12 — Log Ingestion Stalled

**Means:** zero Kong events reached Splunk in 10 minutes.

**This alert monitors the monitor.** Without it, a broken log pipeline looks exactly like a perfectly healthy gateway — no errors, no latency, no alerts. Every other alert in this pack is blind while this one is firing.

**Check, in order:**

1. **Is Kong actually serving traffic?** Test an endpoint directly. Genuine zero traffic and broken logging look identical from Splunk.
2. Kong pods running? `oc get pods`
3. Is the log collector (Fluentd, Vector, whatever forwards `/openshift/logs.stdout`) running and shipping?
4. Splunk indexer or HEC health; disk or licence limits.
5. Did anything change the source path or index name?

```spl
index=kong_common | timechart span=5m count
index=kong_common | stats max(_time) as last_event | eval last_event=strftime(last_event,"%F %T")
```

**If Kong genuinely has quiet periods** (overnight, weekends), widen the window or restrict the alert's schedule to business hours. Do **not** lower its severity — a blind monitoring stack is a high-severity condition regardless of the hour.
