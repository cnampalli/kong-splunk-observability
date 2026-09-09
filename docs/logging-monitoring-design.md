# Logging and monitoring — detailed design

Detailed design (LLD) for observability of Kong Gateway **3.4** proxy traffic fronting `vault-svc` and `admin-api`, implemented as the Splunk app `kong_proxy_monitoring`.

This document describes **what is monitored, how it is measured, why each threshold exists, and who gets woken up**. It is the design record behind the shipped artefacts — it does not replace them:

| This document says | These documents do |
|---|---|
| Why the metric is defined this way | [field-reference.md](field-reference.md) — how to use the fields |
| Why the alert exists and how it is tuned | [alert-runbook.md](alert-runbook.md) — what to do when it fires |
| The escalation model | [triage-guide.md](triage-guide.md) — the live escalation matrix and symptom index |
| The rollout and threshold methodology | [operations-guide.md](operations-guide.md) — the step-by-step install and tuning procedure |
| The data contract we depend on | [log-format-validation.md](log-format-validation.md) — the field-by-field validation against Kong 3.4 |

**Status:** design complete, implementation shipped, **all 13 alerts ship disabled pending threshold fitting** (§6.3, §10.1).
**Audience:** platform/gateway engineering, observability engineering, on-call ops, and whoever reviews this before it goes live.

---

## Contents

1. [Design objectives and principles](#1-design-objectives-and-principles)
2. [Logging design](#2-logging-design)
3. [Metrics catalogue](#3-metrics-catalogue)
4. [Dashboard design](#4-dashboard-design)
5. [Alerting design](#5-alerting-design)
6. [Threshold design](#6-threshold-design)
7. [Notification design](#7-notification-design)
8. [Escalation design](#8-escalation-design)
9. [Runbook design](#9-runbook-design)
10. [Operational lifecycle](#10-operational-lifecycle)
11. [Non-functional design](#11-non-functional-design)
12. [Design decisions register](#12-design-decisions-register)
13. [Assumptions, open items and errata](#13-assumptions-open-items-and-errata)
14. [Acceptance criteria](#14-acceptance-criteria)

---

## 1. Design objectives and principles

### 1.1 The primary question

Every design choice here serves one question, because it is the first question of every gateway incident and the one that halves the search space:

> **Is Kong broken, or is the upstream broken?**

Kong's log serializer can answer this definitively — it records both what Kong did and what the upstream said, in the same event. The design makes that split a first-class, precomputed field (`error_source`), surfaces it as the second row of the default dashboard, and puts it in the body of every 5xx alert email. Nothing else in this pack is more important.

### 1.2 Objectives

| # | Objective | How the design meets it |
|---|---|---|
| O1 | Attribute every failure to gateway or upstream | `error_source` computed once in `kong_base` (§2.6), surfaced on every error panel and 5xx alert |
| O2 | Detect failures that produce no errors | Retry storms, target drop-out, timeout proximity, traffic cessation — four alerts whose entire purpose is pre-error detection (§5.1 pattern P3) |
| O3 | Never report health when we are blind | `Kong - Log Ingestion Stalled` monitors the monitor (§5.2 A12) |
| O4 | Make latency statistics honest | The `-1` sentinel is nulled and a separate latency-safe population exists (§2.7) |
| O5 | Be installable with no privileged deployment | No KV Store, no lookups, no collections, no filesystem state (§2.8, DD-3) |
| O6 | Survive redeployment of a different Kong install | Five-name portability surface (§2.9) |
| O7 | Not get muted in its first fortnight | Everything ships disabled with a documented baseline-then-enable procedure (§10.1) |
| O8 | Give the on-call engineer a decision, not a graph | Every alert email carries the attributing columns inline; every alert has a runbook section (§7.2, §9) |

### 1.3 Principles

1. **Solve each data trap exactly once.** Kong 3.4 sets four specific traps (§2.10). They are handled in one macro, not re-solved in 62 panels and 13 alerts where they would be got wrong differently each time.
2. **Read configuration from the data, never hardcode it.** `read_timeout`, `connect_timeout`, `write_timeout` and `retries` are in *every log event*, per service. Hardcoding `60000` would mean a service configured at 10s had to reach 48s before a timeout-proximity alert cared — long after it had already timed out.
3. **A statistic must describe a population that actually exists.** Requests Kong short-circuited never reached the upstream; including them in an upstream latency percentile measures something that did not happen.
4. **Alerts return rows only when breached.** The trigger is always `number of results > 0`. No custom trigger conditions, no alert logic split between the search and the alert config.
5. **Degrade to empty, never to wrong.** A broken topology source yields unfiltered dashboards, not broken ones. An absent role yields an empty panel, not a misattributed one.
6. **A threshold with no stated derivation is a guess.** Every number in §6.2 has either a structural justification or a baseline query that produces it.
7. **No stored state.** Baselines are computed in-search against a prior window. Nothing to install, nothing to permission, nothing to drift.

---

## 2. Logging design

### 2.1 Log source and transport

```
Kong Gateway 3.4 (OpenShift pod)
   │   http-log / file-log via the Kong log serializer → container stdout
   ▼
OpenShift container stdout
   │   cluster log forwarding (out of scope for this pack)
   ▼
Splunk index=kong_common   source="/openshift/logs.stdout"
   │
   ▼  search time
`kong_index` → `kong_base` → `kong_proxied` / `kong_topology_source`
   │
   ├─► 5 dashboards (62 titled panels)
   └─► 13 alerts
```

The single source of truth for data location is the `kong_index` macro:

```
[kong_index]
definition = index=kong_common source="/openshift/logs.stdout"
```

**Design intent:** an index or source-path change is a one-line edit that every dashboard and every alert inherits automatically. Nothing else in the pack names the index.

**Boundary of responsibility.** This pack owns everything from `index=kong_common` rightwards. The forwarding path from pod stdout into Splunk is owned by the platform/observability team; the pack's only assertion about it is `Kong - Log Ingestion Stalled`, which detects its failure (O3).

### 2.2 Event contract

The event is Kong 3.4's JSON log-serializer output. The fields this design depends on, and their validation status against the 3.4 serializer source, are in [log-format-validation.md](log-format-validation.md). The consumed subset:

| Object | Fields consumed |
|---|---|
| `response` | `status`, `size`, `headers.x-vault-namespace`, `headers.x-ratelimit-remaining`, `headers.x-ratelimit-limit` |
| `latencies` | `request`, `kong`, `proxy` |
| `route` | `name` |
| `service` | `name`, `host`, `read_timeout`, `connect_timeout`, `write_timeout`, `retries` |
| `tries{}` | `ip`, `balancer_latency` (multivalue, ordered) |
| top level | `started_at`, `upstream_uri`, `upstream_status`, `client_ip` |

**Notably absent:** the entire `request` object (`request.method`, `request.uri`, `request.id`, `request.size`, `request.tls.*`). This is [finding 1](log-format-validation.md#finding-1--the-request-object-is-missing-and-it-should-not-be) and it is a **real capability loss** the design works around rather than fixes:

| Lost capability | Design compensation |
|---|---|
| Filter or group by HTTP method | None. Accepted gap. |
| Correlate a client-reported request to a log line by request ID | None. Correlation is by time + client IP + path. |
| Distinguish the pre-rewrite path the client asked for | `upstream_uri` (post-rewrite) is used throughout as `upstream_path` |
| Request body size | Only `response.size` is available, as `resp_bytes` |

If `request.*` is restored upstream, §9 of [operations-guide.md](operations-guide.md) covers the additive changes; nothing in this design has to be undone.

### 2.3 Event identification

```
`kong_index` latencies.request=*
```

`latencies.request=*` is the discriminator. It is present on every log-serializer event and on **no other line in the same stdout stream** — nginx access logs, Kong startup output and Kong error logs all lack it.

**Rejected alternative: `route.name=*`.** Requests that match no route have a null `route`. Those are precisely Kong's own 404s — the events you need when someone reports "my endpoint disappeared". Filtering on `route.name=*` would silently discard the evidence for one of the eight triage symptoms ([S5](triage-guide.md#s5--404s-and-routing)). This is [gotcha 9](field-reference.md#gotcha-9--do-not-filter-on-routename-to-find-kong-events).

### 2.4 Timestamp handling

`started_at` is epoch **milliseconds**; `route.created_at` and `service.created_at` are epoch **seconds** — in the same event ([finding 6](log-format-validation.md#finding-6--mixed-timestamp-units-within-a-single-event)).

```
TIME_PREFIX = \"started_at\"\s*:\s*
TIME_FORMAT = %s%3N
MAX_TIMESTAMP_LOOKAHEAD = 20
```

Pointing the timestamp at a `created_at` field backdates every event to the date the Kong config entity was created — a failure mode that presents as "every event has the same timestamp" and is listed as such in the troubleshooting section of the operations guide.

### 2.5 Field extraction strategy

`props.conf` ships as **conditional, not required**. If a sourcetype is already assigned and dotted fields such as `latencies.request` already resolve in search, the file is not needed at all.

**Design decision: `KV_MODE = json` (search-time), not `INDEXED_EXTRACTIONS = json` (index-time).**

| | `KV_MODE` (chosen) | `INDEXED_EXTRACTIONS` (rejected) |
|---|---|---|
| Applies to already-indexed data | Yes | No — future events only |
| Deployment target | Search head | Forwarder / heavy forwarder |
| Fixes historical events | Yes | No |
| Re-index required | No | Effectively yes |

The data is already being indexed, so a setting that cannot fix history is not a candidate. **Never enable both** — every field double-extracts.

**No `FIELDALIAS` entries, deliberately.** Aliases can only rename. `kong_base` renames *and* normalises the `-1` sentinel and the four shapes of `upstream_status`. A second, dumber renaming layer would be one more thing to keep in sync, and would tempt searches to start from raw fields where the traps are still live.

### 2.6 Normalisation layer — `kong_base`

`kong_base` is the foundation. **Every dashboard panel and every alert is built on it.** It exists so the four Kong-specific data traps are solved exactly once.

```
[kong_base]
definition = `kong_index` latencies.request=*
| eval route           = coalesce('route.name',   "(no-route-matched)"),
       service         = coalesce('service.name', "(no-service)"),
       upstream_host   = coalesce('service.host', "(none)"),
       status          = tonumber('response.status'),
       kong_ms         = tonumber('latencies.kong'),
       total_ms        = tonumber('latencies.request'),
       short_circuited = if(tonumber('latencies.proxy') < 0, 1, 0),
       proxy_ms        = if(tonumber('latencies.proxy') < 0, null(), tonumber('latencies.proxy')),
       ...
```

Full field list: [field-reference.md §2](field-reference.md#2-what-kong_base-gives-you). The four normalisations that carry design weight:

**N1 — the `-1` latency sentinel.** `latencies.proxy = -1` is a *sentinel meaning "the upstream was never contacted"*, not a duration. Kong short-circuited: auth reject, rate limit, or no healthy target. `kong_base` nulls `proxy_ms` and exposes the condition separately as `short_circuited`.

> Consequence if unhandled: averaging or percentiling the raw column drags latency toward negative. **A failing gateway reports as faster than a healthy one.** This is the single most dangerous trap in the dataset, because the failure mode is silent and reassuring.

**N2 — `upstream_status` has four shapes.** A plain code (`200`), a comma-joined retry list (`"502, 200"`), a literal dash (`-`) when nginx reached no upstream, or absent when Kong short-circuited before the balancer ran.

```
| eval us_codes        = split(replace(coalesce('upstream_status', ""), " ", ""), ",")
| eval us_codes        = mvfilter(match(us_codes, "^[0-9]{3}$"))
| eval upstream_status = mvindex(us_codes, -1)
```

Collapsed to *either a valid 3-digit code or null*, taking the **last** element — the attempt that actually resolved the request. Downstream logic needs only `isnull()`.

**N3 — mixed timestamp units.** Handled at extraction (§2.4); `kong_base` does not re-derive `_time`.

**N4 — dotted field names.** `'response.status'` requires single quotes inside `eval`. `kong_base` renames to bare names once, so no downstream author has to remember (this is [gotcha 1](field-reference.md#gotcha-1--dotted-field-names-need-single-quotes-in-eval), and it fails *silently* — an unquoted dotted name evaluates as a field-minus-field arithmetic expression, not an error).

**The attribution field.** `error_source` is the design's central output, evaluated in strict precedence order:

```
| eval error_source = case(isnull(status),                   "unknown",
                           status < 400,                     "ok",
                           short_circuited == 1,             "kong",
                           isnull(upstream_status),          "kong",
                           tonumber(upstream_status) >= 400, "upstream",
                           true(),                           "kong")
```

Read the ordering as a decision tree:

| Condition | Verdict | Reasoning |
|---|---|---|
| No status at all | `unknown` | Malformed or truncated event; do not guess |
| Status < 400 | `ok` | Success, regardless of retries underneath |
| Short-circuited | `kong` | Kong decided, upstream never consulted |
| No upstream status | `kong` | Kong produced the error without a usable upstream response |
| Upstream returned ≥ 400 | `upstream` | Backend's error, faithfully passed through |
| Fallthrough | `kong` | Upstream succeeded but the client got an error ⇒ **Kong rewrote a success into a failure** (a plugin, typically) |

That final fallthrough is not a default — it is a genuine and easily-missed diagnosis. Kong received a 2xx from the upstream and returned an error anyway.

### 2.7 Population selection — two base macros

```
[kong_proxied]
definition = `kong_base` | where short_circuited == 0
```

**The rule: if the statistic describes the upstream, use `kong_proxied`.**

| Use case | Macro | Why |
|---|---|---|
| Request counts, error rates, status codes | `kong_base` | You want every request, including the ones Kong rejected |
| Anything over `proxy_ms` | **`kong_proxied`** | Short-circuited requests never reached the upstream |
| Error attribution | `kong_base` | `error_source` is precomputed; rejections are in scope |
| Kong overhead (`kong_ms`) | **`kong_proxied`** | Comparable population — short-circuits have a different overhead profile by construction |
| Per-target and balancer analysis | `kong_proxied` | A request with no upstream has no target |

Note the third row from the bottom: `kong_ms` uses `kong_proxied` even though `kong_ms` is always populated. This is deliberate — a window heavy with rate-limit rejections would otherwise shift the overhead distribution for reasons unrelated to gateway performance.

### 2.8 Topology discovery

Every Service and Route dropdown in all five dashboards reads one macro:

```
[kong_topology_source]
definition = `kong_base` earliest=-24h latest=now | stats count by service, route
```

**Design decision (DD-3): live bounded search, not KV Store, not CSV lookup.**

A KV Store collection can only be created by `collections.conf` (i.e. by deploying an app) or by a REST call. On a Splunk Cloud tenancy where app deployment is gated, neither is reliably available — and a missing collection breaks *every* dropdown with `collection ... does not exist`. A live search has no such dependency: it works the moment the macro is defined, on any Splunk, with no filesystem access and no app install. This removed a whole class of deployment and permission failure, plus a persisted artefact that would need reviewing for what it might contain.

**The two pinned time bounds are load-bearing.** An input population search carries no time range of its own. Without explicit `earliest`/`latest` this would run over **all time on every page load**, regardless of the page's own time picker.

**Accepted trade-offs:**

| Trade-off | Mitigation |
|---|---|
| One bounded 24h scan per dropdown per page load | Shorten to `-4h` on a very high-volume index; cost scales with the window |
| A route with no traffic in 24h is not listed | "All routes" still shows it in every table; a genuinely silent route is what `Kong - Route Traffic Stopped` exists to catch |
| Dropdowns are stale by up to the panel cache interval | Acceptable; topology changes are detected by alert 13, not by the dropdown |

**Degradation design:** if this macro breaks, dashboards still work. The static `All services` / `All routes` choices are declared in the XML and render regardless, so a failed topology source degrades to an *unfiltered* dashboard rather than an unusable one (principle 5).

### 2.9 Deployment portability surface — the five names

Three dashboards and two alerts are about a specific **role**, not about traffic in general: the service fronting Kong's Admin API is privileged, the Vault auth route has its own failure semantics, and the rate-limiting plugin is attached to one route.

Those names were previously hardcoded in **fourteen** places, which meant a deployment using different names got a blank Security dashboard and two alerts that silently never fired. They now live in three macros:

| Macro | Default | Consumed by |
|---|---|---|
| `kong_admin_service` | `service="admin-api"` | Security and Tenancy dashboard; alert 11 |
| `kong_auth_route` | `route="auth-route"` | Security and Tenancy dashboard; alert 10 |
| `kong_ratelimited_route` | `route="ee-smp-functests"` | Traffic and Rate Limiting dashboard |

Plus **two dashboard input defaults** that cannot be macros — Simple XML `<default>` cannot reference a macro:

| Location | Token | Default |
|---|---|---|
| `kong_security_tenancy.xml` | `svcfilter` | `admin-api` |
| `kong_security_tenancy.xml` | `rtfilter` | `auth-route` |

**Five places, not one — but down from fourteen, and all five are in one table** in [operations-guide.md §5](operations-guide.md#deploying-against-a-different-kong-install--the-five-names).

**Intended degradation:** if a role does not exist in a deployment, point the macro at something matching nothing (`service="__none__"`). Panels render empty and the alert never fires — empty rather than wrong.

### 2.10 Data-quality defects and their compensations

| # | Defect | Compensation | Residual risk |
|---|---|---|---|
| 1 | `request` object absent | None available | Accepted gap (§2.2) |
| 2 | `latencies.kong` on 3.4 **includes request receive time** | Documented; alert 5's threshold is fitted to the inflated value | A large slow request body inflates "gateway overhead". **Re-tune alert 5 on upgrade to 3.5+**, where receive time becomes a separate field |
| 3 | `latencies.proxy = -1` sentinel | N1 — nulled, exposed as `short_circuited` | Any *ad-hoc* search bypassing `kong_base` reintroduces it |
| 4 | `upstream_status` four shapes | N2 — normalised | None |
| 5 | `x-ratelimit-*` headers are **Vault's, not Kong's** | Kong throttling detected structurally (`429` + `short_circuited`); Vault quota panels physically separated and titled "NOT Kong's rate limiter" | Misreading remains possible; called out in the panel title, the runbook, and the "things not to do" table |
| 6 | Mixed timestamp units | Extraction pinned to `started_at` with `%s%3N` | None |
| 7 | `https_redirect_status_code: 475` is redaction noise | Not consumed | None |
| 8 | `client_ip` is `remote_addr` — may be a load balancer | Documented on the Top Talkers panel | **All client-attribution metrics are only as good as `real_ip_header`/`trusted_ips` in Kong.** If every request shows one or two IPs, client concentration and per-client alerting are meaningless. Verify before relying on §3 client metrics |
| 9 | Timeout/retry budget are per-service, not global | Read per-event from `service.read_timeout` / `service.retries` | None — this is strictly better than a constant |

---

## 3. Metrics catalogue

### 3.1 Taxonomy

Metrics are grouped into five families. The families map onto the alert patterns in §5.1 and onto the symptom index in the triage guide.

| Family | Question it answers | Primary population |
|---|---|---|
| **M1 Volume** | Is traffic arriving, from whom, at what shape? | `kong_base` |
| **M2 Correctness** | Are requests succeeding, and *whose fault* when not? | `kong_base` |
| **M3 Latency** | Where does the time go — Kong or the upstream? | `kong_proxied` |
| **M4 Resilience** | Is the gateway masking a failure that has not surfaced yet? | `kong_base` / `kong_proxied` |
| **M5 Security & tenancy** | Who is reaching privileged surfaces, and is auth healthy? | `kong_base` + role macros |

### 3.2 Metric register

Every metric consumed by a panel or an alert. "Pop." is the base macro; **`P` = `kong_proxied`**, **`B` = `kong_base`**.

#### M1 — Volume

| Metric | Definition | Pop. | Unit | Used by |
|---|---|---|---|---|
| `requests` | `stats count` | B | count | All 5 dashboards; alerts 1, 4, 5, 6, 8 as the ratio denominator |
| `requests_per_min_peak` | `max` of per-minute count | B | req/min | Traffic dashboard |
| `distinct_clients` | `dc(client_ip)` | B | count | Traffic, Security dashboards; alerts 2, 9, 10 |
| `bytes_out` | `sum(resp_bytes)` | B | bytes | Top Talkers |
| `share_pct` | `requests * 100 / eventstats sum(requests)` | B | % | Top Talkers, Client Concentration, Per-target health |
| `cumulative_pct` | `streamstats sum(requests) * 100 / total`, rank-ordered | B | % | Client Concentration |
| `paths` / `routes` per client | `dc(upstream_path)`, `dc(route)` | B | count | Top Talkers — a client fanning across many paths is a scanner signature |
| `topology_entities` | `stats count by service, route` over 24h | B | rows | All dropdowns (§2.8) |

#### M2 — Correctness and attribution

| Metric | Definition | Pop. | Unit | Used by |
|---|---|---|---|---|
| `status` | `tonumber('response.status')` | B | code | Everything |
| `status_class` | `1xx`…`5xx`/`unknown` bucket | B | class | Health Overview, Route Latency |
| `success_pct` | `sum(status<400) * 100 / count` | B | % | Health Overview KPI, health matrix, auth outcome, namespace health |
| `error_pct` | `sum(status>=400) * 100 / count` | B | % | Path tables, per-target health, Top Talkers |
| `c5xx` / `error_pct(5xx)` | `sum(status>=500) * 100 / count` | B | % | Health Overview KPI; **alert 1** |
| **`error_source`** | `ok`/`kong`/`upstream`/`unknown` (§2.6) | B | enum | Error attribution chart, error split pie, health matrix; **alert 1** (`caused_by_kong` / `caused_by_upstream`) |
| `short_circuited_pct` | `sum(short_circuited) * 100 / count` | B | % | Health Overview KPI; **alert 2** (as a count) |
| `upstream_status` | normalised last valid 3-digit code | B | code | Attribution, error tables, ticket-generation search |
| `auth_denied` | `sum(status==401 OR status==403)` | B | count | Security dashboard; **alerts 10, 11** |
| `throttled` | `sum(status==429)` | B | count | Traffic dashboard; **alert 9** |
| `enforced_by` | `if(short_circuited==1, "kong-plugin", "upstream-passthrough")` | B | enum | **Alert 9** — the field that stops a Vault 429 being read as a Kong 429 |

#### M3 — Latency

| Metric | Definition | Pop. | Unit | Used by |
|---|---|---|---|---|
| `proxy_ms` | Upstream wait; **null when short-circuited, never `-1`** | P | ms | Latency panels; **alert 4** |
| `kong_ms` | Kong's own overhead (**includes receive time on 3.4**) | P | ms | Overhead panels; **alert 5** |
| `total_ms` | End-to-end request duration | B | ms | Error tables, ticket search |
| `balancer_ms` | Last `tries{}.balancer_latency` — target selection time | P | ms | Balancer selection latency panel |
| `p50/p95/p99/max(proxy_ms)` | Percentiles over upstream wait | P | ms | Health Overview, Route Latency; **alert 4** |
| `p50/p95/max(kong_ms)` | Percentiles over gateway overhead | P | ms | Route Latency; **alert 5** |
| `latency_bucket` | 9 fixed buckets: `<10ms`, `10-50`, `50-100`, `100-250`, `250-500`, `0.5-1s`, `1-5s`, `5-30s`, `>30s` | P | histogram | Latency distribution — the shape a percentile hides (bimodality) |
| `read_timeout_ms` | `service.read_timeout`, **per-service from the event** | B | ms | Timeout-proximity panel; **alert 8** |
| `near_timeout` | `proxy_ms >= read_timeout_ms * 0.8` | P | 0/1 | Timeout-proximity panel; **alert 8** |

> **Design note.** Alert 5 pairs `p95_kong_ms` with `p95_upstream_ms` in the same result row, and the Route Latency dashboard charts "Kong overhead vs upstream wait" as a single stacked area. This is not decoration: the *contrast* is the diagnosis. High Kong overhead with normal upstream wait means the problem is inside the gateway — a conclusion no single-metric alert can reach.

#### M4 — Resilience

| Metric | Definition | Pop. | Unit | Used by |
|---|---|---|---|---|
| `try_count` | `mvcount('tries{}.ip')`, `0` if the balancer never ran | B | count | Balancer panels; **alert 6** |
| `retried` | `try_count > 1` | B | 0/1 | Retry rate KPI; **alert 6** |
| `retry_pct` | `sum(retried) * 100 / count` | B | % | **Alert 6** |
| `max_retries` | `service.retries`, **per-service from the event** | B | count | Retry budget table; **alert 6** |
| `retries_exhausted` | `try_count > max_retries` | B | 0/1 | **Alert 6** — the moment retries stop hiding the failure |
| `exhausted_and_failed` | `retries_exhausted==1 AND status>=500` | B | count | Retry budget table |
| `budget_used_pct` | `max(try_count) * 100 / (max_retries + 1)` | B | % | Retry budget table |
| `upstream_ip` | **Last** `tries{}.ip` — the target that *served* the request | B | IP | Per-target health; **alert 7** |
| `failed_first_try` | **First** `tries{}.ip` — the target that *failed* | B | IP | "Which target is causing the retries?" |
| `current_targets` | `dc(upstream_ip)` | P | count | Targets-in-rotation KPI; **alert 7** |
| `mins_since_last` | `(now() - max(_time)) / 60` per target | P | min | Per-target health — catches a pod that stopped receiving traffic without erroring |

> **The retry arithmetic, stated once.** `retries = 5` permits up to **6 attempts** (one initial + five retries). Exhaustion is therefore `try_count > max_retries`, and `budget_used_pct` divides by `max_retries + 1`. Off-by-one here would either never fire or fire constantly.

> **Why M4 is the highest-value family.** While retries succeed, users get `200`s and the error rate stays **completely flat**. The problem is invisible in every correctness metric. It becomes visible all at once when the budget runs out. `retries_exhausted` and `Upstream Target Dropped` are the two metrics that detect a degrading backend *before* it becomes an outage — which is objective O2.

#### M5 — Security and tenancy

| Metric | Definition | Pop. | Unit | Used by |
|---|---|---|---|---|
| Admin API `requests` / `distinct callers` | Count and `dc(client_ip)` scoped by `kong_admin_service` | B | count | Security KPIs; **alert 11** |
| Admin API `auth_denied` / `errors` | `401`/`403` count; `status>=400` count, per caller | B | count | Caller audit; **alert 11** |
| `paths_touched` | `values(upstream_path)` per Admin API caller | B | list | Caller audit — *what* the caller tried to reach |
| `first_seen` | `min(_time)` per caller, formatted | B | time | **Alert 11** — new caller vs long-standing one |
| `auth_op` | `k8s-login` / `token-renew` / `other`, matched on `upstream_path` | B | enum | Auth panels; **alert 10** |
| `denials` | `401`/`403` count on the auth route | B | count | **Alert 10** |
| `vault_ns` | `response.headers.x-vault-namespace` | B | string | Namespace traffic and health |
| `vault_remaining` / `vault_limit` | `x-ratelimit-remaining` / `-limit` — **Vault's quota, not Kong's** | B | count | Vault quota panels only (§2.10 defect 5) |

> **The `distinct_clients` cross-check is the security design.** `denials` alone cannot distinguish a broken workload from an attack. Alert 10 therefore always reports `distinct_clients` alongside: **one client generating most denials = a stale service-account token; many distinct clients failing at once = a Vault auth backend or Kubernetes issuer problem.** The escalation differs completely (§8.3), so the alert carries the field that decides it.

### 3.3 Metrics deliberately not used

| Not used | Why |
|---|---|
| `x-ratelimit-*` as a Kong rate-limit signal | They are Vault's. Kong throttling is detected structurally as `429` + `short_circuited` |
| Raw `latencies.proxy` | The `-1` sentinel makes every aggregate wrong (§2.10 defect 3) |
| `route.created_at` / `service.created_at` as a time basis | Epoch seconds; describes config creation, not the request |
| `https_redirect_status_code` | Redaction noise |
| A synthetic uptime/availability SLI | Would require a request-count denominator that includes requests Kong never logged (i.e. when Kong is down). Availability of the gateway itself is not observable from the gateway's own logs — see §5.9 |

---

## 4. Dashboard design

### 4.1 Dashboard set

Five dashboards, **62 titled panels**. The division is by *diagnostic question*, not by data source — each dashboard is the answer to one thing an engineer is trying to find out.

| Order | Dashboard | View ID | Answers | Panels |
|---|---|---|---|---|
| 1 (default) | Service Health Overview | `kong_service_health_overview` | Is anything wrong, and is it Kong or the upstream? | 11 |
| 2 | Route and Latency Analysis | `kong_route_latency` | Where is the time going, on which route and path? | 14 |
| 3 | Upstream and Balancer Health | `kong_upstream_balancer_health` | Which backend target is sick, and is Kong hiding it? | 12 |
| 4 | Traffic, Rate Limiting and Clients | `kong_traffic_rate_limiting` | Who is calling, how much, and who is being throttled? | 12 |
| 5 | Security and Tenancy | `kong_security_tenancy` | Who touched the Admin API, and is auth healthy? | 13 |

**Ordering is the triage path.** Dashboard 1 is `default="true"` because [the first 3 minutes of every incident](triage-guide.md#the-first-3-minutes-every-time) end at its error-attribution panel. Dashboards 2–5 are the four branches that panel sends you down.

### 4.2 Navigation design

```xml
<nav search_view="search" color="#5CC05C">
  <view name="kong_service_health_overview" default="true"/>
  ...
  <collection label="Alerts"><saved source="all" match="Kong -"/></collection>
  <collection label="Other views"><view source="unclassified"/></collection>
</nav>
```

Two deliberate mechanisms:

1. **The `Kong -` alert collection.** Every alert stanza is named `Kong - <something>`, so `match="Kong -"` surfaces all 13 in the app menu without listing them individually. **The naming convention is load-bearing** — an alert renamed without the prefix vanishes from the menu.
2. **The `unclassified` safety net.** `<view name="...">` matches on the **view ID** — the XML filename without extension — *not* the `<label>` shown in the UI. Creating a dashboard through Splunk Web derives that ID from whatever title was typed, so `Kong Proxy - Traffic, Rate Limiting and Clients` becomes `kong_proxy___traffic__rate_limiting_and_clients` and no longer matches. The dashboard still works and is still listed under Dashboards — **it just silently vanishes from this menu.** The `unclassified` collection renders any view not explicitly named above, so a drifted ID appears under "Other views" instead of disappearing. A dashboard showing up there *is the diagnostic signal* that its ID does not match.

### 4.3 Common structure and token model

Every dashboard is a `<form version="1.1">` with a fieldset, and every dashboard follows the same vertical grammar:

```
fieldset  →  time range + scope dropdowns + granularity
row 1     →  single-value KPI strip (4–5 tiles), colour-banded
row 2     →  the attributing chart, usually preceded by an <html> explainer
rows 3+   →  breakdown charts, then detail tables (widest scope → narrowest)
```

**Token conventions:**

| Token | Type | Purpose | Default |
|---|---|---|---|
| `tr` | time | Page time range; every panel binds `$tr.earliest$` / `$tr.latest$` | per dashboard |
| `svc` | dropdown | Service scope, populated by `kong_topology_source` | `*` |
| `rt` | dropdown | Route scope, populated by `kong_topology_source` | `*` |
| `span` | dropdown | Chart granularity for `timechart span=$span$` | `5m` or `1h` |
| `svcfilter` / `rtfilter` | dropdown | Security dashboard role scope (§2.9) | `admin-api` / `auth-route` |

Three implementation rules that are easy to get wrong:

1. **Scope is applied as `| search $svc$ | search $rt$`**, appended after the base macro rather than interpolated into it. This keeps `kong_base` a single unparameterised definition and makes `*` a valid no-op value.
2. **Every dropdown declares static choices in addition to the dynamic ones** — `All services`, `All routes`, and the explicit `(no-service)` / `(no-route-matched)` sentinels. The static choices are what make a failed topology source degrade gracefully (§2.8); the sentinels are what make Kong's own 404s selectable.
3. **A literal `$` in panel SPL must be escaped as `$$`.** The Security dashboard's regex anchors appear as `match(upstream_path, "/auth/kubernetes/login$$")` — a single `$` would be parsed as a token delimiter and silently break the match.

**Granularity choices differ by dashboard on purpose:** `1m/5m/1h` where sub-minute bursts matter (Route Latency, Balancer Health) and `5m/1h/1d` where trend matters more than burst (Traffic, Security).

### 4.4 Per-dashboard design

#### D1 — Service Health Overview *(the triage entry point)*

| Row | Content |
|---|---|
| KPI strip | Requests · Success rate · 5xx rate · **Short-circuited %** · Upstream p95 |
| Attribution | **Error attribution over time** (column, `error_source`) + Error split (pie), with an `<html>` explainer defining `kong` vs `upstream` |
| Detail | Service and route health matrix — the only table in the pack with `drilldown=cell` |
| Trend | Traffic by status class · Upstream targets in rotation |
| Detail | Top failing paths (4xx and 5xx) |

The KPI strip is chosen so that **each tile maps to a different triage branch**: 5xx rate → S1, upstream p95 → S2, short-circuited % → S1/S6. "Short-circuited %" earns a top-level tile despite being an unusual metric because it is the fastest possible read on "did we even reach the backend".

#### D2 — Route and Latency Analysis

KPI strip (Requests · Success rate · Upstream p95 · **Kong overhead p95** · Retry rate) → status codes over time → **upstream latency percentiles** and **Kong overhead vs upstream wait** side by side → **latency distribution histogram** and **timeout proximity** → paths by volume and latency · top clients → **Kong overhead by route (plugin cost)** · recent errors.

Two panels carry design weight beyond their obvious use:

- **Latency distribution (9 buckets)** exists because percentiles hide bimodality. A p95 of 2s can mean "everything is 2s" or "95% are 10ms and 5% are 30s" — completely different problems, indistinguishable from a percentile, obvious in the histogram.
- **Kong overhead by route** is how plugin cost is attributed. Kong overhead is a *per-route* property because plugins are attached per route; a fleet-wide overhead number cannot tell you which plugin is expensive.

#### D3 — Upstream and Balancer Health

KPI strip (Targets in rotation · Retried requests · Retry rate · **Retries exhausted** · Requests that reached no upstream) → **retry budget consumption** (full-width, with explainer) → distinct targets over time → **per-target health** (full-width) → latency by target · retries over time → **which target is causing the retries?** · balancer selection latency.

This is the M4 dashboard, and it is built around one insight: **retries make a sick backend invisible.** The layout therefore leads with budget consumption rather than with error rate, and the "which target is causing the retries?" panel inverts the usual view — it groups by `failed_first_try` (the *first* entry in `tries{}`) rather than by `upstream_ip` (the *last*). A pod appearing repeatedly as `failed_first_try` while rarely appearing as `upstream_ip` is the sick one.

`mins_since_last` in per-target health catches the complementary failure: a pod that stopped receiving traffic entirely without ever returning an error.

#### D4 — Traffic, Rate Limiting and Clients

KPI strip (**Kong-enforced 429s** · 429s on the rate-limited route · Total requests · Distinct clients · Peak req/min) → Kong rate limiting over time · 429s by route and client → traffic by route → **top talkers** · **client concentration** → *(visually separated)* **Vault quota headroom — NOT Kong's rate limiter** · Vault quota by namespace.

**The physical separation of the Vault quota row is a design requirement, not a layout preference.** §2.10 defect 5 is the easiest mistake in this dataset to make; the mitigation is a titled boundary, an explanatory `<html>` block, and the words "NOT Kong's rate limiter" in the panel title.

**Client concentration** answers a blast-radius question rather than a health question: a single client above 50% of traffic means its retry storm becomes everyone's outage. It is the only panel whose value is purely architectural risk.

#### D5 — Security and Tenancy

Two logical halves plus a tenancy section:

| Half | Content |
|---|---|
| **Admin API** (privileged surface) | Calls · Distinct callers · Non-2xx · **Auth rejections** → **caller audit** · paths touched → activity over time |
| **Authentication** (auth route) | Auth operations over time · auth failures over time → auth outcome by operation · **clients failing authentication** |
| **Tenancy** | Traffic by Vault namespace → namespace health |

The Admin API half treats **every caller as privileged**, because the Admin API exposes Kong's control plane through the data plane — anyone reaching it can rewrite routing. The caller audit table is therefore keyed on `client_ip` with `first_seen` and `paths_touched`, i.e. structured as an *audit record* rather than as a metric.

### 4.5 Visual threshold design (colour bands)

Dashboard colour bands are a **separate tuning surface from alert thresholds** and are deliberately *tighter* — a dashboard should show concern before an alert pages someone. They are declared as `rangeValues` on single-values and `<scale type="threshold">` on table fields.

**Single-value bands** (green → yellow → orange → red, or inverted for "higher is better"):

| Panel | Bands | Direction |
|---|---|---|
| Success rate (D1, D2) | `95, 99, 99.5` | higher better |
| 5xx rate (D1) | `0.5, 2, 5` | lower better |
| Short-circuited % (D1) | `1, 5, 15` | lower better |
| Upstream p95 (D1, D2) | `500, 2000, 10000` ms | lower better |
| Kong overhead p95 (D2) | `50, 100, 500` ms | lower better |
| Retry rate (D2) | `0.5, 2, 10` % | lower better |
| Retry rate (D3) | `0.1, 1, 5` % | lower better |
| Retried requests (D3) | `1, 50` | lower better |
| Retries exhausted (D3) | `1, 10` | lower better |
| Targets in rotation (D3) | `1, 2` | higher better |
| Kong-enforced 429s (D4) | `1, 100` | lower better |
| Distinct Admin API callers (D5) | `3, 10` | lower better |
| Admin API non-2xx (D5) | `1, 20` | lower better |
| Admin API auth rejections (D5) | `1, 5` | lower better |

**Table field thresholds:**

| Field | Threshold | Meaning at first band |
|---|---|---|
| `error_pct` (paths, per-target) | `1` | any error rate above 1% is coloured |
| `p95_ms` (paths) | `2000, 10000` | aligned with alert 4 |
| `p95_kong_ms` (routes) | `50, 200` | aligned with alert 5's floor |
| `budget_used_pct` | `50, 100` | 100% = exhaustion |
| `exhausted_requests`, `exhausted_and_failed` | `1` | **any** non-zero value is red |
| `c5xx`, `err_kong` | `1` | **any** non-zero value is red |
| `mins_since_last` | `10, 30` | a target silent for 30 min |
| `share_pct` (clients, targets) | `25, 50` | blast-radius concentration |
| `cumulative_pct` (clients) | `80` | 80% of traffic from the listed clients |
| `success_pct` (auth, namespace) | min/mid/max `80 / 97 / 100` | continuous gradient, not bands |

**Note the deliberate asymmetry:** D2's retry-rate tile uses `0.5, 2, 10` while D3's uses `0.1, 1, 5`. D3 is the dashboard someone opens *because* they suspect the balancer, so it is tuned to show detail an order of magnitude earlier. Any of these bands may be re-fitted alongside the alert thresholds in §6; three (`p95_ms`, `p95_kong_ms`, `budget_used_pct`) are intentionally aligned to alert thresholds and **should be re-fitted together with them** so the dashboard and the alert never disagree about what "bad" means.

---

## 5. Alerting design

### 5.1 Detection patterns

The 13 alerts are instances of five patterns. Recognising the pattern is how you decide whether a *new* alert belongs in this pack and how it should be built.

| Pattern | Mechanism | Detects | Alerts |
|---|---|---|---|
| **P1 Ratio breach** | Aggregate over a window, compute %, compare to a fitted threshold, **guarded by a minimum volume** | Degradation proportional to traffic | 1, 6 |
| **P2 Percentile breach** | Percentile over a window against a fitted threshold, volume-guarded | Latency regression | 4, 5 |
| **P3 Leading indicator** | A condition that is **not yet a user-visible failure** | Failure before it surfaces — objective O2 | 6 (`exhausted`), 7, 8 |
| **P4 Absence / structural** | A condition that is true or false, no statistics, **no threshold to tune** | Silent failures that generate no errors at all | 2, 3, 7, 11, 12, 13 |
| **P5 In-search baseline** | Compare the recent window to a prior window **inside the same search**, no stored state | Change relative to normal, with nothing to install | 3, 7, 13 |

**P4 and P5 are the patterns that make this pack useful.** A ratio or percentile alert can only fire when something is already going wrong at volume. The absence patterns catch the incidents that produce *no errors and no latency*: a route deleted from config, a pod that left the pool, a log pipeline that broke. Those look identical to perfect health in every P1/P2 metric.

**P5, stated precisely.** The baseline is computed by `append`-ing a subsearch over a prior window and reconciling with an outer `stats`:

```
`kong_base`                                             ← current window
| stats count as current by route
| append [ search `kong_base` earliest=-25h@h latest=-1h@h    ← baseline window
           | bin _time span=15m
           | stats count as c by _time, route
           | stats avg(c) as baseline_per_15m by route ]
| stats sum(current) as current, sum(baseline_per_15m) as baseline_per_15m by route
| fillnull value=0 current
| where baseline_per_15m >= 20 AND current == 0
```

Two non-obvious properties:

1. **The subsearch supplies the route name as well as the baseline.** A route with zero events in the current window produces *no row at all* in the outer search — so without the subsearch there would be nothing to compare against, and zero-traffic would be undetectable. This is the whole trick.
2. **The baseline window is `-25h@h` to `-1h@h`, not `-24h` to `now`.** Excluding the last hour stops an in-progress outage from depressing its own baseline.

The `fillnull value=0 current` is what converts "absent from the current window" into "zero", making the `current == 0` comparison possible.

**Subsearch limits.** Splunk caps subsearches (default 50k rows / 60s). Every subsearch here is a `stats by` returning one row per entity — single digits for a typical Kong install — so the cap is not a practical concern. **On a deployment with hundreds of routes, lengthen the window rather than raising the cap.**

### 5.2 Alert catalogue

All 13 alerts, in `savedsearches.conf`. Splunk has no `alerts.conf` — **an alert *is* a saved search carrying `alert.*` properties**, which is why they live in this file ([finding 8](log-format-validation.md#finding-8--splunk-has-no-alertsconf)).

| # | Alert | Sev | Pattern | Pop. | Schedule | Window | Suppress | Fires when |
|---|---|---|---|---|---|---|---|---|
| 1 | Service 5xx Rate Breach | High (4) | P1 | B | `*/5 * * * *` | 5m | 30m | `error_pct > 2` and `requests >= 50` |
| 2 | Upstream Unreachable | **Crit (5)** | P4 | B | `1-59/5 * * * *` | 5m | 15m | `short_circuited==1 AND status>=500`, `failures >= 10` |
| 3 | Route Traffic Stopped | High (4) | P4+P5 | B | `2-59/5 * * * *` | 15m | 60m | `baseline_per_15m >= 20 AND current == 0` |
| 4 | Upstream Latency p95 Breach | Med (3) | P2 | **P** | `3-59/5 * * * *` | 10m | 30m | `p95(proxy_ms) > 2000` and `requests >= 50` |
| 5 | Gateway Self-Latency Breach | High (4) | P2 | **P** | `4-59/5 * * * *` | 10m | 30m | `p95(kong_ms) > 100` and `requests >= 50` |
| 6 | Balancer Retry Storm | Med (3) | P1+P3 | B | `5-59/10 * * * *` | 10m | 30m | `retry_pct > 5` **OR** `exhausted_requests > 0` |
| 7 | Upstream Target Dropped | Med (3) | P3+P4+P5 | **P** | `7,22,37,52 * * * *` | 15m | 60m | `current_targets < baseline_targets` (24h baseline) |
| 8 | Approaching Read Timeout | Med (3) | P3 | **P** | `8-59/10 * * * *` | 10m | 30m | any `proxy_ms >= read_timeout_ms * 0.8` |
| 9 | Rate Limit Rejections Spike | Low (2) | P1 | B | `9-59/10 * * * *` | 10m | 60m | `throttled >= 20` |
| 10 | Authentication Failure Spike | High (4) | P1 | B | `6-59/10 * * * *` | 10m | 30m | `denials >= 25` on the auth route |
| 11 | Admin API Unexpected Activity | **Crit (5)** | P4 | B | `*/10 * * * *` | 10m | 30m | `auth_denied > 0` **OR** `errors >= 5` |
| 12 | Log Ingestion Stalled | High (4) | P4 | `kong_index` | `*/5 * * * *` | 10m | 60m | `events == 0` |
| 13 | New Route Or Service Detected | Med (3) | P4+P5 | B | `27 */6 * * *` | 6h | 120m | Traffic in last 6h, none in preceding 7d |

Severity scale is Splunk's: **1 Info · 2 Low · 3 Medium · 4 High · 5 Critical**.

**Design notes on individual alerts:**

- **A1** splits `caused_by_kong` from `caused_by_upstream` in the result row so **the email itself says which team owns the problem** (§7.2). This is the single highest-value design detail in the alerting layer.
- **A2** is separate from A1 rather than a variant of it because `short_circuited + 5xx` is a *different failure with a different fix* — pod down, DNS, network policy — not an application error. Merging them would produce an alert whose runbook had to branch immediately.
- **A5** carries `p95_upstream_ms` **for contrast**: normal upstream wait with high Kong overhead localises the fault inside the gateway.
- **A6** has a compound condition on purpose. `exhausted_requests > 0` is a **P3 structural** signal and outranks the percentage — the runbook instructs reading it first, because non-zero means the failure already reached a user.
- **A7** is expected during deliberate scale-down or rolling deploys. **Suppress during planned changes; do not disable** (§10.4). Escalate immediately if `current_targets` is 1 — every request now depends on one pod.
- **A8** reads the timeout **per service from the event**, so only the `0.8` fraction is tunable. It is the cheapest alert to act on: nothing is broken for users yet.
- **A11** ships with no allowlist and an explicit extension point — add a `NOT client_ip IN (...)` clause for known CI/CD and operator IPs. **Attribute every current caller before enabling** (§10.1), or the first firing is an unattributable IP at 2am with no baseline.
- **A12** works because `stats count` **with no `BY` clause always emits exactly one row, including when the search matched nothing.** That is what makes "zero events" a detectable condition rather than an empty result set — an ordinary search returning no rows cannot trigger a `> 0` alert. It is also the only alert that runs on `kong_index` rather than `kong_base`, because a pipeline break must be detected even if the events are malformed.
- **A13** runs 6-hourly rather than hourly because its subsearch scans **7 days**. A Kong config change does not need sub-hourly detection, and a 7-day scan every hour would be a poor trade.

### 5.3 Trigger model

Uniform across all 13 alerts:

```
alert_type = number of events
alert_comparator = greater than
alert_threshold = 0
```

**The search returns rows only when the condition is breached**, so the trigger is always "Number of Results is greater than 0". No custom trigger conditions anywhere.

**Why:** the alerting logic lives entirely in SPL, where it can be run in the search bar, tested against history, and read in one place. Splitting logic between a search and a trigger expression creates a condition that cannot be reproduced by running the search — the worst property an alert can have.

`counttype`, `relation` and `quantity` are also set on every stanza. They are the **legacy equivalents** of the three properties above, included for compatibility with older Splunk versions. The UI shows only one set; they must be kept consistent if edited.

`alert.track = 1` on every alert, so firings are recorded in the Triggered Alerts view and post-incident review can ask "did this fire and get ignored?".

### 5.4 Scheduling design

Schedules are staggered across minute offsets so that thirteen scheduled searches do not all dispatch simultaneously:

| Offset family | Alerts | Minutes |
|---|---|---|
| `*/5` | 1, 12 | 0, 5, 10, … |
| `1-59/5` … `4-59/5` | 2, 3, 4, 5 | 1,6,11… / 2,7,12… / 3,8,13… / 4,9,14… |
| `5-59/10` … `9-59/10` | 6, 10, 8, 9 | 5,15,25… / 6,16,26… / 8,18,28… / 9,19,29… |
| `*/10` | 11 | 0, 10, 20, … |
| explicit list | 7 | 7, 22, 37, 52 |
| 6-hourly | 13 | `:27` past every 6th hour |

**Accuracy note on the stagger.** Each `/5` alert occupies **two** minute-offsets mod 10 (`1-59/5` fires at 1, 6, 11, 16 … i.e. offsets 1 *and* 6). The five `/5` alerts (1–5) therefore occupy **all ten offsets between them**, so every `/10` alert unavoidably shares its minute with one of them — no choice of offset avoids this. The realistic worst case is **three concurrent dispatches**:

| Minute | Alerts dispatching |
|---|---|
| 0, 10, 20, 30, 40, 50 | 1 (`*/5`), 11 (`*/10`), 12 (`*/5`) |
| 5, 15, 25, 35, 45, 55 | 1, 12, 6 (`5-59/10`) |
| all others | at most 2 |

Three concurrent scheduled searches is well within a normal search-head concurrency budget, so **no change is warranted** — but the pack's own comment that the stagger prevents same-minute dispatch is stronger than what the crons achieve (§13 errata E4). If concurrency ever did become a constraint, the only lever that helps is **moving a `/5` alert onto a `/10` schedule** — re-offsetting the `/10` alerts cannot help while all ten offsets are occupied. That lever is not free: alerts 1 and 12 run every 5 minutes against 5- and 10-minute windows deliberately (§5.5), so halving their frequency **doubles detection latency on the 5xx breach and on ingestion stall**. Accept the three concurrent dispatches instead unless the search head is genuinely saturated.

### 5.5 Evaluation window vs schedule

Several alerts run more frequently than their window is long, so consecutive runs **overlap**:

| Alert | Schedule | Window | Overlap |
|---|---|---|---|
| 3 | 5 min | 15 min | 3× |
| 12 | 5 min | 10 min | 2× |
| 6, 8, 9, 10 | 10 min | 10 min | none |
| 1, 2 | 5 min | 5 min | none |

Overlap is **deliberate**: it shortens detection latency without shortening the window (which would make the statistic noisier). The cost is that the same condition is evaluated repeatedly, which would produce duplicate notifications — **suppression is what absorbs that** (§5.6). Overlap and suppression are a matched pair; changing a window or a schedule without re-checking the suppression period reintroduces duplicates.

All windows are snapped (`-5m@m`, `-10m@m`, `-6h@h`) so runs cover contiguous, non-drifting intervals.

### 5.6 Suppression design

Every alert uses **time-based suppression**:

```
alert.suppress = 1
alert.suppress.period = <15m | 30m | 60m | 120m>
```

Periods are chosen against expected mean time to repair, not arbitrarily:

| Period | Alerts | Rationale |
|---|---|---|
| 15m | 2 | Critical and fast-moving; re-notify quickly if unresolved |
| 30m | 1, 4, 5, 6, 8, 10, 11 | Typical investigate-and-mitigate cycle |
| 60m | 3, 9, 12 | Slower-moving or lower-urgency conditions |
| 120m | 13 | Config changes arrive in batches during a deployment |

**The trade-off, stated plainly.** Suppression is **global to the search**. Once `Kong - Service 5xx Rate Breach` fires for `vault-svc`, a breach on `admin-api` inside the same 30 minutes is **also suppressed**.

That is the chosen default because it **cannot storm**. The alternative, per alert:

```
alert.digest_mode = 0
alert.suppress.fields = service
```

which yields one email per breaching row, throttled independently per service — louder, but nothing is masked. **Decide per alert. Alert 11 (Admin API) is the usual candidate for per-result mode**, since two different unrecognised callers are two different security events and masking the second is unacceptable.

### 5.7 Digest mode

`alert.digest_mode = 1` on all 13: **one notification per firing, with every breaching row inline**. Combined with `action.email.format = table`, an engineer receiving a 5xx alert covering three services sees all three in one email with the attribution columns, rather than three emails.

### 5.8 Coverage matrix — failure mode to detection

The design's coverage claim, made explicit. This is the table to audit when someone asks "would we have caught that?".

| Failure mode | Visible as | Detected by | Latency to detect |
|---|---|---|---|
| Upstream returns 5xx | `error_source=upstream` | A1 | ≤ 5 min |
| Kong cannot reach upstream at all | `short_circuited + 5xx` | **A2** | ≤ 5 min |
| Upstream pod slow | `proxy_ms` p95 rise | A4, A8 | ≤ 10 min |
| Kong plugin slow / datastore behind rate limiter slow | `kong_ms` p95 rise, `proxy_ms` normal | **A5** | ≤ 10 min |
| One backend pod sick, others absorbing | **no error, no latency change** | **A6** (retry storm), A7 | ≤ 10 min |
| Retry budget exhausted, failure reaches user | 5xx after N attempts | **A6** (`exhausted`) | ≤ 10 min |
| Pod silently left the pool | **no symptom at all** | **A7** | ≤ 15 min |
| Approaching timeout, not yet failing | **no symptom at all** | **A8** | ≤ 10 min |
| Route deleted from Kong config | **traffic ceases, no errors** | **A3** | ≤ 15 min |
| Client died / DNS changed upstream of gateway | **traffic ceases, no errors** | **A3** | ≤ 15 min |
| Kong rate limiter rejecting | `429 + short_circuited` | A9 | ≤ 10 min |
| Stale service-account token on one workload | 401/403, one client | A10 (`distinct_clients` low) | ≤ 10 min |
| Vault auth backend / K8s issuer broken | 401/403, many clients | A10 (`distinct_clients` high) | ≤ 10 min |
| Credential stuffing | 401/403, many clients, varied targets | A10 → **Security** (§8.3) | ≤ 10 min |
| Unauthorised Admin API access attempt | any 401/403 on the Admin service | **A11** | ≤ 10 min |
| Unexpected route to a privileged service | new entity carrying traffic | **A13** | ≤ 6 h |
| Log pipeline broken | **healthy-looking silence** | **A12** | ≤ 5 min |
| Kong config deployed | new entity carrying traffic | A13 | ≤ 6 h |
| Requests matching no route (endpoint "disappeared") | Kong 404s, null route | *Dashboards only* — see §5.9 | — |
| Single client dominating traffic (blast radius) | concentration | *Dashboards only* — see §5.9 | — |

### 5.9 Deliberate non-alerts and residual blind spots

Design honesty requires naming what this pack does **not** detect.

**Deliberate non-alerts** — visible on dashboards, intentionally not alerted:

| Condition | Why no alert |
|---|---|
| 404 / no-route-matched rate | Normal baseline varies enormously by deployment (scanners, health checks, bad clients). Any fixed threshold would be noise. Investigated reactively via [S5](triage-guide.md#s5--404s-and-routing) |
| Client concentration above 50% | An architectural risk, not an incident. Belongs in a design review, not a page |
| 4xx rate generally | Mostly client behaviour. Only the security-relevant subsets (401/403, 429) are alerted |
| Vault quota headroom | A property of the upstream, not the gateway. Vault should alert on its own quota |
| Namespace-level health | Tenancy reporting; no single threshold is meaningful across namespaces |

**Residual blind spots** — genuinely not covered:

| Blind spot | Consequence | Mitigation |
|---|---|---|
| **Kong wholly down** | No logs are produced, so this is indistinguishable from "no traffic" *from log data alone*. A12 fires, but says "ingestion stalled", not "gateway down" | A12's runbook and [S7](triage-guide.md#s7--no-data-in-splunk-at-all) **both begin by curling the gateway directly.** Log-based monitoring cannot self-diagnose its own subject's absence — an external synthetic check is the correct fix and is out of scope for this pack |
| **No request-level correlation ID** | Cannot join a client-reported failure to a specific log event | Correlate by time + `client_ip` + `upstream_path`. Resolved if `request.id` is restored (§2.2) |
| **HTTP method invisible** | Cannot alert on, say, unexpected `DELETE` to the Admin API | `paths_touched` in A11 is the partial substitute. Resolved if `request.*` is restored |
| **`client_ip` may be a load balancer** | Every client-attribution metric and A10/A11's `distinct_clients` cross-check may be meaningless | **Verify `real_ip_header`/`trusted_ips` in Kong before relying on client attribution** (§2.10 defect 9). This is a prerequisite, not a caveat |
| **Overnight/weekend quiet periods** | A12 may fire on genuine quiet, and A3's baseline is diluted | For A12, **widen the window or schedule it business-hours-only — do not lower its severity.** A12 at reduced severity is a blind spot that looks like coverage |
| **`latencies.kong` includes receive time on 3.4** | A5 conflates gateway overhead with slow client uploads | Threshold fitted to the inflated value; **re-tune on 3.5+** (§2.10 defect 2) |
| **No alert on `error_source=unknown`** | Malformed events accumulate unnoticed | Visible in the attribution chart. Worth adding if it is ever non-trivial — see §13 open items |

---

## 6. Threshold design

### 6.1 Classification

Every threshold is one of three kinds, and the kind determines whether it needs fitting:

| Kind | Meaning | Needs baseline fitting? |
|---|---|---|
| **Structural** | Derived from the mechanism, not from traffic. `current == 0`, `exhausted > 0`, `events == 0` | **No.** Do not "tune" these — a structural threshold that has been softened has been disabled |
| **Statistical** | A percentage or percentile that only means something relative to this deployment's normal | **Yes.** These are the reason everything ships disabled |
| **Data-derived** | Read per-event from the log; not configurable at all | **No.** `read_timeout_ms`, `max_retries` |

This classification is why the recommended enable order works: the **structural** alerts (12, 2, 11) are safe on day one because their thresholds cannot be wrong.

### 6.2 Complete threshold register

The full tuning surface. **One shared constant is a macro because four searches use it; every other threshold is inline in the `where` clause of the alert that owns it**, so it can be read in context rather than looked up.

| Threshold | Default | Kind | Location | Derivation |
|---|---|---|---|---|
| Minimum request volume for ratio alerts | `50` | Statistical | **macro** `kong_min_requests` | ≈ half of `avg_requests_per_5m` for your **quietest** service |
| 5xx breach percentage | `2` % | Statistical | A1 `where error_pct > 2` | **Above** observed p99 |
| Short-circuited 5xx count | `10` | Statistical | A2 `where failures >= 10` | Low absolute count; near-zero baseline expected |
| Traffic-stopped baseline floor | `20` per 15m | Statistical | A3 `where baseline_per_15m >= 20` | Above the quietest bucket of routes you want covered |
| Upstream p95 | `2000` ms | Statistical | A4 `where p95_ms > 2000` | ≈ observed `upstream_p99` |
| Kong self-latency p95 | `100` ms | Statistical | A5 `where p95_kong_ms > 100` | ≈ observed `kong_p99`, **floor 50 ms** |
| Retry rate | `5` % | Statistical | A6 `where retry_pct > 5` | Well **below** 5% if baseline retries are ≈ 0 |
| Retry exhaustion | any `> 0` | **Structural** | A6 `OR exhausted_requests > 0` | Mechanism: the budget ran out |
| Target dropped | fewer than baseline | **Structural** | A7 — no threshold | Mechanism: 24h in-search baseline |
| Timeout proximity fraction | `80` % | Statistical | A8 `proxy_ms >= read_timeout_ms * 0.8` | **The timeout itself is data-derived** — only the fraction is tunable |
| 429 burst | `20` | Statistical | A9 `where throttled >= 20` | Above normal throttle volume for the limited route |
| Auth denial burst | `25` | Statistical | A10 `where denials >= 25` | Above normal 401/403 volume on the auth route |
| Admin API error count | `5` | Statistical | A11 `where auth_denied > 0 OR errors >= 5` | The `auth_denied > 0` half is **structural** |
| Ingestion stall | `events == 0` | **Structural** | A12 | Mechanism: zero events |
| New route/service | absent for 7d | **Structural** | A13 | Mechanism: in-search 7-day baseline |
| Upstream read timeout | *per service* | **Data-derived** | `service.read_timeout` | Not tunable — and correct across services with different settings |
| Retry budget | *per service* | **Data-derived** | `service.retries` | Not tunable |

The last two rows previously existed as configurable constants and were **removed on purpose**. Hardcoding `read_timeout_ms = 60000` was wrong across services: one configured with a 10s timeout would have had to reach 48s before A8 considered it slow — long after it had already timed out.

### 6.3 Derivation methodology

**Everything ships `disabled = 1`.** The defaults come from the Kong config (`read_timeout 60000`, `retries 5`) and general gateway norms — **not from this deployment's traffic.** Enabling 13 alerts against unfitted thresholds is how a monitoring rollout gets muted in its first fortnight (objective O7).

**Run every baseline over a full week** (`earliest=-7d@d latest=@d`). Weekday and weekend profiles differ enough that a weekday-only sample misfires on Monday.

The governing rule: **a statistical threshold should mean "worse than the worst normal week", not "slightly unusual".** Fit above p99, not above p95.

#### Baseline queries

**5xx percentage and `kong_min_requests`:**

```spl
`kong_base`
| bin _time span=5m
| stats count as requests, sum(eval(if(status>=500,1,0))) as errors by _time, service
| eval error_pct = round(errors*100/requests, 2)
| stats count as buckets, avg(requests) as avg_requests_per_5m,
        perc50(error_pct) as p50, perc95(error_pct) as p95,
        perc99(error_pct) as p99, max(error_pct) as worst
        by service
```

Set A1 above `p99`. Set `kong_min_requests` to roughly half of `avg_requests_per_5m` for the quietest service.

**Latency thresholds (A4, A5):**

```spl
`kong_proxied`
| stats perc50(proxy_ms) as upstream_p50, perc95(proxy_ms) as upstream_p95,
        perc99(proxy_ms) as upstream_p99, max(proxy_ms) as upstream_max,
        perc50(kong_ms)  as kong_p50,     perc95(kong_ms)  as kong_p95,
        perc99(kong_ms)  as kong_p99,     max(kong_ms)     as kong_max
        by service
```

A4 ≈ `upstream_p99`. A5 ≈ `kong_p99`, **with a 50 ms floor** — Kong overhead is normally single-digit milliseconds and a threshold that tight fires on ordinary jitter.

**Retry rate (A6):**

```spl
`kong_base`
| bin _time span=10m
| stats count as requests, sum(retried) as retried by _time, service
| eval retry_pct = round(retried*100/requests, 3)
| stats perc50(retry_pct), perc95(retry_pct), perc99(retry_pct), max(retry_pct) by service
```

If baseline retries are near zero, **drop the threshold well below 5%.** On a healthy cluster any sustained retry rate is a signal, and 5% would already be a serious incident by the time it fired.

**Traffic-stopped baseline (A3):**

```spl
`kong_base`
| bin _time span=15m
| stats count as c by _time, route
| stats min(c) as quietest, perc05(c) as p05, avg(c) as mean, max(c) as busiest by route
```

Any route whose `quietest` bucket is already 0 will never alert with the default guard — **that is intended**. To cover a genuinely low-volume route, lower the `baseline_per_15m >= 20` guard and accept the noise.

**Admin API caller attribution — required before enabling A11:**

```spl
`kong_base` | where service=="admin-api"
| stats count, values(status) as codes, min(_time) as first, max(_time) as last by client_ip
| eval first=strftime(first,"%F %T"), last=strftime(last,"%F %T")
```

Attribute **every** IP to a known workload and add unexplained ones to the investigation list *before* enabling.

### 6.4 The volume guard

`kong_min_requests = 50` is a macro rather than an inline constant because four alerts use it (1, 4, 5, 6) — the stated rule being that **a constant earns a macro when it is used in more than one place.**

Its purpose: without it, **one failed request in a quiet window is a 100% error rate and pages someone at 3am.** Every ratio and percentile alert applies it *before* computing the ratio:

```
| where requests >= `kong_min_requests`
| eval error_pct = round(errors*100/requests, 2)
| where error_pct > 2
```

Note the ordering — the guard precedes the ratio. Reversing it would compute meaningless ratios and then filter them, which produces the same result here but breaks as soon as the ratio is used for anything else in the pipeline.

### 6.5 Review cadence and change control

| When | Action |
|---|---|
| Before enabling | Fit every statistical threshold from §6.3 |
| One week after enabling each tier | **Re-tune anything that fired more than twice without a human deciding it mattered** |
| On Kong 3.5+ upgrade | Re-tune A5 — `latencies.kong` no longer includes receive time |
| On a change to `service.read_timeout` or `service.retries` | **Nothing.** Both are read from the data |
| On adding or renaming a service or route | Check the five names (§2.9); A13 will fire and confirm the change |
| Quarterly, or after any traffic-profile change | Re-run §6.3 baselines and compare |

**Edit in `local/`, never `default/`.** Splunk reads `default/` then overlays `local/`; anything changed in `default/` is lost on the next redeploy. Copy only the stanza being changed into `local/macros.conf` or `local/savedsearches.conf`. **Edits made through the Splunk UI land in `local/` automatically — it is only hand-editing the files that gets this wrong.**

**Never disable a noisy alert — tune its threshold.** A disabled alert is a blind spot nobody remembers.

---

## 7. Notification design

### 7.1 Channel

**Email only, for all 13 alerts.** No webhook, no PagerDuty, no Slack integration in this version.

Rationale and the honest limitation: email needs no external integration, no shared secret, and no dependency this pack cannot verify. It is also **not a paging channel** — Critical alerts 2 and 11 are delivered on a channel with no acknowledgement, no rota awareness, and no escalation-on-no-response. Bridging Critical severities into whatever actually pages the on-call rota is the first extension this design expects (§13 open items).

### 7.2 Message design

`alert_actions.conf` sets **only formatting**. It does **not** set recipients and does **not** configure the mail server.

```
[email]
inline = 1
format = table
sendresults = 1
include.view_link = 1
include.results_link = 1
include.search = 1
include.trigger = 1
include.trigger_time = 1
sendcsv = 0
sendpdf = 0
```

| Setting | Design intent |
|---|---|
| `inline = 1`, `format = table`, `sendresults = 1` | **The triggering rows are in the email body**, so the on-call engineer can triage without logging in. This is what makes A1's `caused_by_kong` / `caused_by_upstream` split useful at 3am |
| `include.view_link`, `include.results_link`, `include.search` | Pivot straight into Splunk; the search text is included so it can be modified rather than reconstructed |
| `include.trigger`, `include.trigger_time` | What fired and when — needed for correlating with deploys |
| `sendcsv = 0`, `sendpdf = 0` | **Attachments off.** These alerts return small aggregate tables, and CSV attachments frequently trip mail-gateway scanning — a silent delivery failure |

**The result row *is* the message.** Every alert's `stats` is designed so its columns answer the runbook's first question:

| Alert | Columns that decide the next action |
|---|---|
| A1 | `caused_by_kong` vs `caused_by_upstream` → **which team owns it** |
| A2 | `upstream_host`, `affected_clients`, `status_codes` → scope |
| A3 | `baseline_per_15m` → how busy the silent route normally is |
| A5 | `p95_kong_ms` **vs** `p95_upstream_ms` → is the fault inside Kong |
| A6 | `exhausted_requests` **before** `retry_pct`; `configured_retries` per service |
| A7 | `current_list` vs `baseline_list` → **which pod left** |
| A8 | `configured_timeout_ms` → the value actually used; `paths` → which operation |
| A9 | `enforced_by` → Kong's plugin or the upstream's own 429 |
| A10 | `distinct_clients` → one broken workload or an attack |
| A11 | `first_seen`, `paths_touched` → new caller, and what it reached for |

### 7.3 Subject convention

```
[Kong][<SEVERITY>] <condition> - <consequence or attribution>
```

Examples as shipped:

```
[Kong][CRITICAL] Upstream unreachable - Kong cannot reach the backend
[Kong][CRITICAL] Admin API - denied or erroring caller
[Kong][HIGH] Gateway self-latency breach - problem is inside Kong
[Kong][HIGH] Log ingestion stalled - Kong monitoring is blind
[Kong][HIGH] Route traffic stopped - $job.resultCount$ route(s) silent
[Kong][MEDIUM] Upstream target left the pool
[Kong][LOW] Rate limit rejection spike
```

Three properties: **`[Kong]` prefix** for mail-rule filtering; **severity in the subject** so triage order is visible on a phone lock screen without opening anything; **the consequence, not just the condition** — "problem is inside Kong" and "monitoring is blind" pre-empt the first minute of investigation.

`$job.resultCount$` is substituted where the count of affected entities is the most useful thing in the subject line.

Subject severity is manually kept consistent with `alert.severity`. All 13 currently agree.

### 7.4 Recipient routing

**Recipients are per-alert (`action.email.to` in `savedsearches.conf`), not global** — deliberately, so a Medium latency warning and a Critical Admin API security alert can go to different people.

All 13 ship as `CHANGEME@example.com`. **Every one must be replaced before enabling.**

The intended routing model, to be filled in alongside the contact register in §8.4:

| Severity | Alerts | Route to |
|---|---|---|
| Critical (5) | 2, 11 | On-call rota **+** Security for 11 |
| High (4) | 1, 3, 5, 10, 12 | On-call rota; 12 also to Splunk/observability |
| Medium (3) | 4, 6, 7, 8, 13 | Team distribution list / business hours |
| Low (2) | 9 | Team distribution list |

### 7.5 Failure modes of the notification path

| Failure | Symptom | Prevention |
|---|---|---|
| Mail server not configured | **Alert triggers, then silently fails to deliver.** Looks exactly like coverage | Verify Settings → Server settings → Email settings **before enabling anything** |
| `CHANGEME@example.com` left in place | Same as above | Explicit pre-enable checklist item (§10.1) |
| CSV attachment blocked by mail gateway | Delivery failure or quarantine | `sendcsv = 0`, `sendpdf = 0` |
| Suppression masks a second entity | Second breach never notified | Understood and accepted (§5.6); switch the alert to per-result mode if unacceptable |
| Alert fires into an unattended mailbox | Nobody responds | The contact register in §8.4 must be completed before go-live |

**The mail server is an instance-level setting. This app cannot and should not configure it.** An alert with no mail server is worse than no alert at all, because it looks like coverage.

---

## 8. Escalation design

### 8.1 Severity to response

Severity answers "how fast", and it is set from **blast radius plus reversibility**, not from how alarming the condition sounds.

| Sev | Alerts | Meaning | Expected response |
|---|---|---|---|
| **5 Critical** | 2, 11 | Users are failing now (2), or the control plane may be exposed (11) | Immediate, out of hours |
| **4 High** | 1, 3, 5, 10, 12 | Users affected or **monitoring is blind** | Prompt; out of hours if sustained |
| **3 Medium** | 4, 6, 7, 8, 13 | Degradation or a **leading indicator** — users not yet failing | Business hours, same day |
| **2 Low** | 9 | Expected-behaviour signal needing attribution | Business hours |

Two placements are worth defending:

- **A12 (Log Ingestion Stalled) is High, not Medium**, even though nothing is broken for users. Every other alert is blind while it is true. A monitoring system that under-rates its own blindness reports health it cannot see.
- **A8 (Approaching Read Timeout) is Medium despite nothing having failed.** It is the cheapest alert in the pack to act on, precisely because there is no incident yet — and the runbook's instruction is explicit: **do not "fix" it by raising `read_timeout`,** which converts a fast failure into a slow one and lets connections pile up.

### 8.2 Ownership routing on `error_source`

**`error_source` is the single field that decides who owns the problem.** The design's core routing rule:

```
error_source = upstream  →  the application team owning that service
error_source = kong      →  the gateway / platform team
error_source = unknown   →  observability team (malformed or truncated events)
```

**Never escalate without it.** An escalation that omits `error_source` hands the receiving team the job of working out whether it is even theirs.

The `kong` branch subdivides by the latency contrast:

| `kong_ms` | `proxy_ms` | Conclusion | Owner |
|---|---|---|---|
| Normal | High | Backend is slow | Application team |
| **High** | Normal | **Fault is inside the gateway** — plugin, DNS, rate-limiter datastore | Gateway/platform team |
| Normal | Null (`short_circuited`) | Kong rejected before the upstream, or no healthy target | Gateway/platform **and** backend owner |

### 8.3 Escalation matrix

The operative version lives in [triage-guide.md](triage-guide.md#escalation-matrix); reproduced here as the design record:

| Finding | Owner | Urgency |
|---|---|---|
| `error_source=upstream`, backend returning 5xx | Application team owning that service | Match user impact |
| `error_source=kong`, `503`, no targets reachable | Gateway/platform team **and** the backend's owner | High |
| Backend latency high, Kong latency normal | Application team | Medium unless breaching SLA |
| **Kong latency high, backend normal** | **Gateway/platform team** | **High** |
| Auth failures, one caller | That application's team | Medium |
| Auth failures, many callers | Vault/identity team | High |
| **Auth failures, many callers over time, varied targets** | **Security** | High |
| Kong-enforced 429s | Gateway owner (limit) or caller (behaviour) | Low–Medium |
| **Unrecognised Admin API caller** | **Security** | **Critical** |
| Kong down, no response | Platform/on-call | Critical |
| Kong up, logs stopped | Splunk/observability team | High — **monitoring is blind** |
| Route missing from Kong config | Gateway/platform team | Match user impact |

**The three-way split on auth failures is the security design.** The same alert (A10) routes to an application team, the identity team, or Security depending entirely on `distinct_clients` and the spread over time and targets. This is why A10 carries those fields (§3.2 M5).

### 8.4 Contact register — **incomplete, blocks go-live**

| Team | Owns | Contact / rota | Hours |
|---|---|---|---|
| Gateway / Kong platform | Kong config, pods, routes, plugins | `TO BE COMPLETED` | |
| `vault-svc` application | Vault service and its pods | `TO BE COMPLETED` | |
| `admin-api` | Kong Admin API surface | `TO BE COMPLETED` | |
| Vault / identity | Vault auth backends, policies, sealing | `TO BE COMPLETED` | |
| Security | Admin API misuse, credential stuffing | `TO BE COMPLETED` | |
| Splunk / observability | Log pipeline, indexing | `TO BE COMPLETED` | |
| Platform / OpenShift | Pods, ingress, network policy, DNS | `TO BE COMPLETED` | |

> **An escalation path discovered at 3am is an escalation path that does not work.** This table and the `action.email.to` values in §7.4 are the same decision expressed twice; both must be completed before any alert is enabled. It is tracked as a go-live blocker in §13.

### 8.5 Handoff contract

Every escalation carries the same structured payload, whether it goes up or across to an application team. It is the difference between a one-hour and a one-day resolution.

```
WHAT USERS SEE
  Symptom / Started / Still happening / Impact

WHAT THE GATEWAY SAYS
  Service / route
  error_source:       kong | upstream        <- who owns it
  Status codes
  short_circuited:    was the backend contacted at all?
  upstream_status:    what the backend actually returned
  Latency:            kong_ms p95 / proxy_ms p95
  Retries:            max try_count, retries_exhausted
  Targets in play

CHECKED ALREADY
  Splunk receiving Kong logs / backend pod readiness / recent deployments
```

Most of it is generated by one search — this is a **design output**, not a convenience: the escalation payload is derived from the same normalised fields as the alerts, so it cannot disagree with them.

```spl
`kong_base`
| where status>=400
| stats count as requests, values(status) as status_codes,
        values(error_source) as error_source, sum(short_circuited) as short_circuited,
        values(upstream_status) as backend_returned,
        p95(kong_ms) as kong_p95_ms, p95(proxy_ms) as backend_p95_ms,
        max(try_count) as max_attempts, sum(retries_exhausted) as gave_up,
        values(upstream_ip) as targets, dc(client_ip) as distinct_callers
        by service, route
| sort - requests
```

---

## 9. Runbook design

### 9.1 Three-tier architecture

Documentation is tiered by **what the reader already knows and how much time they have**, because a document that serves an incident cannot also serve onboarding.

| Tier | Document | Entry condition | Optimised for |
|---|---|---|---|
| **T1 Reactive — alert-driven** | [alert-runbook.md](alert-runbook.md) | A named alert fired | One section per alert, in alert order. No searching |
| **T1 Reactive — symptom-driven** | [triage-guide.md](triage-guide.md) | Something is broken, no alert (or the alert is not the whole story) | A 3-minute triage that decides most incidents, then 8 symptom branches (S1–S8) |
| **T2 Self-service** | [app-team-guide.md](app-team-guide.md) | An application team wants to know if it is them or the gateway | Answering "is it me?" without platform-team involvement |
| **T3 Preparatory** | [kong-primer-for-ops.md](kong-primer-for-ops.md) | Before the first incident | Kong and Splunk from zero. **Read once, not during an incident** |
| **T3 Reference** | [field-reference.md](field-reference.md), [log-format-validation.md](log-format-validation.md), [operations-guide.md](operations-guide.md) | Writing searches, validating the format, installing and tuning | Lookup, not narrative |

**Both T1 documents share one first move:** *open Service Health Overview and read the error attribution panel.* Every entry path converges on the same first decision, which is what makes the split into two T1 documents safe rather than confusing.

### 9.2 Alert to runbook to symptom mapping

| Alert | Runbook section | Related symptom branch |
|---|---|---|
| A1 Service 5xx Rate Breach | [§1](alert-runbook.md#1--service-5xx-rate-breach) | S1 Errors and 5xx |
| A2 Upstream Unreachable | [§2](alert-runbook.md#2--upstream-unreachable) | S1 |
| A3 Route Traffic Stopped | [§3](alert-runbook.md#3--route-traffic-stopped) | S6 Traffic stopped |
| A4 Upstream Latency p95 | [§4](alert-runbook.md#4--upstream-latency-p95-breach) | S2 It's slow |
| A5 Gateway Self-Latency | [§5](alert-runbook.md#5--gateway-self-latency-breach) | S2 |
| A6 Balancer Retry Storm | [§6](alert-runbook.md#6--balancer-retry-storm) | S1 |
| A7 Target Dropped | [§7](alert-runbook.md#7--upstream-target-dropped-from-rotation) | S1 |
| A8 Approaching Read Timeout | [§8](alert-runbook.md#8--approaching-upstream-read-timeout) | S2 |
| A9 Rate Limit Rejections | [§9](alert-runbook.md#9--rate-limit-rejections-spike) | S4 Rate limiting |
| A10 Auth Failure Spike | [§10](alert-runbook.md#10--authentication-failure-spike) | S3 Authentication failures |
| A11 Admin API Activity | [§11](alert-runbook.md#11--admin-api-unexpected-activity) | S8 Unexpected Admin API access |
| A12 Log Ingestion Stalled | [§12](alert-runbook.md#12--log-ingestion-stalled) | S7 No data in Splunk |
| A13 New Route Or Service | [§13](alert-runbook.md#13--new-route-or-service-detected) | S5 404s and routing |

Two symptom branches have **no corresponding alert** — S5 (404s and routing) and parts of S4 — which is the §5.9 deliberate-non-alert decision showing up as a documentation gap that is filled reactively rather than by paging.

### 9.3 Runbook section template

Every alert-runbook section follows the same five-part shape, so an engineer reads the same structure at 3am regardless of which alert fired:

| Part | Content | Why |
|---|---|---|
| **Means** | Plain-language restatement, including the default threshold | The alert name is never enough |
| **Read the email first** | Which result column decides the branch | Avoids a Splunk login for the most common case |
| **Then / Check** | Copy-pasteable SPL, ready to run | No query construction under pressure |
| **Usual causes** | Ranked by observed frequency | Most incidents are the first item |
| **Tuning / False positives** | When the alert is wrong, and what to change | The alternative is someone disabling it |

### 9.4 Anti-actions

The runbook set carries an explicit **"things not to do"** table. These are design-relevant because each one is a plausible action that makes the incident worse, and each maps to a property of the data:

| Do not | Because |
|---|---|
| Restart Kong to "clear" backend errors | If `error_source=upstream`, Kong is working. Restarting drops in-flight requests and fixes nothing |
| Raise `read_timeout` to stop 504s | Converts a fast failure into a slow one and lets connections pile up |
| Average `latencies.proxy` in your own searches | `-1` means "backend never contacted". **Averaging it makes a failing gateway look fast.** Use `kong_proxied` |
| Use `x-ratelimit-*` to check Kong's rate limiting | They come from Vault, not Kong |
| Disable a noisy alert | Tune its threshold. **A disabled alert is a blind spot nobody remembers** |
| Assume "no logs" means "outage" | Test the gateway directly first (§5.9) |
| Escalate without `error_source` | It is the single field that decides who owns the problem |

---

## 10. Operational lifecycle

### 10.1 Rollout

**Phase 0 — verify the foundation.** In order, because each step is meaningless if the previous one failed:

1. Data arriving and JSON resolving: `` `kong_index` | head 5 ``. Install `props.conf` **only if** dotted fields do not resolve (§2.5).
2. `` `kong_base` `` parses and produces sane fields:
   ```spl
   `kong_base` | head 5 | table _time service route status upstream_status error_source proxy_ms
   ```
3. `proxy_ms` is **never `-1`** and `upstream_status` shows **only 3-digit codes or null** — this verifies N1 and N2.
4. Dashboards render and dropdowns populate (§2.8). If dropdowns show only the static choices, the topology macro is the suspect, and dashboards still work.

**Phase 1 — mail path.** Configure the instance mail server. Replace **every** `CHANGEME@example.com`. Complete the contact register (§8.4). **This phase gates everything after it** — an alert with no mail server is worse than no alert.

**Phase 2 — enable structurally-safe alerts immediately.** Their thresholds cannot be mis-fitted (§6.1):

```
Kong - Log Ingestion Stalled
Kong - Upstream Unreachable
Kong - Admin API Unexpected Activity      ← after attributing every current caller (§6.3)
```

**Phase 3 — after one day of baseline.** Enough data for a rough sense of normal volume:

```
Kong - Service 5xx Rate Breach
Kong - Route Traffic Stopped
Kong - Authentication Failure Spike
```

**Phase 4 — after a full baseline week**, once percentiles are known: everything else. Fit each threshold from §6.3 before enabling it.

**Phase 5 — one week per tier.** **Re-tune anything that fired more than twice without a human deciding it mattered.**

### 10.2 Tuning loop

```
observe a week  →  fit from §6.3  →  enable  →  watch a week
        ▲                                            │
        └──── fired without mattering, ≥2 times ◄────┘
```

The exit condition is not "no alerts fired" — it is **"every alert that fired was worth someone's attention."**

### 10.3 Change control

| Change | Where | Notes |
|---|---|---|
| Index or source path | `local/macros.conf` → `[kong_index]` | One line; everything follows |
| A threshold | `local/savedsearches.conf` → that alert's stanza | Or `local/macros.conf` for `kong_min_requests` |
| Role names | `local/macros.conf` → the three role macros | Plus the two dashboard defaults (§2.9) |
| Dashboard colour bands | `local/data/ui/views/<view>.xml` | Re-fit alongside the aligned alert thresholds (§4.5) |
| A new alert | `local/savedsearches.conf`, named `Kong - <something>` | The `Kong -` prefix is required for the nav collection (§4.2) |

**Splunk reads `default/` then overlays `local/`. Anything edited in `default/` is lost on redeploy.** Copy only the stanza being changed. UI edits land in `local/` automatically.

### 10.4 Planned-change suppression

Three alerts fire on **deliberate** changes. The correct handling is suppression for the change window, **not disabling**:

| Alert | Fires on | Handling |
|---|---|---|
| A7 Target Dropped | Rolling deploy, scale-down | Suppress during the window — or accept it as deployment confirmation |
| A13 New Route Or Service | Any Kong config deployment | Suppress during the window; it is the audit trail the rest of the time |
| A3 Route Traffic Stopped | Decommissioning a route | Suppress, then remove the route from expectations |

A disabled alert is a blind spot nobody remembers to re-enable. A suppressed alert re-arms itself.

---

## 11. Non-functional design

### 11.1 Search cost

| Load | Shape | Control |
|---|---|---|
| Dashboard panels | On demand, bounded by `$tr$` | User-selected time range; `head` caps on all detail tables (10–30 rows) |
| Dropdown population | **One bounded 24h scan per dropdown per page load** | The pinned `earliest=-24h` in `kong_topology_source`. Shorten to `-4h` on a high-volume index |
| Scheduled alerts | 5–15 min windows, staggered | Max **3 concurrent dispatches** (§5.4) |
| A13 only | **7-day subsearch** | Runs 6-hourly, not hourly. This is the single most expensive scheduled search in the pack |
| A3, A7 | 24h baseline subsearch | Returns one row per entity; cheap |

**The two knobs that matter** if search load becomes a problem, in order: shorten `kong_topology_source`'s window, then lengthen A13's schedule. Do not raise subsearch caps (§5.1).

### 11.2 Permissions

```
[]         access = read : [ * ], write : [ admin, power ]   export = system
[macros]   ... export = system
[views]    ... export = system
[savedsearches] ... export = system
[props]    access = read : [ * ], write : [ admin ]   export = system
```

**Macros are exported to system scope deliberately.** `kong_base` is useful from the search bar in any app context; restricting it to this app would mean ad-hoc troubleshooting queries fail with "macro not found" unless the user remembered to switch app first — precisely the wrong failure at the wrong moment.

**There are no lookup or collection permissions because there is nothing to permission.** No KV Store collection, no CSV lookup, no persisted state (DD-3). That removes a whole class of failure: nothing to create, nothing needing app deployment on a gated tenancy, no read grant that decides whether a dropdown populates, and **no persisted artefact to review for what it might contain.**

### 11.3 Data sensitivity

| Field | Sensitivity | Design position |
|---|---|---|
| `client_ip` | Potentially identifying | Consumed for top-talker, concentration and security attribution — a required capability. May be a load-balancer address anyway (§2.10 defect 9) |
| `upstream_path` | May encode identifiers in a path segment | Displayed in path tables and included in A8/A9/A11 result rows |
| `vault_ns` | Tenancy identifier | Used for tenancy reporting |
| Response headers | Only three consumed, all non-secret | `x-vault-namespace`, `x-ratelimit-remaining`, `x-ratelimit-limit` |
| Request bodies, auth headers, tokens | **Not present in the serializer output and not consumed** | — |

Alert emails carry result rows inline (§7.2), which means **`client_ip` and `upstream_path` values leave Splunk over email.** Recipient lists should be scoped accordingly — this is a further reason recipients are per-alert rather than global (§7.4).

Index retention and any masking applied at ingest are properties of `index=kong_common` and are **not defined by this pack** (§13 open items).

---

## 12. Design decisions register

| ID | Decision | Alternatives rejected | Consequence |
|---|---|---|---|
| **DD-1** | One normalisation macro (`kong_base`) that every panel and alert builds on | Per-panel field handling; `FIELDALIAS` renaming | Four data traps solved once. **`kong_base` is a single point of failure — it is also the single point of verification**, and phase 0 of the rollout tests it explicitly |
| **DD-2** | A second population macro (`kong_proxied`) rather than a `where` clause per panel | Filtering inline everywhere | Impossible to compute an upstream statistic over requests that never reached the upstream *by accident* |
| **DD-3** | Topology discovered by a live bounded search; **no KV Store, no lookup, no stored state** | KV Store collection; CSV lookup | Installs on a gated Splunk Cloud tenancy with no filesystem access. Costs one bounded 24h scan per dropdown per page load, and silent routes are not listed |
| **DD-4** | Role names in three macros + two dashboard defaults | Hardcoding (was 14 places) | Redeployable against a different Kong install by editing five things. Simple XML cannot reference macros in `<default>`, hence five not three |
| **DD-5** | Configuration read **from the event**, never hardcoded | `kong_read_timeout_ms` / retry-budget macros (both removed) | Correct across services with different settings; nothing to tune or drift |
| **DD-6** | All alert logic in SPL; trigger always `results > 0` | Custom trigger conditions | An alert's condition is exactly reproducible by running its search |
| **DD-7** | Everything ships **disabled** | Ship enabled with default thresholds | Requires a deliberate rollout (O7). **Risk: a pack that is never enabled provides nothing** — mitigated by the tiered enable order making phase 2 same-day |
| **DD-8** | Time-based, search-global suppression | Per-result suppression by default | Cannot storm. Costs masking of a second entity within the window; per-alert opt-out documented |
| **DD-9** | Email only, per-alert recipients | Global recipient; webhook/paging integration | No external dependency. **Critical alerts are on a non-paging channel** — §13 open item |
| **DD-10** | `KV_MODE = json` search-time extraction, conditional | `INDEXED_EXTRACTIONS`; mandatory props | Works on already-indexed data with no re-index and no forwarder change |
| **DD-11** | Baselines computed in-search against a prior window | Persisted baseline; summary index | No stored state. Costs a subsearch per run and excludes the last hour to avoid self-depression |
| **DD-12** | Kong-enforced throttling detected **structurally** (`429` + `short_circuited`) | `x-ratelimit-*` headers | Correct. Those headers are Vault's; using them would attribute upstream quota exhaustion to the gateway |
| **DD-13** | Attribution (`error_source`) precomputed as a field, not derived per query | Per-query `case` expressions | One definition, used identically by dashboards, alerts and the escalation payload — they cannot disagree |

---

## 13. Assumptions, open items and errata

### 13.1 Assumptions

| # | Assumption | If wrong |
|---|---|---|
| A-1 | Kong Gateway **3.4**; log-serializer JSON to stdout | `latencies.kong` semantics change on 3.5+ (re-tune A5); new fields may appear |
| A-2 | Data indexed at `index=kong_common source="/openshift/logs.stdout"` | One-line change to `kong_index` |
| A-3 | Services `vault-svc` and `admin-api`; routes `auth-route`, `generic-route`, `ee-smp-functests` → `vault-svc` and `admin-route` → `admin-api` | Edit the five names (§2.9) |
| A-4 | The rate-limiting plugin is scoped to one route | A `kong-plugin` 429 on another route means the scope is wider than documented — A9 surfaces this |
| A-5 | `client_ip` is meaningful, i.e. `real_ip_header`/`trusted_ips` are configured in Kong | **All client attribution and A10/A11's `distinct_clients` cross-check become meaningless.** Verify before go-live |
| A-6 | Traffic is continuous enough that a 10-minute silence is abnormal | A12 fires on genuine quiet — widen its window or restrict it to business hours; **do not lower its severity** |
| A-7 | Email is an acceptable delivery channel for Critical severities | Bridge to a paging channel (open item O-1) |
| A-8 | `service.retries` and `service.read_timeout` are populated on every event | A6's exhaustion test and A8's proximity test become inert for that service (`isnotnull` guards prevent false firing, so they fail closed) |

### 13.2 Open items

| ID | Item | Why it matters |
|---|---|---|
| **O-1** | **Bridge Critical alerts (2, 11) to a real paging channel** | Email has no acknowledgement, rota awareness, or escalation-on-no-response (DD-9) |
| **O-2** | **Complete the contact register (§8.4) and all 13 `action.email.to` values** | **Go-live blocker.** Currently `TO BE COMPLETED` / `CHANGEME@example.com` |
| **O-3** | **External synthetic check on the gateway** | Log-based monitoring cannot detect its own subject's total absence (§5.9) |
| **O-4** | Verify `real_ip_header` / `trusted_ips` in Kong | Prerequisite for every client-attribution metric (A-5) |
| **O-5** | Define index retention and any ingest-time masking for `kong_common` | Not defined by this pack; affects how far back an investigation can go (§11.3) |
| **O-6** | Restore the `request` object in the serializer config | Unlocks method, URI, request ID and TLS version; removes the correlation gap (§2.2) |
| **O-7** | Consider an alert on a non-trivial `error_source=unknown` rate | Currently a dashboard-only signal; malformed events could accumulate unnoticed (§5.9) |
| **O-8** | Decide whether A11 moves to per-result suppression | Two unrecognised Admin API callers are two security events; the second is currently masked (§5.6) |
| **O-9** | Define an Admin API caller allowlist | A11 ships with the `NOT client_ip` extension point unused |
| **O-10** | Re-tune A5 and its dashboard bands on upgrade to Kong 3.5+ | `latencies.kong` stops including receive time (§2.10 defect 2) |
| **O-11** | **Identify the four unlabelled Kong plugins** (5 configured; only `rate-limiting` on `ee-smp-functests` is known) | A5 and the "Kong overhead by route — plugin cost" panel attribute overhead to a route, but **the design cannot name which plugin is expensive** while four are unidentified. This is the main gap in gateway-side latency diagnosis |

### 13.3 Errata in the shipped artefacts

Cosmetic and documentation inconsistencies found while writing this design. **None affects alerting behaviour**; all are worth fixing so the docs do not contradict the config.

| ID | Location | Issue | Correct value |
|---|---|---|---|
| **E1** | `savedsearches.conf` header; `operations-guide.md` §6.1, §6.3 heading | Says "twelve" alerts in several places; there are **13** (the §6.3 table itself lists 13, and §6.1's first line says "thirteen") | 13 |
| **E2** | `savedsearches.conf` header comment | Mid-sentence editing artefact: *"Two shared / One shared constant (`kong_min_requests`)…"* | One shared constant |
| **E3** | `savedsearches.conf` alert 8 `action.email.subject` | Hardcodes *"the 60s upstream read timeout"* while the logic reads `read_timeout` **per service** from the event | Remove the `60s`; the email body's `configured_timeout_ms` carries the real value |
| **E4** | `operations-guide.md` §6.3 footnote | *"Schedules are staggered … so twelve searches do not dispatch on the same minute"* — up to **three** do (§5.4) | Staggered to at most three concurrent dispatches |
| **E5** | `operations-guide.md` §7 threshold table, A13 row | *"compares traffic to the topology collection"* — stale wording; the collection was removed and the baseline is now in-search | Compares against the preceding 7 days, in-search |
| **E6** | `savedsearches.conf` alert 8 comment; `alert-runbook.md` §6 closing note | Refer to `read_timeout 60000` / `retries = 5` as configured facts | True of the current deployment, but both are now read per-service; phrase as "in this deployment" |

---

## 14. Acceptance criteria

The design is correctly implemented when all of the following hold.

**Logging layer**
- [ ] `` `kong_index` | head 5 `` returns Kong events, and dotted fields resolve
- [ ] `` `kong_base` `` parses and yields `service`, `route`, `status`, `error_source`, `proxy_ms`
- [ ] `proxy_ms` is **never `-1`** anywhere in the result set (N1 verified)
- [ ] `upstream_status` shows **only 3-digit codes or null** — never `-`, `""`, or a comma-joined list (N2 verified)
- [ ] Event timestamps match request time, not Kong config creation time (§2.4 verified)
- [ ] `read_timeout_ms` and `max_retries` are populated from the event, per service

**Dashboards**
- [ ] All five appear in the app navigation **in their expected positions**, not under "Other views" (§4.2)
- [ ] Service and Route dropdowns populate from live topology, and the static choices are present
- [ ] Every panel renders — an empty panel is a scope or role-name problem (§2.9), not a rendering one
- [ ] The Security dashboard defaults resolve to the correct Admin service and auth route
- [ ] Colour bands render on the KPI tiles and the flagged table fields

**Alerting**
- [ ] All 13 stanzas parse; each search runs standalone in the search bar
- [ ] Each search returns **zero rows under healthy conditions** and rows only when breached
- [ ] All 13 appear under the app's "Alerts" nav collection (`Kong -` prefix intact)
- [ ] Instance mail server configured and a test alert **delivered**
- [ ] **No `CHANGEME@example.com` remains** on any enabled alert
- [ ] Severities and subject-line severities agree
- [ ] Concurrent dispatch stays within the search-head budget (§5.4)

**Thresholds**
- [ ] Every **statistical** threshold has been fitted from a full week of baseline (§6.3)
- [ ] No **structural** threshold has been softened (§6.1)
- [ ] `kong_min_requests` set from the quietest service's observed volume
- [ ] All edits live in `local/`, not `default/`

**Escalation and runbooks**
- [ ] Contact register (§8.4) fully completed — **no `TO BE COMPLETED` rows**
- [ ] Every alert's recipient matches the §7.4 routing model
- [ ] Every alert has a runbook section, and the mapping in §9.2 resolves
- [ ] One on-call engineer has walked the 3-minute triage path end to end **before** go-live

**Coverage**
- [ ] Each row of §5.8 traces to an enabled alert, or is consciously accepted as a §5.9 gap
- [ ] `A-5` verified: `client_ip` is a real client address, not a load balancer
