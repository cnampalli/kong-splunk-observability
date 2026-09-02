# Field reference

How Kong 3.4's JSON maps into Splunk, what `kong_base` renames it to, and the SPL mistakes that field names cause.

---

## 1. Raw JSON to Splunk field names

Splunk flattens nested JSON with dots, and arrays with `{}`.

| Kong JSON | Splunk field name | Type | Notes |
|---|---|---|---|
| `client_ip` | `client_ip` | string | Kong's `remote_addr`. May be a load balancer — see gotcha 6. |
| `started_at` | `started_at` | number | Epoch **milliseconds** (13 digits) |
| `upstream_uri` | `upstream_uri` | string | Path sent upstream, **after** rewriting |
| `upstream_status` | `upstream_status` | string | Four possible shapes — see gotcha 3 |
| `response.status` | `response.status` | number | What the client received |
| `response.size` | `response.size` | number | Bytes sent to client |
| `response.headers.*` | `response.headers.x-vault-namespace` etc. | string | Lowercased header names |
| `latencies.kong` | `latencies.kong` | number | ms. On 3.4 includes receive time |
| `latencies.proxy` | `latencies.proxy` | number | ms, or `-1` sentinel |
| `latencies.request` | `latencies.request` | number | ms, total end to end |
| `route.name` | `route.name` | string | Null when no route matched |
| `route.methods[]` | `route.methods{}` | multivalue | Configured methods, not the observed one |
| `route.paths[]` | `route.paths{}` | multivalue | Route match patterns |
| `service.name` | `service.name` | string | `vault-svc` or `admin-api` |
| `service.host` | `service.host` | string | e.g. `vault.internal.com` |
| `service.read_timeout` | `service.read_timeout` | number | 60000 ms |
| `service.retries` | `service.retries` | number | 5 |
| `tries[].ip` | `tries{}.ip` | multivalue | One entry per attempt |
| `tries[].port` | `tries{}.port` | multivalue | |
| `tries[].balancer_latency` | `tries{}.balancer_latency` | multivalue | ms, usually rounds to 0 |
| `tries[].balancer_latency_ns` | `tries{}.balancer_latency_ns` | multivalue | nanoseconds, the precise one |
| `route.created_at` | `route.created_at` | number | Epoch **seconds** — config age, not request time |

---

## 2. What `kong_base` gives you

Use these. They are clean names, correctly typed, with the traps already handled.

| Macro field | Derived from | Meaning |
|---|---|---|
| `route` | `route.name` | Route name, or `(no-route-matched)` |
| `service` | `service.name` | Service name, or `(no-service)` |
| `upstream_host` | `service.host` | Configured upstream hostname |
| `status` | `response.status` | Numeric status the client received |
| `status_class` | derived | `1xx` / `2xx` / `3xx` / `4xx` / `5xx` / `unknown` |
| `kong_ms` | `latencies.kong` | Gateway overhead |
| `proxy_ms` | `latencies.proxy` | Upstream wait. **Null when short-circuited**, never `-1` |
| `total_ms` | `latencies.request` | End to end |
| `short_circuited` | `latencies.proxy < 0` | `1` = never reached the upstream |
| `upstream_status` | normalised | Valid 3-digit code, or null. Never `-`, `""` or a list |
| `error_source` | derived | `ok` / `kong` / `upstream` / `unknown` |
| `resp_bytes` | `response.size` | Bytes to client |
| `upstream_path` | `upstream_uri` | Post-rewrite path |
| `vault_ns` | `response.headers.x-vault-namespace` | Vault namespace |
| `try_count` | `mvcount(tries{}.ip)` | Attempts. `0` if never reached the balancer |
| `retried` | `try_count > 1` | `1` = a balancer retry happened |
| `upstream_ip` | last of `tries{}.ip` | The target that **served** the request |
| `balancer_ms` | last of `tries{}.balancer_latency` | Target selection time |
| `read_timeout_ms` | `service.read_timeout` | **Per-service**, read from the event — not a configured constant |
| `connect_timeout_ms` | `service.connect_timeout` | Per-service |
| `write_timeout_ms` | `service.write_timeout` | Per-service |
| `max_retries` | `service.retries` | The service's configured retry budget |
| `retries_exhausted` | `try_count > max_retries` | `1` = Kong ran out of retries; the failure reached the user |

**Not in `kong_base`, available if `request.*` is restored:** `request.id`, `request.method`, `request.uri`, `request.querystring`, `request.size`, `request.tls.version`. See [log-format-validation.md](log-format-validation.md) finding 1.

---

## 3. Which base macro to use

| Task | Macro | Why |
|---|---|---|
| Counting requests, errors, status codes | `kong_base` | You want every request, including rejected ones |
| Anything involving `proxy_ms` | **`kong_proxied`** | Short-circuited requests never reached the upstream. Including them measures something that did not happen. |
| Error attribution | `kong_base` | `error_source` is already computed |
| Kong overhead (`kong_ms`) | `kong_proxied` | Comparable population; short-circuits have a different overhead profile |

The rule: **if the statistic describes the upstream, use `kong_proxied`.**

---

## 4. SPL gotchas

### Gotcha 1 — dotted field names need single quotes in `eval`

```spl
| eval s = response.status          ← WRONG. Parsed as field "response" minus field "status".
| eval s = 'response.status'        ← correct
| stats avg("latencies.proxy")      ← correct in stats (double quotes)
```

`kong_base` renames everything once so you never hit this downstream. If you are writing ad-hoc SPL against raw fields, quote them.

### Gotcha 2 — `latencies.proxy = -1` is not a duration

```spl
| stats avg('latencies.proxy')      ← WRONG. -1 values drag the mean down.
| `kong_proxied` | stats avg(proxy_ms)   ← correct
```

A gateway rejecting 30% of traffic at the edge will report *better* upstream latency than a healthy one if you get this wrong. It fails silently — no error, just a wrong number.

### Gotcha 3 — `upstream_status` is not always a number

Four shapes: `200`, `"502, 200"` (retry), `-` (no upstream reached), null (short-circuited).

```spl
| eval us = tonumber('upstream_status')     ← null on every retried request
| `kong_base` | ... upstream_status ...     ← correct, already normalised
```

Note the direction of the failure: the naive version silently *drops* the retried requests — the ones most worth looking at.

### Gotcha 4 — `started_at` is milliseconds, `created_at` is seconds

```spl
| eval t = strftime(started_at, "%F %T")           ← WRONG. Off by ~54,000 years.
| eval t = strftime(started_at/1000, "%F %T")      ← correct
```

`route.created_at` and `service.created_at` are seconds and describe **when the Kong config entity was created**, not the request. Never use them as an event timestamp.

### Gotcha 5 — `tries{}` is multivalue, and order carries meaning

```spl
| eval served_by  = mvindex('tries{}.ip', -1)   ← the target that succeeded
| eval failed_try = mvindex('tries{}.ip',  0)   ← on a retry, the one that failed
| eval attempts   = coalesce(mvcount('tries{}.ip'), 0)   ← coalesce matters
```

`mvcount()` on an absent field returns **null**, not 0. Without `coalesce`, short-circuited requests get a null `try_count` and silently drop out of any `where try_count ...` filter.

`tries` is also absent entirely when Kong short-circuits — there was no balancer attempt to record.

### Gotcha 6 — `client_ip` may not be the client

`client_ip` is Kong's `remote_addr`. Behind an ingress controller or load balancer that is the LB's address. If the top-talkers panel shows one or two IPs carrying all traffic, that is what is happening — configure `real_ip_header` and `trusted_ips` in `kong.conf` before treating that panel as a security signal.

### Gotcha 7 — `x-ratelimit-*` headers are Vault's, not Kong's

Kong emits `RateLimit-Limit` and window-suffixed `X-RateLimit-Limit-Second`. The bare lowercase form in these logs is HashiCorp Vault's quota, proxied through. Detect Kong throttling as `status==429 AND short_circuited==1`. Full reasoning in [log-format-validation.md](log-format-validation.md) finding 5.

### Gotcha 8 — the timeout and retry budget are per-service, not global

```spl
| where proxy_ms >= 48000                        ← WRONG. Assumes every service uses a 60s timeout.
| where proxy_ms >= (read_timeout_ms * 0.8)      ← correct, uses each service's own value
```

`service.read_timeout` and `service.retries` are in every event. A service configured with a 10-second timeout will time out at 10s while a hardcoded 48s threshold is still calling it healthy. Same for retries: `max_retries` is what makes `retries_exhausted` computable, and it differs per service.

Note the arithmetic — `retries = 5` means up to **6** attempts, so exhaustion is `try_count > max_retries`, not `>=`.

### Gotcha 9 — do not filter on `route.name=*` to find Kong events

```spl
index=kong_common source="/openshift/logs.stdout" route.name=*        ← drops no-route 404s
index=kong_common source="/openshift/logs.stdout" latencies.request=* ← correct
```

Requests matching no route have a null `route`. Those are Kong-generated 404s — exactly the events you need when someone reports that an endpoint disappeared. `latencies.request` is present on every serializer event and on no other line in the stdout stream.

---

## 5. Useful one-liners

```spl
# Is data arriving, and does JSON extraction work?
index=kong_common source="/openshift/logs.stdout" latencies.request=* | head 5

# Does kong_base work?
`kong_base` | head 5 | table _time service route status upstream_status error_source proxy_ms

# Kong or upstream? The first question in an incident.
`kong_base` | where status>=400 | stats count by error_source, service, route

# Which target is silently failing (Kong is retrying past it)?
`kong_base` | where retried==1 | eval failed=mvindex('tries{}.ip',0) | stats count by failed

# Confirm the -1 sentinel is handled
`kong_base` | stats count, min(proxy_ms) as min_proxy, sum(short_circuited) as short_circuited

# Confirm upstream_status normalisation (should show only 3-digit codes)
`kong_base` | stats count by upstream_status

# What is Kong's own overhead costing, per route?
`kong_proxied` | stats p50(kong_ms), p95(kong_ms), count by route | sort - "p95(kong_ms)"
```
