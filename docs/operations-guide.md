# Operations guide

How to get this running in Splunk. Written for someone with Splunk access and no prior context on this build.

**Time required:** ~30 minutes for the app install path, ~2 hours for the manual path, plus a baseline week before enabling alerts.

---

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Install — pick a path](#2-install--pick-a-path)
3. [Verify field extraction](#3-verify-field-extraction-do-this-first)
4. [Install the macros](#4-install-the-macros)
5. [Install the dashboards](#5-install-the-dashboards)
6. [Configure the alerts](#6-configure-the-alerts)
7. [Tune the thresholds](#7-tune-the-thresholds)
8. [Troubleshooting](#8-troubleshooting)
9. [Upgrade hazards](#9-upgrade-hazards)
10. [Acceptance checklist](#10-acceptance-checklist)

---

## 1. Prerequisites

| Requirement | Value | How to check |
|---|---|---|
| Index exists and has data | `kong_common` | `index=kong_common \| head 10` |
| Source path | `/openshift/logs.stdout` | `index=kong_common \| stats count by source` |
| Kong version | 3.4 | Every SPL here is written against the 3.4 serializer |
| Your role can search the index | — | If step 3 returns nothing but a colleague sees data, it is index permissions |
| Role to install | `admin` or `power` | Needed to create macros, dashboards and alerts |
| Mail server configured | — | Settings → Server settings → Email settings. **Required before enabling any alert.** |

Everything is built on two services and four routes:

| Route | Service | Notes |
|---|---|---|
| `admin-route` | `admin-api` | Kong's control plane, exposed through the data plane. Treat as privileged. |
| `auth-route` | `vault-svc` | Vault k8s login and token renewal |
| `generic-route` | `vault-svc` | |
| `ee-smp-functests` | `vault-svc` | Carries the `rate-limiting` plugin |

---

## 2. Install — pick a path

### Path A — deploy the app (fastest, needs filesystem access)

```bash
cp -r splunk-app/kong_proxy_monitoring $SPLUNK_HOME/etc/apps/
$SPLUNK_HOME/bin/splunk restart
```

On a search head cluster, push via the deployer instead:

```bash
cp -r splunk-app/kong_proxy_monitoring $SPLUNK_HOME/etc/shcluster/apps/
$SPLUNK_HOME/bin/splunk apply shcluster-bundle -target https://<sh>:8089
```

Then go to [step 3](#3-verify-field-extraction-do-this-first). Skip steps 4–5; they are already installed. You still need step 6 — **alerts ship disabled on purpose**.

### Path B — manual UI configuration (Splunk Cloud, or no filesystem access)

Work through steps 3 → 4 → 5 → 6 in order. Nothing is skippable, and the order matters: dashboards fail with "macro not found" if the macros are not in place first.

---

## 3. Verify field extraction (do this first)

Nothing else works if this does not. Run:

```spl
index=kong_common source="/openshift/logs.stdout" | head 5
```

You should see events. Now check that JSON fields resolve:

```spl
index=kong_common source="/openshift/logs.stdout" latencies.request=*
| head 5
| table _time, client_ip, "response.status", "route.name", "service.name", "latencies.proxy", upstream_status
```

**If the table is populated:** extraction works. Go to step 4 and **skip `props.conf` entirely** — you do not need it.

**If the columns are empty**, JSON extraction is not happening. Fix it in the UI:

1. Settings → Source types → find the sourcetype on your data (`index=kong_common | stats count by sourcetype`)
2. Set **Indexed Extractions** to `none` and **KV Mode** to `json`
3. Under Advanced, set `TIME_FORMAT` to `%s%3N` and `TIME_PREFIX` to `\"started_at\"\s*:\s*`
4. Save, then re-run the query

> **Why `KV_MODE` and not `INDEXED_EXTRACTIONS`:** `KV_MODE = json` is search-time — it works immediately on data already indexed. `INDEXED_EXTRACTIONS = json` is index-time: it only affects data ingested afterwards, must be deployed to the forwarder rather than the search head, and will not fix a single historical event. Do not enable both; that double-extracts every field.

> **Timestamp warning:** point extraction at `started_at` (epoch **milliseconds**, 13 digits). Do **not** use `route.created_at` or `service.created_at` — those are epoch **seconds** describing when the Kong config was created. Using them backdates every event to the route's creation date.

The reference `props.conf` is at `splunk-app/kong_proxy_monitoring/default/props.conf` if you prefer to copy values from it.

---

## 4. Install the macros

**Every dashboard and every alert depends on `kong_base`.** Install these first.

Settings → Advanced search → Search macros → New Search Macro. For each, set **Name** exactly and paste the **Definition** from `splunk-app/kong_proxy_monitoring/default/macros.conf`. Leave "Arguments" blank on all of them.

| Name | Purpose |
|---|---|
| `kong_index` | Index and source in one place. Change here if yours differ. |
| `kong_base` | **The foundation.** Normalises every field trap. |
| `kong_proxied` | `kong_base` filtered to requests that reached the upstream. Use for all latency stats. |
| `kong_read_timeout_ms` | `60000` — from the Kong service config. Used by one alert and one dashboard. |
| `kong_min_requests` | `50` — minimum volume before a ratio alert may fire. Used by four alerts. |

Five macros, not fifteen. A constant only earns a macro here when more than one
search uses it; the other ten alert thresholds are written inline in the `where`
clause of the alert that owns them, where you can read them in context. [Section
7](#7-tune-the-thresholds) lists all of them and where to edit each.

Set permissions on each: Settings → Search macros → Permissions → **All apps (system)**, Read: Everyone. Without this, ad-hoc searches from other app contexts fail with "macro not found".

**Smoke test — do not continue until this returns rows:**

```spl
`kong_base`
| head 5
| table _time, service, route, status, upstream_status, error_source, short_circuited, proxy_ms, total_ms
```

Then confirm the normalisation actually worked:

```spl
`kong_base`
| stats count,
        min(proxy_ms) as min_proxy_ms,
        sum(short_circuited) as short_circuited,
        dc(upstream_status) as distinct_upstream_status
```

- `min_proxy_ms` must **not** be negative. If it is, `kong_base` was pasted incorrectly.
- `short_circuited` may legitimately be 0 if nothing was rejected in the window.

```spl
`kong_base` | stats count by upstream_status
```

- Every value must be a 3-digit code or blank. If you see `-` or `502, 200`, the `mvfilter` line did not paste correctly.

---

## 5. Install the dashboards

For each XML file in `splunk-app/kong_proxy_monitoring/default/data/ui/views/`:

1. Dashboards → Create New Dashboard
2. Give it the title from the file's `<label>` element, save
3. Open it → Edit → **Source**
4. Delete everything, paste the full file contents, Save

| File | Title | Use it when |
|---|---|---|
| `kong_service_health_overview.xml` | Service Health Overview | **Start here in an incident.** Golden signals + Kong-vs-upstream attribution |
| `kong_route_latency.xml` | Route and Latency Analysis | Drilling into a route, or investigating slowness at any scope |
| `kong_upstream_balancer_health.xml` | Upstream and Balancer Health | A backend pod is suspected |
| `kong_traffic_rate_limiting.xml` | Traffic, Rate Limiting and Clients | Throttling, or a noisy client |
| `kong_security_tenancy.xml` | Security and Tenancy | Admin API audit, per-namespace or auth questions |

Route and Latency Analysis works at two scopes from one page: leave Route on "All routes" for the cross-cutting latency view, or pick a single route to drill in. It replaces what were previously two separate dashboards.

The Overview drills through to it when you click a row in the health matrix, passing both the service and the route. That link assumes the app context `kong_proxy_monitoring` — if you installed into a different app, edit the `<link>` element in `kong_service_health_overview.xml` to match.

> **Dropdowns are static, by design.** They list the two services and four routes as literal `<choice>` entries rather than running a search to discover them. A populated dropdown costs a full `kong_base` scan over the selected time range *on every page load*, which at a 24-hour range on a busy index is real I/O to render six words. Worse, a route with no traffic in the window vanishes from the list — and a silent route is exactly what you would be looking for. **If you add or rename a route in Kong, add a `<choice>` line** to the dashboards that carry a Route selector (`kong_service_health_overview.xml`, `kong_route_latency.xml`, `kong_traffic_rate_limiting.xml`). Unknown traffic still appears in every table regardless, under `(no-route-matched)`.

---

## 6. Configure the alerts

### 6.1 Read this first

**All twelve alerts ship disabled (`disabled = 1`). That is deliberate.**

The thresholds come from the Kong config and general gateway norms, not from your traffic. Enabling all twelve untuned is how a monitoring rollout gets muted in its first fortnight. Work through [step 7](#7-tune-the-thresholds) first.

**Before enabling anything:**
1. Confirm the mail server works (Settings → Server settings → Email settings).
2. Replace `CHANGEME@example.com` on every alert you enable.

An alert with no mail server triggers and then silently fails to deliver — worse than no alert, because it looks like coverage.

### 6.2 Recommended enable order

| When | Alerts | Why these first |
|---|---|---|
| **Immediately** | Log Ingestion Stalled · Upstream Unreachable · Admin API Unexpected Activity | Threshold-light. They detect structural conditions, not statistical ones. |
| **After 1 day** | Service 5xx Rate Breach · Route Traffic Stopped · Authentication Failure Spike | Need a rough sense of normal volume |
| **After a baseline week** | Everything else | Percentile-based, so they need a real distribution |

### 6.3 The twelve

| # | Alert | Severity | Schedule | Window | Fires when |
|---|---|---|---|---|---|
| 1 | Service 5xx Rate Breach | High | `*/5 * * * *` | 5m | 5xx% > 2 (inline), min 50 requests |
| 2 | Upstream Unreachable | Critical | `1-59/5 * * * *` | 5m | ≥10 short-circuited 5xx |
| 3 | Route Traffic Stopped | High | `2-59/5 * * * *` | 15m | Busy route drops to zero |
| 4 | Upstream Latency p95 Breach | Medium | `3-59/5 * * * *` | 10m | p95 > 2000 ms (inline) |
| 5 | Gateway Self-Latency Breach | High | `4-59/5 * * * *` | 10m | Kong overhead p95 > 100 ms (inline) |
| 6 | Balancer Retry Storm | Medium | `5-59/10 * * * *` | 10m | >5% of requests retried |
| 7 | Upstream Target Dropped | Medium | `7,22,37,52 * * * *` | 15m | Fewer targets than 24h baseline |
| 8 | Approaching Read Timeout | Medium | `8-59/10 * * * *` | 10m | Any request >80% of 60s |
| 9 | Rate Limit Rejections Spike | Low | `9-59/10 * * * *` | 10m | ≥20 429s |
| 10 | Authentication Failure Spike | High | `6-59/10 * * * *` | 10m | ≥25 401/403 on auth-route |
| 11 | Admin API Unexpected Activity | Critical | `*/10 * * * *` | 10m | Any Admin API 401/403, or ≥5 errors |
| 12 | Log Ingestion Stalled | High | `*/5 * * * *` | 10m | Zero events |

Schedules are staggered on purpose so twelve searches do not dispatch on the same minute.

Triage steps for each are in [alert-runbook.md](alert-runbook.md).

### 6.4 Creating them manually

Every alert uses the same trigger pattern: **the search returns rows only when the condition is breached**, so the trigger is always "Number of Results is greater than 0". You do not need custom trigger conditions.

For each alert in `savedsearches.conf`:

1. Search → paste the `search =` value → run it once to confirm it parses
2. Save As → Alert
3. Fill in per the mapping below
4. **Leave it disabled** until you have tuned it

**`savedsearches.conf` → UI field mapping.** The two use different names, which is the main source of confusion:

| `savedsearches.conf` | Splunk UI field |
|---|---|
| `[Stanza Name]` | Title |
| `search` | Search string |
| `description` | Description |
| `cron_schedule` | Alert type → Scheduled → Run on Cron Schedule → Cron Expression |
| `dispatch.earliest_time` / `dispatch.latest_time` | Time Range on the search |
| `alert_type = number of events` | Trigger alert when → **Number of Results** |
| `alert_comparator = greater than` | is **greater than** |
| `alert_threshold = 0` | **0** |
| `alert.severity` | Severity (1 Info, 2 Low, 3 Medium, 4 High, 5 Critical) |
| `alert.suppress = 1` | Throttle → checked |
| `alert.suppress.period` | Suppress triggering for |
| `action.email.to` | Trigger Actions → Send email → To |
| `action.email.subject` | Subject |
| `action.email.inline = 1` | Include → Inline table |
| `disabled = 1` | The alert's Enabled/Disabled toggle |

> `counttype`, `relation` and `quantity` also appear in the file. They are the legacy equivalents of `alert_type` / `alert_comparator` / `alert_threshold`, included so the file works on older Splunk versions. The UI only shows one set. Ignore them.

### 6.5 Throttling behaviour — a real trade-off

Every alert uses time-based suppression, which is **global to the search**. Once "Service 5xx Rate Breach" fires for `vault-svc`, a breach on `admin-api` within the same window is also suppressed.

That is the safe default — it cannot storm. If you want per-service granularity, on that alert set:

```
alert.digest_mode = 0
alert.suppress.fields = service
```

You then get one email per breaching row, throttled independently per service. Louder, but nothing gets masked. Decide per alert; the Admin API one is the usual candidate for per-result mode.

---

## 7. Tune the thresholds

Run these over **a full week** of real traffic (`earliest=-7d@d latest=@d`). Weekday and weekend profiles differ enough that a weekday-only sample will misfire on Monday.

### Where every threshold lives

This is the complete tuning surface. Two are macros because more than one search uses them; the rest are inline in the alert that owns them.

| Threshold | Default | Where to edit |
|---|---|---|
| Minimum request volume for ratio alerts | `50` | **macro** `kong_min_requests` |
| Upstream read timeout | `60000` | **macro** `kong_read_timeout_ms` |
| 5xx breach percentage | `2` | Alert 1 — `where error_pct > 2` |
| Short-circuited 5xx count | `10` | Alert 2 — `where failures >= 10` |
| Traffic-stopped baseline | `20` per 15m | Alert 3 — `where baseline_per_15m >= 20` |
| Upstream p95 | `2000` ms | Alert 4 — `where p95_ms > 2000` |
| Kong self-latency p95 | `100` ms | Alert 5 — `where p95_kong_ms > 100` |
| Retry rate | `5` % | Alert 6 — `where retry_pct > 5` |
| Target-dropped | structural | Alert 7 — no threshold; compares to a 24h baseline |
| Timeout proximity | `80` % of read timeout | Alert 8 — `proxy_ms >= (kong_read_timeout_ms * 0.8)` |
| 429 burst | `20` | Alert 9 — `where throttled >= 20` |
| Auth denial burst | `25` | Alert 10 — `where denials >= 25` |
| Admin API error count | `5` | Alert 11 — `where auth_denied > 0 OR errors >= 5` |
| Ingestion stall | structural | Alert 12 — no threshold; fires on zero events |

> **Edit in `local/`, not `default/`.** Splunk reads `default/` then overlays `local/`. Anything you change in `default/` is lost the next time the app is redeployed. Copy just the stanza you are changing into `local/macros.conf` or `local/savedsearches.conf`. Edits made through the Splunk UI land in `local/` automatically — it is only hand-editing the files that gets this wrong.

### Set the 5xx breach percentage (alert 1) and `kong_min_requests`

```spl
`kong_base`
| bin _time span=5m
| stats count as requests, sum(eval(if(status>=500,1,0))) as errors by _time, service
| eval error_pct = round(errors*100/requests, 2)
| stats count as buckets,
        avg(requests) as avg_requests_per_5m,
        perc50(error_pct) as p50, perc95(error_pct) as p95,
        perc99(error_pct) as p99, max(error_pct) as worst
        by service
```

Set alert 1's `error_pct` threshold **above p99** — you want the alert to mean "worse than the worst normal week", not "slightly unusual". Set `kong_min_requests` to roughly half of `avg_requests_per_5m` for your quietest service, so quiet periods cannot produce a misleading ratio.

### Set the latency thresholds (alerts 4 and 5)

```spl
`kong_proxied`
| stats perc50(proxy_ms) as upstream_p50, perc95(proxy_ms) as upstream_p95,
        perc99(proxy_ms) as upstream_p99, max(proxy_ms) as upstream_max,
        perc50(kong_ms)  as kong_p50,     perc95(kong_ms)  as kong_p95,
        perc99(kong_ms)  as kong_p99,     max(kong_ms)     as kong_max
        by service
```

Set alert 4's threshold to roughly `upstream_p99`, and alert 5's to roughly `kong_p99` — with a floor of 50ms, because Kong overhead is normally single-digit milliseconds and a threshold that tight will fire on ordinary jitter.

### Set the retry threshold (alert 6, currently 5%)

```spl
`kong_base`
| bin _time span=10m
| stats count as requests, sum(retried) as retried by _time, service
| eval retry_pct = round(retried*100/requests, 3)
| stats perc50(retry_pct), perc95(retry_pct), perc99(retry_pct), max(retry_pct) by service
```

If baseline retries are near zero, drop the threshold well below 5% — on a healthy cluster any sustained retry rate is a signal, and 5% would be a serious incident by the time it fired.

### Set the traffic-stopped baseline (alert 3, currently 20 per 15m)

```spl
`kong_base`
| bin _time span=15m
| stats count as c by _time, route
| stats min(c) as quietest, perc05(c) as p05, avg(c) as mean, max(c) as busiest by route
```

Any route whose `quietest` bucket is already 0 will never alert with the default guard — that is intended. If you need coverage on a genuinely low-volume route, lower the `baseline_per_15m >= 20` guard in that alert's search and accept more noise.

### Check the Admin API caller list before enabling alert 11

```spl
`kong_base` | where service=="admin-api"
| stats count, values(status) as codes, min(_time) as first, max(_time) as last by client_ip
| eval first=strftime(first,"%F %T"), last=strftime(last,"%F %T")
```

Attribute every IP to a known workload. Add unexplained ones to the investigation list *before* enabling, or the first firing will be an unattributable IP at 2am with no baseline to compare against.

---

## 8. Troubleshooting

### "Error in 'SearchParser': The search specifies a macro 'kong_base' that cannot be found"

The macro is not installed, or not visible from the current app. Check Settings → Advanced search → Search macros, set the filter to "All", and confirm permissions are **Global / All apps**.

### Dashboards render but every panel is empty

Work backwards:

```spl
index=kong_common source="/openshift/logs.stdout"      → any events at all?
... latencies.request=*                                 → does JSON extraction work?
`kong_base` | head 5                                    → does the macro work?
```

The first that returns nothing is your failure point. If step 2 fails, return to [step 3](#3-verify-field-extraction-do-this-first).

### Latency numbers look impossibly good, or are negative

A panel is using `latencies.proxy` raw instead of `proxy_ms` from `kong_base`, so the `-1` sentinel is being averaged in. Confirm with:

```spl
`kong_base` | stats min(proxy_ms), min('latencies.proxy'), sum(short_circuited)
```

`min(proxy_ms)` must not be negative. `min('latencies.proxy')` being `-1` is normal and expected.

### Retried requests are missing from a custom search

You used `tonumber('upstream_status')`, which returns null on the comma-joined retry form. Use `upstream_status` from `kong_base` instead. See [field-reference.md](field-reference.md) gotcha 3.

### `try_count` filters drop events unexpectedly

`mvcount()` on an absent field returns null, not 0. Short-circuited requests have no `tries` array. `kong_base` already wraps this in `coalesce(..., 0)`; if you rebuilt the logic yourself, add it.

### Every event has the same timestamp

Timestamp extraction is pointed at `route.created_at` or `service.created_at` (epoch seconds, config creation time) instead of `started_at` (epoch milliseconds). Fix in the sourcetype settings — [step 3](#3-verify-field-extraction-do-this-first).

### Alerts never fire

In order: is the alert enabled (all ship disabled)? Is `enableSched = 1`? Does the search return rows when run manually over a window you know contains an incident? Is the mail server configured? Check Activity → Triggered Alerts — if entries appear there but no email arrives, it is mail configuration, not the alert.

### Alerts fire constantly

Thresholds are untuned. Go to [step 7](#7-tune-the-thresholds). Do not fix this by disabling the alert and moving on — that is how the whole pack ends up ignored.

### `x-ratelimit-*` panels are empty

Expected if the upstream is not emitting those headers. They are **Vault's** quota headers, not Kong's — see [log-format-validation.md](log-format-validation.md) finding 5. Kong's own rate limiting is measured elsewhere on that dashboard, via 429 + short-circuit.

---

## 9. Upgrade hazards

### Kong 3.4 → 3.5 or later

Two serializer changes will shift your numbers without any config change:

1. **`latencies.kong` loses receive time.** On 3.4 it is `KONG_PROXY_LATENCY + KONG_RECEIVE_TIME`. On 3.5+ receive time becomes a separate `latencies.receive` field. Kong overhead will appear to drop. **Re-tune alert 5's threshold after upgrading**, and consider adding `receive_ms` to `kong_base`.

2. **New fields appear:** `latencies.receive`, `source`, `workspace`, `workspace_name`. Nothing breaks — `kong_base` ignores fields it does not reference — but `source` in particular is worth adopting, as it directly labels whether Kong or the upstream produced the response and would simplify the `error_source` logic.

### If `request.*` starts appearing

Currently absent (see [log-format-validation.md](log-format-validation.md) finding 1). If it returns, add to `kong_base`:

```spl
       request_id     = 'request.id',
       method         = 'request.method',
       client_path    = 'request.uri',
       request_bytes  = tonumber('request.size'),
       tls_version    = 'request.tls.version',
```

That unlocks HTTP-method breakdown and — more importantly — `request_id` correlation against Kong error logs via the `X-Kong-Request-Id` header.

### If the index or source path changes

Edit the `kong_index` macro only. Everything else follows.

---

## 10. Acceptance checklist

Sign-off criteria. Tick every box.

**Extraction**
- [ ] `index=kong_common source="/openshift/logs.stdout" latencies.request=*` returns events
- [ ] Dotted fields resolve in a `table` command
- [ ] Event timestamps match request times, not a single repeated config-creation date

**Macros**
- [ ] All eight macros exist with global permissions
- [ ] `` `kong_base` | head 5 `` returns rows
- [ ] `min(proxy_ms)` is not negative
- [ ] `` `kong_base` | stats count by upstream_status `` shows only 3-digit codes or blank
- [ ] `try_count` is never null

**Dashboards**
- [ ] All five load without a macro or parse error
- [ ] Service Health Overview populates every panel
- [ ] Dashboards issue no search on page load beyond their own panels (dropdowns are static)
- [ ] Clicking a row in the health matrix opens Route and Latency Analysis with the route and service pre-selected
- [ ] Route and Service dropdowns list every route and service in your Kong config

**Alerts**
- [ ] Mail server configured and test email received
- [ ] `CHANGEME@example.com` replaced on every enabled alert
- [ ] Each enabled alert returns rows when run over a known-incident window
- [ ] Each enabled alert returns **zero** rows over a known-quiet window
- [ ] Thresholds set from the baseline week, not left at shipped defaults

**Handover**
- [ ] On-call knows to open Service Health Overview first
- [ ] On-call understands `error_source` = kong vs upstream
- [ ] Open items in [log-format-validation.md](log-format-validation.md) assigned to someone
