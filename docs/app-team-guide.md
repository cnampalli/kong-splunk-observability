# Kong Proxy — guide for application teams

**For product owners and engineers whose application is onboarded to Kong via a route and service.**

Two things this answers: *is my application healthy through the gateway*, and *when something breaks, is it mine or the platform's*. Answering the second one yourself, before raising a ticket, is the single biggest thing you can do to shorten a resolution.

New to any of this? [kong-primer-for-ops.md](kong-primer-for-ops.md) explains the concepts. You do not need it to use this page.

### Before you start: getting access

Everything here runs in Splunk. You need:

1. **A Splunk account** with permission to search `index=kong_common`. If a search returns nothing but a colleague sees results, it is index permissions — ask the Splunk team, not the gateway team.
2. **The `kong_base` macro** shared with your app context. If you get `Error in 'SearchParser': ... macro 'kong_base' that cannot be found`, that is a setup issue — ask whoever installed the pack.

To run a search: open Splunk → **Search & Reporting** → paste into the search bar → set the time range dropdown → enter. [Primer section 8](kong-primer-for-ops.md#8-using-splunk-from-zero) has more if you need it.

---

## 1. What the gateway does to your traffic

Four behaviours that surprise application teams. Read these once — they change how you design your API.

### It retries your requests, up to 5 times

When Kong cannot get a usable answer from one of your pods, it retries the same request against a **different pod**. Our services are set to 5 retries, so 6 attempts in total.

**Retries happen on connection-level failures** — connection refused, connection reset, or a timeout — **not on HTTP error responses.** If your application cleanly returns a `500`, Kong treats that as a valid answer and passes it straight through without retrying.

> **A single client request can still reach your application more than once.**
>
> The dangerous case is when your pod *did* receive and process the request, but the connection dropped or timed out before the response got back. Kong sees no answer, assumes the pod is unhealthy, and sends the same request to another pod. Your application has now executed it twice.
>
> If your endpoint has side effects — charges a card, sends an email, creates a record — that is a duplicate. Kong has no way to know. Make such endpoints **idempotent**: accept an idempotency key from the caller, or make repeated identical calls safe to process. This matters most for slow endpoints, because timeouts are what trigger it.

Check whether it is happening to you:

```spl
`kong_base` | search route="YOUR-ROUTE"
| stats count as requests, sum(retried) as retried_requests,
        sum(retries_exhausted) as gave_up, max(try_count) as most_attempts
```

`retried_requests` above zero means some of your calls were delivered more than once.

### It gives up after 60 seconds

`read_timeout` is 60 seconds. A request your backend has not answered by then becomes a **504** to the caller, and your application may still be processing it — unaware nobody is listening.

If you have legitimately long operations, they should be asynchronous (return a job ID immediately, poll for the result). Asking for a higher timeout converts a fast failure into a slow one and lets connections pile up.

### It can reject requests before your application ever sees them

Authentication and rate-limiting plugins run **inside Kong**. A `401`, `403` or `429` may never have reached you. If someone reports errors you have no record of, that is why — check `short_circuited` (see [section 3](#3-is-it-me-or-the-gateway)).

### The caller IP you see may not be the real caller

`client_ip` in the gateway logs is whatever connected to Kong. Behind a load balancer that is the load balancer's address. If every request appears to come from one or two IPs, that is what is happening — do not use it for per-customer analysis without confirming with the platform team.

---

## 2. Check your own service, in one search

Replace `YOUR-ROUTE` with your route name. Time range: last 24 hours to start.

```spl
`kong_base` | search route="YOUR-ROUTE"
| stats count as requests,
        sum(eval(if(status<400,1,0)))  as succeeded,
        sum(eval(if(status>=400 AND status<500,1,0))) as client_errors_4xx,
        sum(eval(if(status>=500,1,0))) as server_errors_5xx,
        sum(eval(if(error_source=="upstream",1,0))) as our_fault,
        sum(eval(if(error_source=="kong",1,0)))     as gateway_fault,
        p50(proxy_ms) as our_latency_p50,
        p95(proxy_ms) as our_latency_p95,
        p95(kong_ms)  as gateway_overhead_p95,
        sum(retried)  as retried,
        dc(client_ip) as distinct_callers
| eval success_rate=round(succeeded*100/requests, 2)
```

How to read it:

| Column | What it tells you |
|---|---|
| `success_rate` | Your headline number |
| `our_fault` vs `gateway_fault` | **Who owns the errors.** See [section 3](#3-is-it-me-or-the-gateway). |
| `our_latency_p95` | How long *your application* took. This is yours to own. |
| `gateway_overhead_p95` | How long Kong added. Normally single-digit milliseconds. |
| `retried` | Requests delivered to you more than once |
| `distinct_callers` | How many clients you have |

Don't know your route name?

```spl
`kong_base` | stats count by service, route | sort - count
```

---

## 3. Is it me or the gateway?

Every request records **`error_source`**. This is the field that settles the argument.

```spl
`kong_base` | search route="YOUR-ROUTE"
| where status>=400
| stats count by error_source, status, short_circuited
| sort - count
```

| `error_source` | Meaning | Whose |
|---|---|---|
| `upstream` | **Your application returned this error.** Kong passed it through unchanged. | Yours |
| `kong` | **Kong produced this itself** — it either never reached you, or overrode your answer. | Platform's |

And `short_circuited` tells you whether your application was contacted at all:

| `short_circuited` | Meaning |
|---|---|
| `1` | Kong answered by itself. **Your application never saw this request.** |
| `0` | The request reached you; this is your response coming back. |

Together they cover every case:

| `error_source` | `short_circuited` | What happened | Raise with |
|---|---|---|---|
| `upstream` | `0` | Your application returned an error | Your own team |
| `kong` | `1` | Rejected at the gateway — auth, rate limit, or no healthy pod | Platform team |
| `kong` | `0` | Reached your pods but Kong could not get a usable answer — crash, timeout, connection reset | Both. Start with your pod health. |

---

## 4. Common situations

### "Users report errors we have no record of"

Almost always requests rejected inside Kong before reaching you.

```spl
`kong_base` | search route="YOUR-ROUTE"
| where short_circuited==1
| stats count by status, error_source, client_ip
| sort - count
```

`401`/`403` → credentials. `429` → rate limit. `503` → Kong could not find a healthy pod of yours.

### "It's slow and we think it's the gateway"

```spl
`kong_proxied` | search route="YOUR-ROUTE"
| timechart span=5m p95(kong_ms) as "Gateway overhead", p95(proxy_ms) as "Your application"
```

Two lines. If "Your application" is the tall one, the latency is yours. Gateway overhead is normally single-digit milliseconds — if it is comparable to your own latency, raise it with the platform team.

### "We're getting 504s"

Your application took longer than 60 seconds.

```spl
`kong_proxied` | search route="YOUR-ROUTE"
| where proxy_ms >= (read_timeout_ms * 0.8)
| table _time, upstream_path, proxy_ms, read_timeout_ms, client_ip
| sort - proxy_ms
```

Look at whether it is one endpoint, one caller, or one of your pods. Different fixes.

### "One of our pods is sick but users aren't complaining yet"

Retries are hiding it. Users still get `200`s and your error rate looks flat — until the retries run out.

```spl
`kong_base` | search route="YOUR-ROUTE"
| where retried==1
| eval failed_first=mvindex('tries{}.ip', 0)
| stats count as retry_events by failed_first
| sort - retry_events
```

A pod appearing repeatedly here is the unhealthy one. This is a genuinely early warning — act on it before it becomes an outage.

### "Our traffic dropped to zero"

Either your callers stopped, or your route was removed from Kong config. From the gateway both look identical.

```spl
`kong_base` | search route="YOUR-ROUTE" | timechart span=5m count
```

A cliff edge usually means config; a taper usually means clients. Check your own callers first, then ask the platform team whether a Kong config change went out.

---

## 5. Onboarding a new route — what to tell the platform team

| Question | Why it matters |
|---|---|
| Is your endpoint **idempotent**? | Kong retries up to 5 times. Non-idempotent endpoints can be executed more than once. |
| What is your **slowest legitimate operation**? | If it can exceed 60s, the default timeout will cut it off. |
| What is your **expected request rate**, peak and normal? | Sets rate limits, and the minimum volume thresholds on alerts |
| Do you need **authentication at the gateway**, or do you handle it? | Decides whether an auth plugin is attached |
| **Who is on call** for this service? | Goes in the escalation matrix. Without it, tickets bounce. |
| Are there **quiet periods** (overnight, weekends)? | Otherwise the "traffic stopped" alert pages someone every Saturday |

---

## 6. Raising a good ticket

Run this and paste the output. It contains almost everything the platform team will ask for:

```spl
`kong_base` | search route="YOUR-ROUTE"
| where status>=400
| stats count as requests, values(status) as status_codes,
        values(error_source) as error_source, sum(short_circuited) as short_circuited,
        values(upstream_status) as our_app_returned,
        p95(kong_ms) as gateway_p95_ms, p95(proxy_ms) as our_p95_ms,
        max(try_count) as max_attempts, sum(retries_exhausted) as gave_up,
        dc(client_ip) as distinct_callers
        by service, route
```

Plus:

- **When it started** (UTC), and whether it is still happening
- **What changed** on your side — deployments, config, scaling — and when
- **Which callers** are affected, if you know
- **Your pod status** — how many ready, any restarts

> Include `error_source` even if you think it points at you. Leading with it shows you have already triaged, and it is the first thing the platform team will look for.

---

## 7. Alerts that concern you

These fire to the platform team. Knowing they exist tells you what will already have been noticed.

| Alert | Fires when | Usually your side? |
|---|---|---|
| Service 5xx Rate Breach | 5xx rate exceeds threshold | Yes, if `caused_by_upstream` |
| Upstream Unreachable | Kong cannot reach any of your pods | Yes — pod health |
| Route Traffic Stopped | Your route goes silent | Maybe — clients, or config |
| Upstream Latency p95 Breach | Your application is slow | Yes |
| Balancer Retry Storm | Kong is retrying past an unhealthy pod | Yes — early warning |
| Approaching Read Timeout | Requests near the 60s cut-off | Yes |
| Rate Limit Rejections Spike | Callers being throttled | Depends who set the limit |
| Authentication Failure Spike | Auth failures on the auth route | Usually caller credentials |
| Gateway Self-Latency Breach | Kong itself is slow | No — platform |
| Log Ingestion Stalled | Monitoring blind | No — platform |

Full triage steps for each: [alert-runbook.md](alert-runbook.md).

---

## Related documents

| Document | For |
|---|---|
| [kong-primer-for-ops.md](kong-primer-for-ops.md) | Kong and Splunk concepts explained from zero |
| [triage-guide.md](triage-guide.md) | The platform team's symptom-driven triage |
| [alert-runbook.md](alert-runbook.md) | What a specific alert means |
| [field-reference.md](field-reference.md) | Every available field, for writing your own searches |
