# kong-proxy-splunk

Splunk dashboards and alerts for Kong Gateway **3.4** proxy logs fronting `vault-svc` and `admin-api`.

Built to answer the first question of any gateway incident: **is Kong broken, or is the upstream broken?** These logs can answer that definitively — Kong records both what it did and what the upstream said — and the pack is designed around making that split visible.

---

## What's here

```
kong-proxy-splunk/
├── docs/
│   ├── operations-guide.md          # START HERE to install. Configure Splunk from scratch.
│   ├── alert-runbook.md             # Per-alert triage: what fired, what to check
│   ├── log-format-validation.md     # Your log format vs the official Kong 3.4 contract
│   └── field-reference.md           # Splunk field map + the SPL traps this data sets
└── splunk-app/
    └── kong_proxy_monitoring/       # Drop into $SPLUNK_HOME/etc/apps/
        ├── default/
        │   ├── app.conf
        │   ├── props.conf           # Field extraction (only needed if JSON isn't resolving)
        │   ├── macros.conf          # kong_base — everything depends on this
        │   ├── collections.conf     # kong_topology KV Store collection (dropdown source)
        │   ├── transforms.conf      # Exposes the collection to SPL as a lookup
        │   ├── savedsearches.conf   # 13 alerts (ship DISABLED) + topology refresh (ships ENABLED)
        │   ├── alert_actions.conf   # Email formatting
        │   └── data/ui/
        │       ├── nav/default.xml
        │       └── views/           # 5 Simple XML dashboards
        └── metadata/default.meta
```

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

**Dropdowns discover your topology rather than assuming it.** They read the `kong_topology` KV Store collection, maintained hourly by the `Kong - Refresh Topology` search. That is a collection read on the search head, not an index scan — so discovery is free at page load, unlike a live population search that would scan the full selected time range every time a dashboard opened. And because the collection remembers entities for 30 days, a route that has gone silent stays selectable, which a live search would drop precisely when you needed it.

KV Store rather than a CSV lookup deliberately: a CSV lookup is bundled into the search bundle and replicated to the **indexer tier** on every search, while the collection stays on the search head and is access-controlled per collection. The refresh search is `| stats count by service, route`, and that aggregation is a security control — only service names, route names, a count and a timestamp can reach the collection, never client IPs, Vault namespaces or request paths.

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
