# Kong Proxy — triage guide

**Something is wrong. Start here.**

New to Kong or Splunk? Read [kong-primer-for-ops.md](kong-primer-for-ops.md) first — but not right now, if this is live. You can work through this guide without it.

---

## How urgent is this?

Decide before you start digging. Impact is measured in **user-facing failure**, not in how alarming the alert looked.

| Situation | Priority | Response |
|---|---|---|
| Kong not responding at all — no traffic reaching it | **Critical** | Page on-call immediately, then investigate |
| Success rate below 50% on any route | **Critical** | Page on-call |
| Unrecognised caller on the Admin API | **Critical** | Escalate to security before investigating further |
| Success rate 50–95% | **High** | Work it now, notify affected app teams |
| Kong up but logs stopped | **High** | Users unaffected, but **every alert is now blind** |
| Requests approaching the 60s timeout | **Medium** | Nothing has failed yet — this is the warning shot |
| Latency elevated, no failures | **Medium** | Investigate in hours unless breaching an SLA |
| Retries elevated, users unaffected | **Medium** | A backend pod is sick and being masked. Act before it worsens. |
| Rate limiting on test traffic | **Low** | Usually working as designed |

---

## The first 3 minutes, every time

Do these three things before anything else. They take under three minutes and usually decide the whole incident.

### Step 1 — Is Splunk even seeing traffic?

```spl
`kong_base` | timechart span=1m count
```
Time range: **Last 60 minutes**

| What you see | What it means | Go to |
|---|---|---|
| A normal-looking line | Kong is receiving and logging traffic. Continue to step 2. | Step 2 |
| Line drops to **zero** and stays there | Either Kong is completely down, or logging broke. | [S7](#s7--no-data-in-splunk-at-all) |
| **No results at all** | Same as above, or your time range is wrong. Check the time picker first. | [S7](#s7--no-data-in-splunk-at-all) |

> If users are reporting errors but Splunk shows **no traffic at all**, that is a strong signal the problem is *in front of* Kong — the load balancer, DNS, or the Kong pods themselves. Kong cannot log a request it never received.

### Step 2 — Is it us or the backend?

```spl
`kong_base` | where status>=400 | stats count by error_source, service, route | sort - count
```
Time range: **Last 60 minutes**

| Result | Meaning | Do |
|---|---|---|
| No rows | No errors. If users are complaining, the problem may be latency, not failure. | [S2](#s2--its-slow) |
| Mostly `error_source=upstream` | **The backend returned these errors.** Kong delivered them faithfully. | Escalate to the application team owning that service. Use the [ticket template](#what-to-put-in-a-ticket). |
| Mostly `error_source=kong` | **Kong produced these errors itself.** The backend was never contacted, or Kong overrode it. | Continue to step 3 — this one is ours. |

This single split saves more time than anything else in this guide. Do not skip it.

### Step 3 — Open the Service Health Overview dashboard

It shows the same information as a picture, plus the shape over time. Note:

- **Success rate tile** — how bad is it, really?
- **Error attribution chart** — did it start suddenly, or creep?
- **Health matrix** — which service and route, specifically?

Then find your symptom below.

---

## Symptom index

Match what was actually reported to you.

| What you were told | Go to |
|---|---|
| "The API is down / returning errors" | [S1 — Errors and 5xx](#s1--errors-and-5xx) |
| "It's slow", "timeouts", "requests hanging" | [S2 — It's slow](#s2--its-slow) |
| "We can't log in", "getting 401 / 403" | [S3 — Authentication failures](#s3--authentication-failures) |
| "We're being rate limited", "getting 429" | [S4 — Rate limiting](#s4--rate-limiting) |
| "Our endpoint returns 404 / not found" | [S5 — 404s and routing](#s5--404s-and-routing) |
| "Our traffic has stopped" / a route went quiet | [S6 — Traffic stopped](#s6--traffic-stopped) |
| Dashboards empty, no data | [S7 — No data in Splunk](#s7--no-data-in-splunk-at-all) |
| An alert fired and I don't know what it means | [alert-runbook.md](alert-runbook.md) |
| Unexpected access to the Admin API | [S8 — Admin API access](#s8--unexpected-admin-api-access) |

---

## S1 — Errors and 5xx

**Reported as:** "The API is down", "we're getting 500s", "requests are failing."

### Severity

| Condition | Priority |
|---|---|
| Success rate under 50% on any route | **Critical** — page the on-call |
| Success rate 50–95% | **High** |
| Occasional errors, service usable | **Medium** — investigate in hours |

### Step 1: Split the errors

```spl
`kong_base`
| where status>=500
| stats count by service, route, status, error_source
| sort - count
```

### Step 2: Follow the branch

**If `error_source=upstream`** — the backend application returned the error.

Kong is healthy. Confirm and hand over:

```spl
`kong_base`
| where status>=500 AND error_source="upstream"
| stats count, values(upstream_status) as backend_returned, dc(client_ip) as affected_callers
        by service, route, upstream_path
| sort - count
```

`backend_returned` is what the backend actually said. Escalate to that application's team with the [ticket template](#what-to-put-in-a-ticket). **Do not restart Kong** — it is working.

**If `error_source=kong`** — Kong could not get a usable answer. Narrow further:

```spl
`kong_base`
| where status>=500 AND error_source="kong"
| stats count by status, short_circuited, service, route, upstream_host
| sort - count
```

| `status` | `short_circuited` | Meaning | Next |
|---|---|---|---|
| `503` | `1` | No healthy backend pod available at all | Step 3 |
| `502` | `0` | A backend pod accepted then failed the request | Step 3 |
| `504` | `0` | Backend took longer than 60s | [S2](#s2--its-slow) |
| `500` | `1` | Kong itself errored — rare, usually a plugin | Escalate to the Kong platform team |

### Step 3: Check the backend pods

```spl
`kong_base`
| where status>=500
| stats count as failures, values(status) as codes, dc(upstream_ip) as targets_tried,
        max(try_count) as attempts, sum(retries_exhausted) as gave_up
        by service, upstream_host
```

| Finding | Means | Action |
|---|---|---|
| `targets_tried` is 0 or blank | Kong never reached *any* pod | Check pods are running **and passing their readiness probe** — a pod can be running but not ready. Then check DNS resolves from inside a Kong pod, and that no network policy changed. |
| `gave_up` > 0 | Kong used every retry and still failed | The whole backend pool is unhealthy, not one pod. Users are definitely seeing errors. |
| `targets_tried` = 1 while normally more | Only one pod left serving | Scale up urgently — you have no redundancy left |

Then open **Upstream and Balancer Health** and look at the per-target table. A single pod with a high error rate should be taken out of service (cordoned) or restarted; the others will absorb the traffic.

> **A pod may disappear and come back on its own.** Kong watches for repeated failures and temporarily removes a failing target from rotation, re-adding it later. If a target vanishes from the per-target table and returns without anyone touching it, that is Kong protecting itself — but the underlying pod problem is still real and still worth fixing.

### Most common causes, in order

1. **A backend deployment went bad** — check for a recent release first, always
2. **Pods running but not ready** — failing their readiness probe yet still in rotation
3. **Backend resource exhaustion** — memory, or a full connection/thread pool. Latency climbs sharply while CPU looks fine.
4. **Vault specifically:** sealed vault, or its storage backend unavailable. A sealed Vault is running but refuses everything — looks like a total outage but is not a gateway problem.
5. **Stale DNS in Kong** — if backend pods were replaced and got new IPs, Kong may keep trying the old ones until its DNS cache expires. Suspect this when the pods look healthy but Kong cannot reach them, especially shortly after a redeploy.
6. **Network policy or service mesh change** — cuts Kong off from the backend with no code change anywhere

> **Check for a recent change before deep-diving.** Most 5xx incidents start with a deployment — of the backend, of Kong config, or of the platform. Establishing "what changed in the last hour" is faster than reading logs.

---

## S2 — It's slow

**Reported as:** "It's slow", "requests are timing out", "intermittent hangs."

### Step 1: Confirm it is real, and find where the time goes

```spl
`kong_proxied`
| stats count, p50(kong_ms) as kong_p50, p95(kong_ms) as kong_p95,
        p50(proxy_ms) as backend_p50, p95(proxy_ms) as backend_p95,
        p95(total_ms) as total_p95
        by service, route
| sort - backend_p95
```

> Note this uses `` `kong_proxied` ``, not `` `kong_base` ``. It excludes requests Kong rejected without contacting the backend — including those would understate backend latency. See the primer, [section 7](kong-primer-for-ops.md#7-the-three-latency-numbers).

### Step 2: Read the two columns against each other

| `kong_p95` | `backend_p95` | Verdict | Action |
|---|---|---|---|
| Normal (under ~50ms) | High | **The backend is slow.** Gateway is fine. | Escalate to the application team |
| High | Normal | **Kong is slow.** | Step 3 |
| Both high | Kong is likely waiting on something shared | Step 3, then escalate both |
| Both normal | Not a latency problem at the gateway | Check the client side, or the network between client and Kong |

### Step 3: If Kong itself is slow

Kong's own time covers plugin execution, route matching, and reading the request body.

```spl
`kong_proxied`
| stats count, p95(kong_ms) as kong_p95, p95(proxy_ms) as backend_p95 by route
| sort - kong_p95
```

| Finding | Likely cause |
|---|---|
| Only `ee-smp-functests` is slow | The rate-limiting plugin's datastore is slow — check Redis or the Kong DB depending on its policy |
| All routes equally slow | Kong pods under CPU/memory pressure, or the Kong datastore is slow |
| One route with large payloads | On Kong 3.4, `kong_ms` **includes time reading the request body**, so a big upload on a slow link inflates it without anything being wrong |

### Step 4: Are we about to start timing out?

The backend read timeout is 60 seconds. Requests approaching it are about to become `504`s.

```spl
`kong_proxied`
| where proxy_ms >= (read_timeout_ms * 0.8)
| table _time, service, route, upstream_path, proxy_ms, read_timeout_ms, client_ip, upstream_ip
| sort - proxy_ms
```

Any rows here are a warning, not yet an outage. Look at whether it is one path, one caller, or one backend pod — the three have different fixes.

> **Do not "fix" this by raising `read_timeout`.** That converts a fast failure into a slow one and lets connections pile up. Find why it is slow.

---

## S3 — Authentication failures

**Reported as:** "We can't log in", "getting 401", "token errors."

### Step 1: How many, and how many callers?

```spl
`kong_base`
| where status==401 OR status==403
| stats count as denials, dc(client_ip) as distinct_callers, values(status) as codes
        by route, service
| sort - denials
```

**`distinct_callers` decides everything:**

| Callers | Almost certainly | Priority |
|---|---|---|
| **1–2** | A misconfigured application — stale or wrong credentials | Medium. Contact that app team. |
| **Many, all at once** | The auth backend is broken, or credentials were rotated | High. This is an outage. |
| **Many, spread over time, varied targets** | Possible credential stuffing | **Security incident.** Escalate to security. |

### Step 2: Which operation is failing?

For Vault auth specifically:

```spl
`kong_base` | `kong_auth_route`
| where status==401 OR status==403
| eval operation=case(match(upstream_path,"/auth/kubernetes/login$"),"login",
                      match(upstream_path,"/auth/token/renew-self$"),"token renewal",
                      true(),"other")
| stats count by operation, client_ip
| sort - count
```

| Failing | Means |
|---|---|
| **login** | Service account tokens are wrong, expired, or the Kubernetes auth backend is misconfigured |
| **token renewal** | Existing tokens are expiring instead of being refreshed. **Expect a surge of logins to follow** as clients fall back to full re-authentication. |

### Step 3: Confirm it is not the backend

```spl
`kong_base`
| where status==401 OR status==403
| stats count by error_source, short_circuited
```

- `short_circuited=1` → a Kong **plugin** rejected it. Credential or plugin config problem.
- `short_circuited=0` → **Vault itself** rejected it. Vault-side policy or auth backend problem — escalate to the Vault team.

---

## S4 — Rate limiting

**Reported as:** "We're getting 429", "we're being throttled."

### Step 1: Who is throttling — Kong or the backend?

```spl
`kong_base`
| where status==429
| eval throttled_by=if(short_circuited==1, "Kong plugin", "Backend passed it through")
| stats count by throttled_by, route, client_ip
| sort - count
```

| `throttled_by` | Meaning | Owner |
|---|---|---|
| **Kong plugin** | Kong's rate-limiting plugin rejected it before the backend was contacted | Gateway team — is the limit right, or is the caller misbehaving? |
| **Backend passed it through** | The backend applied its own quota | The application team |

The rate-limiting plugin is configured on **`ee-smp-functests`** only. A "Kong plugin" 429 on any other route means the plugin's scope is wider than documented — worth confirming against the Kong config.

### Step 2: One caller, or everyone?

```spl
`kong_base` | where status==429
| stats count as throttled, dc(upstream_path) as paths by client_ip
| sort - throttled
```

- **One caller dominating** → usually a retry loop with no backoff. Contact them; a client retrying instantly on 429 makes it permanently worse.
- **Spread evenly** → the limit is genuinely too low for current traffic. A limit change is a config decision, not an ops one — raise it with the gateway owner.

> ### Do not use the `x-ratelimit-*` headers to investigate this
> Those headers in our logs come from **Vault**, not Kong. Kong emits `RateLimit-Limit` and window-suffixed `X-RateLimit-Limit-Second`; it never emits a bare `x-ratelimit-limit`. Reading them tells you about Vault's quota while you think you are looking at the gateway's. Detect Kong throttling structurally instead: `status==429 AND short_circuited==1`.

---

## S5 — 404s and routing

**Reported as:** "Our endpoint returns 404", "the URL stopped working."

### Step 1: Did a route match at all?

```spl
`kong_base`
| where status==404
| stats count by route, upstream_path, client_ip
| sort - count
```

| `route` value | Meaning |
|---|---|
| `(no-route-matched)` | **Kong had no rule for this request.** Either the URL is wrong, or a route was deleted or changed in Kong config. |
| A real route name | The route matched fine and the **backend** returned 404 — the path does not exist on the application. Not a gateway problem. |

That distinction is the whole investigation. `(no-route-matched)` is ours; a named route is theirs.

### Step 2: If nothing matched, did the route disappear?

```spl
`kong_base` | where route="(no-route-matched)"
| stats count, values(upstream_path) as attempted_paths, min(_time) as started by client_ip
| eval started=strftime(started,"%F %T")
```

Then check whether the route still exists in Kong config, and whether there was a recent config deployment. A config push that drops a route produces exactly this: sudden 404s with no errors anywhere else.

Cross-check with the **`Kong - Route Traffic Stopped`** alert — if a route vanished, its traffic will have gone to zero at the same moment.

---

## S6 — Traffic stopped

**Reported as:** "We're not seeing any requests", or the `Kong - Route Traffic Stopped` alert fired.

This one is deceptive: **there is no error to find.** Traffic simply ceased.

### Step 1: Confirm and find when

```spl
`kong_base` | timechart span=5m count by route
```
Time range: **Last 24 hours**

Look at *how* it stopped:

| Shape | Usually |
|---|---|
| **Cliff edge** — full traffic to zero instantly | A config change. A route was deleted, or something in front of Kong stopped routing. |
| **Taper** — declining over minutes | The client side is failing or scaling down |

### Step 2: Rule out each layer, in order

1. **Does the route still exist in Kong?** Check the config, and check for a recent deployment.
2. **Is the client actually running?** Traffic stopping looks identical from the gateway whether Kong lost the route or the caller went away. Ask the application team.
3. **Did anything change in front of Kong?** DNS, ingress, load balancer.
4. **Is it just quiet?** Some routes have genuine idle periods — overnight, weekends. Compare against the same time yesterday before escalating.

### Step 3: Is it everything, or one route?

```spl
`kong_base` | timechart span=5m count
```

All traffic to zero is a different, larger incident — go to [S7](#s7--no-data-in-splunk-at-all).

---

## S7 — No data in Splunk at all

**Two very different problems look identical here.** Separate them first.

### Step 1: Is Kong actually serving traffic?

**Test the gateway directly** — call a known endpoint through Kong from somewhere that normally reaches it. Any endpoint will do; you are testing whether Kong answers at all, not whether the answer is correct.

```bash
# -i shows the status line, -k skips cert checks, --max-time avoids hanging
curl -i -k --max-time 10 https://<kong-hostname>/<any-known-path>
```

| Result | Meaning |
|---|---|
| **Any HTTP status back** — even `401`, `404` or `503` | **Kong is alive and answering.** The logging pipeline is what broke. Monitoring is blind but users are unaffected. Go to step 3. |
| **Connection refused / timeout / no response** | **Kong is not reachable.** This is a user-facing outage. Go to step 2. |

An error status is good news here — it proves Kong is running and processing requests.

Do not skip this test. Assuming "no logs = outage" causes false escalations; assuming "no logs = logging problem" misses real outages.

### Step 2: If Kong is down

Escalate immediately as a user-facing outage, then check in this order:

1. Are the Kong pods running and ready?
2. Is the load balancer in front of Kong healthy?
3. Did anything deploy recently — Kong config, or the platform?
4. Is the Kong datastore reachable? Kong may refuse to start without it.

### Step 3: If Kong is up but logs stopped

Users are unaffected, but **every other alert in this pack is now blind**. Treat as High.

```spl
index=kong_common | timechart span=5m count
index=kong_common | stats max(_time) as last_event | eval last_event=strftime(last_event,"%F %T")
```

`last_event` tells you exactly when it stopped, which usually points straight at the cause.

Check in order:

1. The log collector on OpenShift (Fluentd/Vector — whatever ships `/openshift/logs.stdout`) — running and shipping?
2. Splunk indexer health, disk, licence limits
3. Did the source path or index name change?
4. Is it a permissions problem on *your* account? Ask a colleague to run the same search.

---

## S8 — Unexpected Admin API access

**`admin-route` reaches Kong's control plane.** Anyone calling it can read or rewrite routes, services and plugins.

**Treat any unrecognised caller as a security incident until someone claims it.**

### Step 1: Who is calling?

```spl
`kong_base` | `kong_admin_service`
| stats count as requests, values(status) as codes,
        sum(eval(if(status==401 OR status==403,1,0))) as denied,
        values(upstream_path) as paths_touched,
        min(_time) as first_seen
        by client_ip
| eval first_seen=strftime(first_seen,"%F %T")
| sort - requests
```

### Step 2: Assess

| Finding | Action |
|---|---|
| All IPs are known (CI/CD, operators) | Normal. Note it and move on. |
| **An unrecognised IP** | Escalate to security now. Do not wait to investigate further. |
| `denied` > 0 | Someone tried to reach the control plane and was rejected. That is the finding, regardless of intent. |
| `paths_touched` shows writes (`POST`, `PUT`, config paths) | Higher severity than reads. Correlate with a change ticket. |

### Step 3: Correlate with config changes

If the Admin API was written to, something in Kong's config may have changed. Check whether `Kong - New Route Or Service Detected` fired around the same time.

---

## Escalation matrix

| Finding | Owner | Urgency |
|---|---|---|
| `error_source=upstream`, backend returning 5xx | Application team owning that service | Match user impact |
| `error_source=kong`, `503`, no targets reachable | Gateway/platform team **and** the backend's owner | High |
| Backend latency high, Kong latency normal | Application team | Medium unless breaching SLA |
| Kong latency high, backend normal | Gateway/platform team | High |
| Auth failures, one caller | That application's team | Medium |
| Auth failures, many callers | Vault/identity team | High |
| Auth failures, many callers over time, varied targets | **Security** | High |
| Kong-enforced 429s | Gateway owner (limit) or caller (behaviour) | Low–Medium |
| Unrecognised Admin API caller | **Security** | Critical |
| Kong down, no response | Platform/on-call | Critical |
| Kong up, logs stopped | Splunk/observability team | High — monitoring is blind |
| Route missing from Kong config | Gateway/platform team | Match user impact |

### Who to actually contact

**Fill this in before you need it.** An escalation path discovered at 3am is an escalation path that does not work.

| Team | Owns | Contact / rota | Hours |
|---|---|---|---|
| Gateway / Kong platform | Kong config, Kong pods, routes, plugins | `TO BE COMPLETED` | |
| `vault-svc` application | Vault service and its pods | `TO BE COMPLETED` | |
| `admin-api` | Kong Admin API surface | `TO BE COMPLETED` | |
| Vault / identity | Vault auth backends, policies, sealing | `TO BE COMPLETED` | |
| Security | Admin API misuse, credential stuffing | `TO BE COMPLETED` | |
| Splunk / observability | Log pipeline, indexing | `TO BE COMPLETED` | |
| Platform / OpenShift | Pods, ingress, network policy, DNS | `TO BE COMPLETED` | |

---

## What to put in a ticket

Whether escalating up or handing to an application team, include all of this. It is the difference between a one-hour and a one-day resolution.

```
WHAT USERS SEE
  Symptom:            e.g. "500 errors on POST /v1/auth/kubernetes/login"
  Started:            2026-09-03 14:05 UTC
  Still happening:    yes / no
  Impact:             which applications, roughly how many requests

WHAT THE GATEWAY SAYS
  Service / route:    vault-svc / auth-route
  error_source:       kong  |  upstream        <- who owns it
  Status codes:       503 (1,240),  200 (88)
  short_circuited:    yes / no                 <- was the backend contacted?
  upstream_status:    what the backend actually returned, if anything
  Latency p95:        kong_ms 4ms / proxy_ms 58,000ms
  Retries:            try_count max 6, retries_exhausted 340
  Targets in play:    10.205.234.124, 10.205.234.98

CHECKED ALREADY
  - Splunk is receiving Kong logs: yes
  - Backend pods running: 3/3 ready
  - Recent deployments: Kong config pushed 13:58 UTC
```

Generate most of that with one search:

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

## Things not to do

| Do not | Because |
|---|---|
| Restart Kong to "clear" backend errors | If `error_source=upstream`, Kong is working. Restarting drops in-flight requests and fixes nothing. |
| Raise `read_timeout` to stop 504s | Converts a fast failure into a slow one and lets connections pile up. Find the slowness. |
| Average `latencies.proxy` in your own searches | `-1` means "backend never contacted". Averaging it makes a failing gateway look fast. Use `` `kong_proxied` ``. |
| Use `x-ratelimit-*` headers to check Kong's rate limiting | They come from Vault, not Kong. See [S4](#s4--rate-limiting). |
| Disable a noisy alert | Tune its threshold instead — see [operations-guide.md](operations-guide.md) section 7. A disabled alert is a blind spot nobody remembers. |
| Assume "no logs" means "outage" | Test the gateway directly first. See [S7](#s7--no-data-in-splunk-at-all). |
| Escalate without `error_source` | It is the single field that decides who owns the problem. |

---

## Related documents

| Document | For |
|---|---|
| [kong-primer-for-ops.md](kong-primer-for-ops.md) | Learning Kong and Splunk concepts. Read once, before an incident. |
| [alert-runbook.md](alert-runbook.md) | An alert fired and you need its specific meaning and triage steps |
| [app-team-guide.md](app-team-guide.md) | For application teams checking their own route |
| [field-reference.md](field-reference.md) | Writing your own searches |
| [operations-guide.md](operations-guide.md) | Installing and tuning the Splunk pack |
| [logging-monitoring-design.md](logging-monitoring-design.md) | Why a threshold is that number, why an alert exists, and what this pack deliberately does **not** detect |
