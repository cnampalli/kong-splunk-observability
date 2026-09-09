# kong-proxy-splunk

Splunk dashboards and alerts for Kong Gateway **3.4** proxy logs fronting `vault-svc` and `admin-api`.

Built to answer the first question of any gateway incident: **is Kong broken, or is the upstream broken?** These logs can answer that definitively — Kong records both what it did and what the upstream said — and the pack is designed around making that split visible.

---

## What's here

```
kong-proxy-splunk/
├── docs/
│   │  ── for the operations team ──
│   ├── triage-guide.md              # SOMETHING IS BROKEN, START HERE. Symptom-driven.
│   ├── alert-runbook.md             # An alert fired: what it means, what to check
│   ├── kong-primer-for-ops.md       # Kong + Splunk from zero. Read once, before an incident.
│   ├── app-team-guide.md            # For teams whose app is onboarded via a route
│   │  ── for whoever installs it ──
│   ├── operations-guide.md          # Install and tune the Splunk pack
│   ├── log-format-validation.md     # Your log format vs the official Kong 3.4 contract
│   ├── field-reference.md           # Splunk field map + the SPL traps this data sets
│   │  ── for whoever reviews or extends it ──
│   └── logging-monitoring-design.md # The detailed design: metrics, thresholds,
│                                    # alerting model, escalation, design decisions
└── splunk-app/
    └── kong_proxy_monitoring/       # Drop into $SPLUNK_HOME/etc/apps/
        ├── default/
        │   ├── app.conf
        │   ├── props.conf           # Field extraction (only needed if JSON isn't resolving)
        │   ├── macros.conf          # kong_base — everything depends on this
        │   ├── savedsearches.conf   # 13 alerts, all ship DISABLED (see below)
        │   ├── alert_actions.conf   # Email formatting
        │   └── data/ui/
        │       ├── nav/default.xml
        │       └── views/           # 5 Simple XML dashboards
        └── metadata/default.meta
```

## Which document do I need?

| You are | Read |
|---|---|
| **On call, something is broken right now** | [triage-guide.md](docs/triage-guide.md) — symptom-driven, starts with a 3-minute triage that decides most incidents |
| **Responding to a specific alert** | [alert-runbook.md](docs/alert-runbook.md) — one section per alert |
| **New to Kong or Splunk** | [kong-primer-for-ops.md](docs/kong-primer-for-ops.md) — both explained from zero. Read before your first incident, not during it. |
| **An application team with a route on Kong** | [app-team-guide.md](docs/app-team-guide.md) — check your own service, work out if it is you or the gateway |
| **Installing this pack** | [operations-guide.md](docs/operations-guide.md) |
| **Reviewing, extending or signing off the design** | [logging-monitoring-design.md](docs/logging-monitoring-design.md) — the metric catalogue, alerting model, threshold derivation, escalation design and the decisions behind them |

---

## Quick start

```bash
cp -r splunk-app/kong_proxy_monitoring $SPLUNK_HOME/etc/apps/
$SPLUNK_HOME/bin/splunk restart
```

Then verify the foundation works:

```spl
`kong_base` | head 5 | table _time service route status upstream_status error_source proxy_ms
```

No filesystem access (Splunk Cloud)? The dashboards are Simple XML and paste-importable via **Edit → Source**. Full manual path in [operations-guide.md](docs/operations-guide.md).

## Dashboards

| Dashboard | Use it when |
|---|---|
| **Service Health Overview** | **Start here in an incident.** Golden signals + Kong-vs-upstream error attribution |
| **Route and Latency Analysis** | Drilling into a route, or investigating slowness. Works cross-service or single-route from the same page. |
| **Upstream and Balancer Health** | A backend pod is suspected |
| **Traffic, Rate Limiting and Clients** | Throttling, or a noisy client |
| **Security and Tenancy** | Admin API audit, per-namespace, auth operations |

**Dropdowns discover your topology rather than assuming it** — and do so with **no stored state at all**. The `kong_topology_source` macro runs a live search bounded to the last 24 hours:

```spl
search earliest=-24h latest=now `kong_base`
| where service!="(no-service)" AND route!="(no-route-matched)"
| stats count by service, route
```

No KV Store collection, no CSV lookup, no maintenance job. That is deliberate: a collection can only be created by deploying an app or by a REST call, neither reliably available on a Splunk Cloud tenancy, and a missing one breaks every dropdown. A CSV lookup avoids that but gets replicated to the indexer tier on every search and needs a writer to maintain it. A live search works the moment the macro is defined, on any Splunk, with nothing installed.

The trade is one bounded 24h scan per dropdown per page load, and a route silent for over 24h is not listed — it still appears in every table under "All routes". Shorten or lengthen the window in that one macro to move the trade either way.

## Alerts

Thirteen, covering health (5xx rate, upstream unreachable, route gone silent, latency, retries and retry exhaustion, target dropped, timeout proximity), security (rate limiting, auth failures, Admin API access), one that watches the pipeline itself, and one that reports when Kong's own config changes.

**They ship disabled on purpose.** The thresholds come from the Kong config and general gateway norms, not from your traffic. Enabling twelve untuned alerts is how a monitoring rollout gets muted in its first fortnight. Run the baseline queries in [operations-guide.md](docs/operations-guide.md) section 7, then enable in the documented order — the three threshold-light ones are safe immediately.

Replace `CHANGEME@example.com` before enabling anything.

---

## What the validation found

Full detail in [log-format-validation.md](docs/log-format-validation.md). The short version:

**Your logs match the Kong 3.4 contract**, verified against `kong/pdk/log.lua` at `release/3.4.x` rather than the docs site — which matters, because the current docs describe fields (`latencies.receive`, `source`, `workspace`) that do not exist in 3.4. Validating against them would have produced three false findings.

**One genuine gap:** the entire `request` object is absent, though 3.4 does emit it. That costs HTTP method, the pre-rewrite client path, request size, TLS version, and `request.id` — the key that correlates a proxy log with Kong's error log. Not a blocker; `upstream_uri` substitutes throughout. Worth chasing.

**Three traps this data sets**, all handled once in `kong_base` so no panel re-solves them:

- `latencies.proxy = -1` is a **sentinel** meaning "never reached the upstream", not a duration. Averaging it raw makes a failing gateway report better latency than a healthy one — silently.
- `upstream_status` arrives in four shapes (`200`, `"502, 200"`, `-`, null). A naive `tonumber()` drops exactly the retried requests you most want to see.
- `started_at` is epoch milliseconds while `route.created_at` is epoch seconds — in the same event.

**One finding that changed what got built:** the `x-ratelimit-*` headers in these logs are **HashiCorp Vault's**, not Kong's. Kong emits `RateLimit-Limit` and window-suffixed `X-RateLimit-Limit-Second`; it never emits a bare `x-ratelimit-limit`. The arithmetic confirms it — `remaining: 9998` exceeds `limit: 1000`. Building rate-limit monitoring on those headers would have silently monitored Vault instead of the gateway. Kong-enforced throttling is detected structurally instead: `status=429 AND short_circuited=1`.

---

## The one thing to understand

Everything is built on the `kong_base` macro. It normalises the traps above exactly once, which is what lets error attribution stay three lines instead of a nest of string comparisons repeated across twelve alerts:

```spl
| eval error_source = case(
    status < 400,                     "ok",
    short_circuited == 1,             "kong",      /* never reached upstream */
    isnull(upstream_status),          "kong",      /* covers null, "" and "-" */
    tonumber(upstream_status) >= 400, "upstream",
    true(),                           "kong")      /* upstream fine, Kong rewrote it */
```

Use `kong_proxied` — not `kong_base` — for anything measuring upstream latency. Short-circuited requests never reached the upstream, so including them measures something that did not happen.

---

## Environment

| | |
|---|---|
| Kong Gateway | 3.4 |
| Splunk index | `kong_common` |
| Source | `/openshift/logs.stdout` |
| Services | `vault-svc`, `admin-api` |
| Routes | `auth-route`, `generic-route`, `ee-smp-functests` → `vault-svc`; `admin-route` → `admin-api` |
| Plugins | 5 configured; `rate-limiting` on `ee-smp-functests` (other 4 unidentified) |
| Upstream read timeout | 60000 ms |
| Upstream retries | 5 |

Upgrading to Kong 3.5+ changes `latencies.kong` (receive time splits into its own field) and will shift latency thresholds. See [operations-guide.md](docs/operations-guide.md) section 9.
