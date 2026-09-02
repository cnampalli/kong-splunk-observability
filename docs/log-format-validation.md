# Kong log format validation

**Question asked:** does the JSON arriving in Splunk match what Kong officially produces?

**Answer:** yes, with one genuine gap. The sample matches the Kong **3.4** log serializer contract on every field except the entire `request` object, which 3.4 does emit and this deployment is not sending. Three other fields that look missing are correctly absent — they do not exist in 3.4.

---

## How this was validated

Against the serializer source for the running version, not the documentation site:

- [`kong/pdk/log.lua` @ `release/3.4.x`](https://raw.githubusercontent.com/Kong/kong/release/3.4.x/kong/pdk/log.lua) — the authoritative field list
- [`kong/pdk/log.lua` @ `master`](https://raw.githubusercontent.com/Kong/kong/master/kong/pdk/log.lua) — to identify what changed after 3.4
- [Rate Limiting plugin reference](https://docs.konghq.com/hub/kong-inc/rate-limiting/) — for the header contract in finding 5

**This distinction mattered.** The current Kong documentation describes a serializer with `latencies.receive`, `source`, and `workspace` fields. None of those exist in 3.4. Validating the sample against the current docs would have produced three false findings and sent someone hunting for fields that were never supposed to be there.

The 3.4 serializer root, verbatim from source:

```lua
local root = {
  request = {
    id, uri, url, querystring, method, headers, size, tls
  },
  upstream_uri, upstream_status,
  response  = { status, headers, size },
  latencies = { kong, proxy, request },
  tries, authenticated_entity, route, service, consumer,
  client_ip, started_at
}
```

---

## Field-by-field result

| Field | Kong 3.4 | In sample | Verdict |
|---|---|---|---|
| `client_ip` | yes | yes | Match |
| `started_at` | yes | yes | Match |
| `upstream_uri` | yes | yes | Match |
| `upstream_status` | yes | yes | Match |
| `response.status` / `.headers` / `.size` | yes | yes | Match |
| `route` (full entity) | yes | yes | Match |
| `service` (full entity) | yes | yes | Match |
| `tries[]` | yes | yes | Match |
| `latencies.kong` / `.proxy` / `.request` | yes | yes | Match |
| `latencies.receive` | **not in 3.4** | absent | Correct — added in 3.5 |
| `source` | **not in 3.4** | absent | Correct — added after 3.4 |
| `workspace` / `workspace_name` | **not in 3.4** | absent | Correct — added after 3.4 |
| `consumer` | only when authenticated | absent | Expected — no consumer on this request |
| `authenticated_entity` | only when authenticated | absent | Expected |
| **`request.*`** | **yes** | **absent** | **Gap — see finding 1** |

---

## Finding 1 — the `request` object is missing, and it should not be

Kong 3.4 emits a full `request` object. This deployment emits none of it.

**What that costs:**

| Missing field | What you lose |
|---|---|
| `request.id` | The correlation key. `X-Kong-Request-Id` ties a proxy log to the matching Kong error log. Without it, joining the two is guesswork. |
| `request.method` | HTTP verb breakdown. `route.methods` only gives what the route *accepts*, not what was actually called. |
| `request.uri` / `request.querystring` | The true client path before rewriting. `upstream_uri` is the post-rewrite path, which is not always the same thing. |
| `request.size` | Ingress byte volume. Only egress (`response.size`) is currently measurable. |
| `request.tls.version` / `.cipher` | TLS posture reporting — which clients are still on old TLS. |

**Likely causes, in the order worth checking:**

1. **A serializer customisation.** A `kong.log.set_serialize_value()` call, or `custom_fields_by_lua` on the logging plugin, scrubbing `request` for privacy. This is plausible here: the request URI carries the Vault namespace (`/v1/<nsName>/auth/kubernetes/login`), which some teams classify as tenant-identifying.
2. **Splunk-side filtering.** A `props.conf` / `transforms.conf` `SEDCMD` or nullQueue rule dropping the object at ingest.
3. **The sample was hand-trimmed** before being shared. Given the visible sanitisation elsewhere — `xxxx` in every UUID, and a malformed `[a-zA-Z0-0-]` character class where `[a-zA-Z0-9-]` was clearly meant — this is a real possibility and the cheapest to rule out.

**Impact on this build:** low. `upstream_uri` substitutes for path throughout, and no dashboard depends on `request.*`. Panels that would benefit are written to light up automatically if it reappears.

**Recommendation:** check cause 3 first, then cause 1. If restoring `request.id` is possible, do it — request-ID correlation is the single highest-value field for incident work and nothing else replaces it.

---

## Finding 2 — `latencies.kong` on 3.4 includes receive time

```lua
kong = (ctx.KONG_PROXY_LATENCY or ctx.KONG_RESPONSE_LATENCY or 0) + (ctx.KONG_RECEIVE_TIME or 0)
```

On 3.4, Kong's reported overhead is plugin/routing time **plus** the time spent reading the request from the client. Kong 3.5 splits these into `latencies.kong` and `latencies.receive`.

**Consequence:** a client uploading a large body on a slow link inflates what looks like "Kong overhead" on 3.4. And every threshold tuned on 3.4 will read differently after an upgrade — the same traffic will appear to get faster, because receive time moves to its own field.

**Where this is handled:** Alert 5 in `savedsearches.conf` carries an inline warning to re-tune after upgrading. The Latency Analysis dashboard states it in a banner.

---

## Finding 3 — `latencies.proxy = -1` is a sentinel, not a measurement

```lua
proxy = ctx.KONG_WAITING_TIME or -1
```

`-1` means **the request never reached the upstream**. Kong answered it directly — an auth plugin rejected it, a rate limiter rejected it, no healthy target was available, or the request was blocked before proxying.

**Why this matters more than it looks:** `-1` is a valid number. `avg()`, `p95()` and `max()` will happily include it, dragging latency statistics downward. A gateway rejecting 30% of traffic at the edge will report *better* upstream latency than a healthy one. The failure is silent — no error, no null, just a quietly wrong number on a dashboard someone is using to decide whether to page.

**Where this is handled:** `kong_base` converts `-1` to null in `proxy_ms` and exposes the condition separately as `short_circuited`. The `kong_proxied` macro exists so latency statistics can never accidentally include these requests. Every latency panel and alert uses `kong_proxied`, not `kong_base`.

This same field is also what makes Kong-vs-upstream error attribution possible at all — see the summary below.

---

## Finding 4 — `upstream_status` has four shapes

Derived from nginx's `$upstream_status`, it can arrive as any of:

| Shape | Meaning |
|---|---|
| `200` | Normal — one upstream attempt |
| `"502, 200"` | Balancer retried; one entry per try, comma-joined |
| `-` | nginx reached no upstream at all |
| null / absent / `""` | Kong short-circuited before the balancer ran |

**Two ways to get this wrong.** A bare `tonumber()` returns null on the retry case, dropping exactly the requests most worth investigating. Taking the last element blindly breaks on `"502, -"`, where the final attempt failed to connect — you get `-`, not a status.

**Where this is handled:** `kong_base` splits on comma, keeps only well-formed 3-digit codes, and takes the last survivor:

```spl
| eval us_codes        = split(replace(coalesce('upstream_status', ""), " ", ""), ",")
| eval us_codes        = mvfilter(match(us_codes, "^[0-9]{3}$"))
| eval upstream_status = mvindex(us_codes, -1)
```

The output is guaranteed to be a valid 3-digit code or null. Nothing downstream has to think about dashes or lists.

---

## Finding 5 — the `x-ratelimit-*` headers are Vault's, not Kong's

This one changed what got built.

The sample carries `x-ratelimit-limit: 1000`, `x-ratelimit-remaining: 9998`, `x-ratelimit-reset: 3514`. These look like rate-limiting plugin output. They are not.

**Kong's Rate Limiting plugin emits:**

- `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` (the RFC-draft form)
- `X-RateLimit-Limit-Second`, `X-RateLimit-Limit-Minute`, `X-RateLimit-Limit-Hour`, `X-RateLimit-Limit-Day`, `X-RateLimit-Limit-Month`, `X-RateLimit-Limit-Year` and the matching `X-RateLimit-Remaining-*` (the legacy form)
- `Retry-After` on 429s

**Every legacy header carries a window suffix. Kong never emits a bare `x-ratelimit-limit`.**

Three things confirm these are HashiCorp Vault's own quota headers, passed through:

1. The naming matches Vault's rate-limit quota headers exactly.
2. They arrive alongside `x-vault-index` and `x-vault-namespace` — unambiguously upstream headers.
3. The arithmetic is impossible for a single limiter: `remaining` (9998) exceeds `limit` (1000).

**Also worth noting:** the sample is from `auth-route`, while the rate-limiting plugin is configured on `ee-smp-functests`. That is consistent — these headers have nothing to do with that plugin.

**Consequence.** A rate-limit dashboard built on `x-ratelimit-*` would monitor Vault's quota while appearing to monitor Kong's. When Kong's limiter started rejecting traffic, that dashboard would show nothing wrong.

**Where this is handled:** Kong-enforced throttling is detected structurally — `status=429 AND short_circuited=1`, meaning Kong rejected the request before it reached the upstream. The Vault headers are still charted, in a clearly separated section labelled as upstream quota, because they remain genuinely useful as an early warning on Vault's own limits.

---

## Finding 6 — mixed timestamp units within a single event

| Field | Unit | Digits |
|---|---|---|
| `started_at` | epoch **milliseconds** | 13 (`1787711730203`) |
| `route.created_at` | epoch **seconds** | 10 (`1787707424`) |
| `service.created_at` | epoch **seconds** | 10 |
| `tries[].balancer_start` | epoch milliseconds | 13 |
| `tries[].balancer_start_ns` | epoch nanoseconds | 19 |

Three different units in one event. `started_at` is the request time; the `created_at` fields describe when the Kong *configuration entity* was created and have nothing to do with when the request happened.

**The trap:** pointing Splunk's timestamp extraction at `created_at` backdates every event to the date the route was configured. All of them, to the same instant.

**Where this is handled:** `props.conf` sets `TIME_FORMAT = %s%3N` against `started_at` with an explicit warning.

---

## Finding 7 — `https_redirect_status_code: 475` is redaction noise

Kong's route schema accepts `426` (the default), `301`, `302`, `307`, `308`. `475` is not a valid value and the Admin API would reject it.

**Confirmed with the platform owner: this field is always `426` on these routes.** Combined with the other visible sanitisation in the sample, `475` is redaction noise.

Nothing is built on this field. Recorded here only so a future reader does not re-open it as a finding.

---

## Finding 8 — Splunk has no `alerts.conf`

Worth stating because it comes up when someone goes looking for the file.

| What you want to configure | Where it actually lives |
|---|---|
| An alert (search, schedule, trigger condition, throttle) | `savedsearches.conf` — an alert *is* a saved search carrying `alert.*` properties |
| How alerts are delivered (email format, webhook) | `alert_actions.conf` |
| Mail server, from-address, SMTP auth | Instance-level: Settings → Server settings → Email settings. Not app config. |

---

## What this makes possible

The most useful consequence of findings 3 and 4 together is that **Kong-generated errors can be separated from upstream-generated errors**:

```spl
| eval error_source = case(
    status < 400,                     "ok",
    short_circuited == 1,             "kong",      /* never reached upstream */
    isnull(upstream_status),          "kong",
    tonumber(upstream_status) >= 400, "upstream",
    true(),                           "kong")      /* upstream fine, Kong rewrote it */
```

That is the first question in any gateway incident — *is Kong broken, or is Vault broken?* — and these logs can answer it definitively, without guessing, because Kong records both what it did and what the upstream said.

The `isnull()` branch is only safe because `kong_base` has already collapsed null, `""` and `-` into a single null. That normalisation is why this logic is three lines instead of a nest of string comparisons repeated across twelve alerts.

---

## Open items

| Item | Priority | Action |
|---|---|---|
| `request.*` object absent | Medium | Check whether the sample was trimmed; if not, inspect `custom_fields_by_lua` and any `set_serialize_value()` call, then Splunk-side props. Restoring `request.id` unlocks error-log correlation. |
| Two plugins unaccounted for | Low | Five plugins are configured; only `rate-limiting` was named. If either is an auth or logging plugin it adds usable signals — an auth plugin would populate `consumer` and `authenticated_entity`, which would make per-consumer dashboards possible. |
| `client_ip` may be a load balancer | Medium | If Kong sits behind an ingress or LB, `client_ip` is the LB address, not the true origin. Check `real_ip_header` and `trusted_ips` in `kong.conf`. Until confirmed, treat the top-talkers panels as indicative only. |
| Thresholds unfitted | High | Every threshold ships as a documented placeholder. Run the baseline queries in [operations-guide.md](operations-guide.md) section 7 before enabling alerts. |
