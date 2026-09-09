# Kong Proxy — a primer for the operations team

**Read this once, before your first incident. It is not a runbook.**
When something is actually broken, go to [triage-guide.md](triage-guide.md).

You do not need prior Kong or Splunk experience. This explains both, in the order you need them.

---

## Contents

1. [What Kong actually is](#1-what-kong-actually-is)
2. [The four things Kong is made of](#2-the-four-things-kong-is-made-of)
3. [Our specific setup](#3-our-specific-setup)
4. [What happens to a request](#4-what-happens-to-a-request)
5. [Where it can break, and what you see](#5-where-it-can-break-and-what-you-see)
6. [The single most useful idea: Kong or upstream?](#6-the-single-most-useful-idea-kong-or-upstream)
7. [The three latency numbers](#7-the-three-latency-numbers)
8. [Using Splunk, from zero](#8-using-splunk-from-zero)
9. [Glossary](#9-glossary)

---

## 1. What Kong actually is

Kong is a **reverse proxy**. Applications do not talk to our backend services directly — they talk to Kong, and Kong forwards the request on and returns the answer.

```
    App  ──►  Kong  ──►  Backend service
         ◄──       ◄──
```

That middle position is why Kong matters operationally: **it sees every request**, so its logs are the single best place to find out what is happening. It is also why Kong gets blamed for everything — a failure anywhere along the chain looks, to the caller, like "the gateway is broken".

Most of your job during an incident is working out **which hop actually failed**. This pack is built to answer that.

Kong runs as pods on OpenShift. There are usually several, all identical, behind a load balancer.

---

## 2. The four things Kong is made of

Just four nouns. Everything in the dashboards and alerts is one of these.

| Term | What it is | Everyday analogy |
|---|---|---|
| **Service** | A backend that Kong forwards to. Has a hostname, a port, timeouts and a retry count. | The destination |
| **Route** | A rule that decides which requests go to which service — matched on URL path, HTTP method and host. | The address label |
| **Upstream / target** | The actual pod instances behind a service. One service usually has several targets, and Kong load-balances across them. | The individual delivery vans |
| **Plugin** | A piece of logic Kong runs on a request before forwarding it: authentication, rate limiting, logging. | The security checkpoint |

**The relationship that matters:** a request matches **one route**, which points at **one service**, which balances across **several targets**. Plugins run in between.

```
   request ──► route ──► service ──► one of N targets
                 │
                 └── plugins run here
```

A "route" is not a URL and a "service" is not a server. Getting these two words right is most of what separates a clear ticket from a confusing one.

---

## 3. Our specific setup

Two services, four routes. Learn this table — nearly every incident is about one row of it.

| Route | → Service | What it carries | Notes |
|---|---|---|---|
| `auth-route` | `vault-svc` | Vault login and token renewal | Kubernetes login, and token self-renewal |
| `generic-route` | `vault-svc` | General Vault API traffic | |
| `ee-smp-functests` | `vault-svc` | Functional test traffic | Has the **rate-limiting plugin** attached |
| `admin-route` | `admin-api` | **Kong's own control plane** | Privileged — see the warning below |

**Service settings that show up in alerts:**

| Setting | Value | Why you care |
|---|---|---|
| `read_timeout` | 60000 ms (60s) | A request waiting longer than this becomes a 504 |
| `retries` | 5 | Kong will try up to 6 different targets before giving up |
| Protocol/port | HTTPS, 443 | TLS failures to the backend show as connection errors |

> ### `admin-route` deserves special care
> It reaches Kong's **Admin API** — the interface that can read and rewrite routes, services and plugins. Anyone who can call it can reconfigure the gateway. Treat every caller there as privileged, and treat an unfamiliar source IP as a security incident until someone claims it.

---

## 4. What happens to a request

Follow this once and the alerts stop being mysterious.

```
   Client (an application)
     │
     │  ① Connect + TLS handshake
     ▼
 ┌──────────────────────────────────────────────────────┐
 │  KONG                                                │
 │                                                      │
 │  ② Match a route ─────── no match ──────► 404        │
 │                                                      │
 │  ③ Run plugins ───────── auth reject ───► 401 / 403  │
 │                 └─────── rate limit ────► 429        │
 │                                                      │
 │  ④ Pick a target from the service's pool             │
 │                 └─────── none healthy ──► 503        │
 └──────────────────────────────────────────────────────┘
     │
     │  ⑤ Forward and wait (retry up to 5 more targets)
     ▼
   Upstream service (Vault pods / Admin API)
     │
     │  ⑥ Response comes back
     ▼
   Client
```

**The crucial split:** at steps ② ③ ④ Kong answers *by itself*. The backend is never contacted. At steps ⑤ ⑥ the backend was contacted and its answer is passed through.

Kong records which of those happened, which is what lets you tell a gateway problem from a backend problem without guessing.

---

## 5. Where it can break, and what you see

| Broke at | Client sees | In the logs | Usually means |
|---|---|---|---|
| ① Connect / TLS | Timeout, connection refused | **Nothing** — Kong never logged it | Kong pods down, load balancer, network. If users report errors and Splunk shows *no* traffic, suspect this. |
| ② Route match | `404` | `route` is `(no-route-matched)` | Wrong URL, or a route was deleted from Kong config |
| ③ Auth plugin | `401` / `403` | `short_circuited=1` | Bad or expired credentials, or the auth backend is unhappy |
| ③ Rate limit | `429` | `short_circuited=1` | Caller exceeded its quota, or the limit is set too low |
| ④ No healthy target | `503` | `short_circuited=1`, no `tries` | All backend pods down or failing health checks |
| ⑤ Upstream slow | `504` after 60s | `proxy_ms` near 60000 | Backend overloaded or stuck |
| ⑤ Upstream refuses | `502` | Retries visible in `try_count` | Backend pod crashed or restarting |
| ⑥ Upstream error | `500` etc. | `upstream_status` matches | **The backend application itself is broken.** Not a gateway problem. |

**Read that last row carefully.** It is the most common false alarm: the gateway is working perfectly and faithfully delivering an error the backend produced.

---

## 6. The single most useful idea: Kong or upstream?

Every log line carries a field called **`error_source`**. It has one of these values:

| Value | Meaning | Who owns it |
|---|---|---|
| `ok` | Not an error | — |
| `kong` | **Kong produced this error itself.** The backend was never contacted, or Kong overrode its answer. | Gateway / platform team |
| `upstream` | **The backend returned this error** and Kong passed it through. | The application team owning that service |

This one field decides who you escalate to. Ask it first, always:

```spl
`kong_base` | where status>=400 | stats count by error_source, service, route
```

- Mostly `upstream` → the gateway is healthy. Go to the application team.
- Mostly `kong` → the problem is in the gateway or the path to the backend. Yours.

The **Service Health Overview** dashboard shows this as a chart, which is faster than running the search.

---

## 7. The three latency numbers

Every request records three timings, in milliseconds.

| Field | What it measures | High value means |
|---|---|---|
| `kong_ms` | Time **inside** Kong — route matching, plugins, reading the request | A plugin is slow, or Kong itself is under pressure |
| `proxy_ms` | Time **waiting on the backend** | The backend is slow. Not a gateway problem. |
| `total_ms` | End to end, everything | Use for user-facing SLAs |

Roughly: `total_ms ≈ kong_ms + proxy_ms + time sending the response back`.

> ### One thing you must know about `proxy_ms`
> When Kong answers by itself (steps ② ③ ④ above) it never contacted the backend, so there *is* no backend wait time. Kong writes **`-1`** in that field to mean "not applicable".
>
> `-1` is not a fast response. It is not a response at all.
>
> Every dashboard and alert already handles this — they blank it out and count it separately as `short_circuited`. **You only need to care if you write your own search.** If you ever see a negative or suspiciously tiny average latency, this is why: you have averaged in a pile of `-1`s. Use the `kong_proxied` macro instead of `kong_base` for anything measuring backend speed.

---

## 8. Using Splunk, from zero

### Running a search

1. Open Splunk. Click **Search & Reporting** in the app menu (top left).
2. Type or paste into the search bar.
3. Set the **time range** — the dropdown to the right of the search bar. Default is often "Last 24 hours"; during an incident use **Last 60 minutes** or **Last 4 hours**.
4. Press enter.

Every search in these guides is copy-paste ready. If one returns nothing, that is often itself the answer — the guides say what "no results" means each time.

### The backtick thing

Searches start with something like:

```spl
`kong_base`
```

Those are **backticks** (the key left of `1`), not quotes. `kong_base` is a saved shortcut — a *macro* — that expands into a much longer search which finds Kong's logs and tidies up the messy field names.

**Always start with it.** Without it, field names like `status` and `route` do not exist and every search returns nothing.

If you get `Error in 'SearchParser': ... macro 'kong_base' that cannot be found`, the macro is not installed or not shared with your app — that is a setup problem, not something you did wrong. See [operations-guide.md](operations-guide.md) section 8.

### Reading a search

Searches are pipelines. Each `|` passes results to the next step.

```spl
`kong_base`                                 ← get all Kong requests
| where status>=500                         ← keep only server errors
| stats count by route, error_source        ← count them, grouped
| sort - count                              ← biggest first
```

You only ever need to recognise a few:

| Piece | Does |
|---|---|
| `where X` | Keep only rows matching X |
| `stats count by X` | Count, grouped by X |
| `timechart ... ` | Same, but over time as a graph |
| `table a, b, c` | Show just these columns |
| `sort - x` | Biggest first (`-` means descending) |
| `head 20` | First 20 rows only |

### The dashboards

Five, each answering a different question. Open the first one during any incident.

| Dashboard | Open it when |
|---|---|
| **Service Health Overview** | **Always start here.** Is anything broken, and is it us or the backend? |
| **Route and Latency Analysis** | You know which route, and need detail — or things are slow |
| **Upstream and Balancer Health** | You suspect a specific backend pod |
| **Traffic, Rate Limiting and Clients** | Something is being throttled, or one caller is misbehaving |
| **Security and Tenancy** | Admin API access, or Vault auth questions |

Each has dropdowns at the top for Service and Route. Leave them on "All" to start, then narrow.

---

## 9. Glossary

| Term | Plain meaning |
|---|---|
| **Backend / upstream** | The real service Kong forwards to. Here: Vault, or Kong's Admin API. |
| **Balancer** | The bit of Kong that picks which backend pod gets the request. |
| **Client / caller** | Whatever made the request. Usually another application, not a person. |
| **Downstream** | Towards the client. The opposite of upstream. |
| **`error_source`** | Whether Kong or the backend produced an error. See [section 6](#6-the-single-most-useful-idea-kong-or-upstream). |
| **Gateway** | Kong itself. |
| **Macro** | A saved Splunk search shortcut, written in `backticks`. |
| **p95** | 95% of requests were faster than this. Better than an average, because averages hide the slow tail. p99 is the worst 1%. |
| **Plugin** | Logic Kong runs on a request — auth, rate limiting, logging. |
| **Retry** | Kong re-sending a failed request to a *different* backend pod. Ours retries up to 5 times. |
| **Retries exhausted** | All retries used and it still failed. **The user saw an error.** |
| **Route** | The rule matching a request to a service. |
| **Service** | A backend definition — hostname, port, timeouts. |
| **`short_circuited`** | Kong answered by itself; the backend was never contacted. |
| **Sentinel** | A fake value meaning "not applicable". Here, `proxy_ms = -1`. |
| **Target** | One backend pod inside a service's pool. |
| **Throttled** | Rejected with `429` for exceeding a rate limit. |
| **`upstream_status`** | The status code the backend returned — which can differ from what the client got. |

### Platform and infrastructure words

These appear in the triage steps. You do not need deep knowledge of any of them — this is enough to follow the instructions and know who to ask.

| Term | Plain meaning |
|---|---|
| **Backoff** | Waiting longer between each retry. A client that retries instantly on failure ("no backoff") makes an overloaded service worse. |
| **CI/CD** | The automated build-and-deploy pipeline. A common legitimate caller of the Admin API. |
| **Circuit breaker / passive health check** | Kong watching for failures and temporarily removing a pod from rotation after several in a row. It re-adds it later automatically. Explains a pod vanishing and returning without anyone touching it. |
| **Cordon** | Marking a pod or node so it stops receiving new work, without deleting it. Used to take a sick pod out of service while keeping it for investigation. This is a Kubernetes/OpenShift action — if you do not have that access, ask the platform team. |
| **DNS caching** | Kong remembers the IP addresses behind a hostname for a period rather than looking them up every request. If backend pods are replaced and get new IPs, Kong can keep trying the old ones until the cache expires. |
| **Ingress** | The layer in front of Kong that gets outside traffic into the cluster. If it breaks, requests never reach Kong and nothing is logged. |
| **Network policy** | Cluster firewall rules controlling which pods may talk to which. A change here can cut Kong off from a backend with no code change anywhere. |
| **On-call** | The person or rota carrying the pager for a service right now. |
| **Readiness probe** | A health check Kubernetes runs against a pod. A pod failing it should stop receiving traffic. A pod that is *running* but *not ready* is a common cause of errors. |
| **Sealed (Vault)** | HashiCorp Vault's locked state. A sealed Vault is running but refuses all requests until it is unsealed. Looks like a total outage of `vault-svc`. Escalate to the Vault team — it is not a gateway problem. |
| **Service mesh** | An optional networking layer between services (Istio, Linkerd). If present, it sits between Kong and the backend and can itself cause connection failures. |
| **Thread pool / connection pool** | A fixed number of workers an application uses to handle requests. When exhausted, new requests queue and latency climbs sharply while CPU looks fine. |

### Application design words

| Term | Plain meaning |
|---|---|
| **Idempotent** | Safe to run more than once with the same result. `GET` is naturally idempotent; "charge this card" is not. Matters because Kong retries — see [app-team-guide.md](app-team-guide.md) section 1. |
| **Idempotency key** | A unique value the caller sends so the backend can recognise and ignore a duplicate of a request it already processed. |

---

## HTTP status codes you will actually see

| Code | Meaning | First thought |
|---|---|---|
| `200` | Fine | — |
| `401` | Not authenticated | Bad or expired credentials |
| `403` | Authenticated, not allowed | Permissions |
| `404` | No route matched | Wrong URL, or a route was deleted |
| `429` | Too many requests | Rate limit hit |
| `500` | Backend application error | Almost always the app, not Kong |
| `502` | Backend gave an invalid response | Backend pod crashed or restarting |
| `503` | No healthy backend available | All pods down or failing health checks |
| `504` | Backend took longer than 60s | Backend hung or overloaded |

**`502`, `503` and `504` are gateway-shaped codes** — they are Kong reporting a problem reaching the backend. `500` is the backend's own error passed through. That distinction routes the ticket.

---

**Next:** [triage-guide.md](triage-guide.md) — what to do when something is actually wrong.
