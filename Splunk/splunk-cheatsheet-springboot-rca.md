# Splunk Cheatsheet for Senior Java Backend Engineers
### Spring Boot Microservices — RCA & Daily Ops

---

## 1. Mental Model First — How to Approach Any Prod Issue

1. **Get the blast radius**: What's the symptom? Error rate, latency, specific customer, specific endpoint?
2. **Anchor on identifiers**: `traceId` > `spanId` > `requestId` > `userId`/`orderId`. Always search by the most specific ID you have.
3. **Narrow index/sourcetype/time FIRST**, then filter. Splunk is fastest when it doesn't have to scan everything.
4. **Follow the call graph**: gateway → service A → service B → DB/queue. Use traceId to jump between service indexes.
5. **Correlate with metrics**: logs tell you *what*, but check APM/metrics (AppDynamics/Dynatrace/Grafana) for *when it started* and *blast radius*, then come back to Splunk for *why*.
6. **Confirm the fix window**: once you have a hypothesis, bound the search to the exact incident window and validate before/after.

---

## 2. Search Performance Fundamentals (do this before anything else)

```spl
index=orders_prod sourcetype=springboot_json ("ERROR" OR "Exception")
```

- **Always specify `index=` and `sourcetype=`** — these are indexed fields, cheapest filters.
- Put **most selective / rare terms first** in the search bar (Splunk still optimizes, but habit matters for readability & TERM matching).
- Avoid leading wildcards (`*Exception`) — forces full scan. Use `TERM()` or indexed tokens instead.
- Use `fields` command early to drop unneeded fields before heavy transforms:
  ```spl
  index=orders_prod | fields traceId, spanId, level, message, http_status
  ```
- Prefer `tstats` on accelerated data models / indexed fields for huge time ranges:
  ```spl
  | tstats count where index=orders_prod by sourcetype, host
  ```
- Use `fast` mode (Job Settings) when you don't need field discovery — huge speed win on large windows.
- Time range: use the **picker**, not `earliest=`/`latest=` in SPL, unless scripting/scheduled search. When scripting:
  ```spl
  earliest=-4h latest=now
  earliest=-1d@d latest=@d      " yesterday, day-aligned
  ```

---

## 3. Core SPL Commands You'll Use Every Day

| Command | Purpose | Example |
|---|---|---|
| `search` | filter events | `index=payments status=500` |
| `fields` | keep/drop fields (perf) | `\| fields + traceId, message` |
| `table` | tabular display | `\| table _time, traceId, level, message` |
| `stats` | aggregation | `\| stats count by http_status` |
| `eventstats` | stats without collapsing rows | `\| eventstats avg(duration) as avg_dur by endpoint` |
| `streamstats` | running/rolling stats | `\| streamstats current=f count as seq by traceId` |
| `rex` | regex field extraction | see §5 |
| `eval` | derived fields/logic | `\| eval slow=if(duration>1000,"yes","no")` |
| `transaction` | group events into a session/txn | see §7 |
| `join` | correlate across sources (expensive, avoid if possible) | see §7 |
| `lookup` | enrich with static CSV/KV store | `\| lookup service_owners.csv service OUTPUT team` |
| `timechart` | time-bucketed aggregation | `\| timechart span=1m count by http_status` |
| `chart` | pivot-style aggregation | `\| chart count over endpoint by http_status` |
| `top` / `rare` | frequency analysis | `\| top limit=20 endpoint` |
| `dedup` | remove duplicate events | `\| dedup traceId` |
| `sort` | order results | `\| sort -_time` |
| `where` | post-stats filtering | `\| where count > 100` |
| `regex` | filter using regex (not extraction) | `\| regex message="timeout after \d+ms"` |
| `foreach` | apply eval/loop across fields | `\| foreach status_* [eval total=total+<<FIELD>>]` |
| `append` / `appendcols` | combine result sets | for side-by-side comparisons |
| `map` | subsearch per result row (careful, expensive) | rarely needed, prefer `join`/`transaction` |
| `spath` | parse JSON explicitly | `\| spath input=raw_message path=error.code` |
| `mvexpand` | explode multivalue field | when one log line has array of items |

---

## 4. Spring Boot / Microservices Specific Fields

Typical JSON log line (Logback + logstash-encoder / ECS style):

```json
{
  "@timestamp": "2026-07-26T10:15:32.123Z",
  "level": "ERROR",
  "logger_name": "com.company.order.service.OrderService",
  "message": "Failed to process order",
  "traceId": "6f2a1c9b4e8d4f3a9b0c1d2e3f4a5b6c",
  "spanId": "a1b2c3d4e5f60718",
  "service": "order-service",
  "pod": "order-service-7d9f6b8c5-xk2j9",
  "thread_name": "http-nio-8080-exec-4",
  "stack_trace": "...",
  "http_method": "POST",
  "http_path": "/api/v1/orders",
  "http_status": 500,
  "duration_ms": 1523
}
```

### Standard fields to know cold
- `traceId` — global correlation ID across all services (Sleuth/Micrometer Tracing/OpenTelemetry). **Your #1 tool.**
- `spanId` — single hop/operation within a trace (one service call, one DB query span, etc.)
- `parentSpanId` — links span to its parent (build call tree manually if needed)
- `service` / `application` — which microservice emitted this
- `pod` / `host` / `instance_id` — which instance (critical for "only failing on one pod" issues)
- `logger_name` — the Java class, use to jump straight to a component
- `thread_name` — useful for thread pool exhaustion / deadlock investigation
- `level` — TRACE/DEBUG/INFO/WARN/ERROR
- `duration_ms` / `latency_ms` — for perf RCA
- `http_status`, `http_method`, `http_path` — HTTP layer
- `exception` / `stack_trace` / `error.class` / `error.message` — exception details (often present when using ECS logging or MDC-populated fields)
- `MDC.*` fields — anything devs pushed into `MDC.put(...)`, e.g. `MDC.userId`, `MDC.orderId`, `MDC.correlationId`

> Tip: run `index=orders_prod | fieldsummary | table field, distinct_count, values` on a small time window whenever hitting a new service — instantly tells you what fields exist and their cardinality.

---

## 5. Extracting Data with `rex` (when fields aren't auto-extracted)

```spl
" extract traceId from a plain-text (non-JSON) log line
index=orders_prod
| rex field=_raw "traceId=(?<traceId>[a-f0-9]{32})"

" extract duration from message like 'completed in 245ms'
| rex field=message "completed in (?<duration_ms>\d+)ms"

" extract exception class from stack trace
| rex field=_raw "Caused by: (?<root_cause>[\w\.]+Exception)"

" extract key=value pairs generically (logfmt style)
| rex field=_raw max_match=0 "(?<kv_key>\w+)=(?<kv_value>[^\s,]+)"

" pull HTTP status + path out of an access-log style line
| rex field=_raw "\"(?<http_method>\w+) (?<http_path>\S+) HTTP.*\" (?<http_status>\d{3})"
```

For JSON logs that aren't auto-kv'd (e.g. nested inside a string field):
```spl
| spath input=message path=error.details output=error_details
```

---

## 6. THE Core Trace Correlation Workflow

**Scenario**: Customer reports order #12345 failed. You have no traceId yet.

```spl
" Step 1: find the traceId from any service touching this order
index=order_prod OR index=payment_prod OR index=inventory_prod
"12345"
| table _time, index, service, traceId, message
| sort _time
```

```spl
" Step 2: once you have traceId, pull the FULL cross-service timeline
index=*_prod traceId="6f2a1c9b4e8d4f3a9b0c1d2e3f4a5b6c"
| table _time, service, spanId, parentSpanId, logger_name, level, http_status, duration_ms, message
| sort _time
```

```spl
" Step 3: reconstruct span durations / gaps to find where time was lost
index=*_prod traceId="6f2a1c9b4e8d4f3a9b0c1d2e3f4a5b6c"
| eval time_epoch=_time
| sort time_epoch
| streamstats current=f last(time_epoch) as prev_time
| eval gap_ms=(time_epoch-prev_time)*1000
| table _time, service, spanId, message, gap_ms
```

```spl
" Step 4: isolate just the ERROR/WARN in that trace
index=*_prod traceId="6f2a1c9b4e8d4f3a9b0c1d2e3f4a5b6c" (level=ERROR OR level=WARN)
| table _time, service, logger_name, message, stack_trace
```

> If tracing isn't propagated correctly across a hop (common gap: async threads, Kafka consumers, scheduled jobs not inheriting MDC context), you'll see the traceId disappear at that boundary — **that itself is a finding** to raise with the dev team (missing `TaskDecorator`/MDC propagation in `@Async` or executor).

---

## 7. `transaction` vs `stats` vs `join` for Correlation

**`transaction`** — good for grouping start/end events of the same traceId into one duration:
```spl
index=order_prod traceId=*
| transaction traceId startswith="Received request" endswith="Response sent"
| table traceId, duration, eventcount
| sort -duration
```
⚠️ `transaction` is memory-heavy on large data — prefer `stats` where possible:
```spl
index=order_prod traceId=*
| stats earliest(_time) as start, latest(_time) as end, count as span_count by traceId
| eval duration_sec=end-start
| sort -duration_sec
```

**`join`** — only for small subsearches (default limit 50,000 rows, capped), e.g. enriching errors with a lookup index:
```spl
index=order_prod level=ERROR
| join type=left traceId
    [ search index=user_prod | table traceId, userId, userTier ]
| table _time, traceId, userId, userTier, message
```
Prefer `stats` + common field, or a `lookup`, over `join` whenever you can — `join` doesn't scale.

---

## 7A. Distributed Trace Path Reconstruction — "Which services did this traceId hit, and in what order?"

This is the query an expert reaches for **first**, before even reading messages — it tells you the shape of the request's journey across your microservice mesh.

```spl
" 1. Ordered list of every service/hop a traceId passed through
index=*_prod traceId="<PASTE_TRACE_ID>"
| sort _time
| stats earliest(_time) as hop_start, latest(_time) as hop_end, count as log_lines by service, spanId, parentSpanId
| eval hop_duration_ms=(hop_end-hop_start)*1000
| sort hop_start
| table hop_start, service, spanId, parentSpanId, hop_duration_ms, log_lines
```

```spl
" 2. Collapse into a single "path string" — e.g. gateway -> order-service -> payment-service -> inventory-service
index=*_prod traceId="<PASTE_TRACE_ID>"
| sort _time
| stats values(service) as services_hit, dc(service) as service_count, earliest(_time) as trace_start, latest(_time) as trace_end by traceId
| eval total_duration_ms=(trace_end-trace_start)*1000
| eval path=mvjoin(services_hit, " -> ")
| table traceId, path, service_count, total_duration_ms
```

```spl
" 3. Waterfall-style timeline (indent by parent/child depth) — good for spotting where time is actually spent
index=*_prod traceId="<PASTE_TRACE_ID>"
| sort _time
| streamstats current=f last(_time) as prev_time
| eval gap_ms=round((_time-prev_time)*1000,1)
| eval depth=if(isnull(parentSpanId) OR parentSpanId="", 0, 1)
| eval indent=if(depth=0, "", "    ")
| eval label=indent + service + " [" + spanId + "]"
| table _time, label, gap_ms, duration_ms, message
```

```spl
" 4. Detect a BROKEN trace — expected service didn't receive/propagate the traceId (common with @Async, Kafka consumers, scheduled jobs)
index=*_prod traceId="<PASTE_TRACE_ID>"
| stats dc(service) as services_seen, values(service) as service_list by traceId
| eval expected_services="gateway,order-service,payment-service,inventory-service,notification-service"
| eval expected_list=split(expected_services,",")
| eval missing=mvfilter(NOT match(expected_list, "^(".mvjoin(service_list,"|").")$"))
| table traceId, service_list, missing
```
> Tune the `expected_services` list per business flow (checkout, refund, onboarding, etc.) — this turns "silent trace context loss" into an explicit, alertable signal instead of something you notice by accident.

```spl
" 5. Span tree reconstruction using parent/child relationships (proper distributed-tracing tree, not just chronological order)
index=*_prod traceId="<PASTE_TRACE_ID>"
| stats earliest(_time) as start, latest(_time) as end, values(service) as service, values(message) as sample_msg by spanId, parentSpanId
| eval duration_ms=(end-start)*1000
| sort start
| table spanId, parentSpanId, service, duration_ms, sample_msg
" then visualize as a tree in an external tool, or eyeball parent->child chains manually — Splunk has no native tree viz,
" but the Trace Analytics / APM app (if licensed: Splunk APM / SignalFx / Observability Cloud) renders this natively.
```

```spl
" 6. Find the single slowest hop across all services for a trace — the actual bottleneck
index=*_prod traceId="<PASTE_TRACE_ID>"
| stats earliest(_time) as start, latest(_time) as end by service, spanId
| eval duration_ms=(end-start)*1000
| sort -duration_ms
| head 1
| table service, spanId, duration_ms
```

```spl
" 7. Aggregate view: for a GIVEN endpoint, what's the typical service fan-out / hop count across many traces (baseline for anomaly detection)
index=*_prod http_path="/api/v1/checkout" earliest=-1h
| stats dc(service) as hop_count by traceId
| stats avg(hop_count) as avg_hops, max(hop_count) as max_hops, min(hop_count) as min_hops, count as trace_samples
```

```spl
" 8. Cross-environment / cross-region trace continuity check (if services span clusters or regions)
index=*_prod traceId="<PASTE_TRACE_ID>"
| stats values(region) as regions_hit, values(cluster) as clusters_hit, values(service) as services by traceId
```

**Notes for expert use:**
- If your org has **Splunk APM (SignalFx / Observability Cloud)** or an **OpenTelemetry Collector** exporting spans into Splunk, prefer the native **Trace View / Service Map** UI for the tree/waterfall visualization — the SPL above is the fallback for teams only shipping logs (not full OTel spans) to Splunk.
- `parentSpanId` reliability depends on the tracing library: Spring Cloud Sleuth (legacy) and Micrometer Tracing (current, Boot 3.x) both populate it, but **naming/casing can differ** (`X-B3-ParentSpanId`, `parentId`, etc.) — always confirm actual field names with `| fieldsummary` per service before trusting query #5.
- For **Kafka-based flows**, the traceId must be propagated via message headers manually (Spring Kafka doesn't do this for free) — if a trace "restarts" at the consumer with no parentSpanId link, that's almost always a missing header-propagation implementation, not a Splunk issue.

---

## 8. Error Rate / Latency RCA Queries

```spl
" Error rate over time by service, spot the exact spike
index=*_prod sourcetype=springboot_json
| timechart span=1m count(eval(level="ERROR")) as errors, count as total
| eval error_rate=round(errors/total*100,2)
```

```spl
" Top exceptions in the last hour, ranked
index=order_prod level=ERROR earliest=-1h
| rex field=_raw "(?<exception_class>[\w\.]+Exception)"
| top limit=20 exception_class
```

```spl
" P50/P90/P99 latency by endpoint
index=order_prod http_path=*
| stats p50(duration_ms) as p50, p90(duration_ms) as p90, p99(duration_ms) as p99, count by http_path
| sort -p99
```

```spl
" Latency degradation compare: this hour vs same hour yesterday
index=order_prod earliest=-1h@h latest=now
| stats avg(duration_ms) as avg_today
| appendcols
    [ search index=order_prod earliest=-1d-1h@h latest=-1d@h
      | stats avg(duration_ms) as avg_yesterday ]
| eval pct_change=round((avg_today-avg_yesterday)/avg_yesterday*100,2)
```

```spl
" Find which pod/instance is causing errors (bad deploy / one bad node)
index=order_prod level=ERROR
| stats count by pod
| sort -count
```

```spl
" 5xx errors correlated with a specific downstream dependency call
index=order_prod http_status>=500
| rex field=message "calling (?<downstream_service>\w+-service)"
| stats count by downstream_service, http_status
```

```spl
" Detect thread pool exhaustion / connection pool starvation
index=order_prod ("RejectedExecutionException" OR "Timeout waiting for connection" OR "HikariPool" "Connection is not available")
| stats count by thread_name, message
```

```spl
" GC pauses / OOM correlation with latency spike (if GC logs indexed)
index=order_prod ("OutOfMemoryError" OR "GC overhead limit exceeded" OR "Full GC")
| table _time, pod, message
```

---

## 9. Comparing Deploys / Canary Analysis

```spl
" Error rate before vs after a deploy timestamp (replace with real deploy time)
index=order_prod
| eval period=if(_time < strptime("2026-07-26 09:00:00","%Y-%m-%d %H:%M:%S"), "before", "after")
| stats count(eval(level="ERROR")) as errors, count as total by period
| eval error_rate=round(errors/total*100,2)
```

```spl
" Compare two pod versions during canary rollout
index=order_prod pod=*
| eval version=if(match(pod,"canary"),"canary","stable")
| stats count(eval(level="ERROR")) as errors, count as total, avg(duration_ms) as avg_latency by version
| eval error_rate=round(errors/total*100,2)
```

---

## 10. Useful `eval` Functions Cheat List

| Function | Use |
|---|---|
| `if(cond, a, b)` | conditional field |
| `case(cond1, val1, cond2, val2, true(), default)` | multi-branch |
| `match(field, "regex")` | boolean regex match |
| `like(field, "pattern%")` | SQL-style wildcard match |
| `coalesce(f1, f2, f3)` | first non-null value |
| `mvcount(field)` / `mvindex(field, n)` | multivalue handling |
| `strftime(_time, "%Y-%m-%d %H:%M:%S")` | format epoch |
| `strptime(str, "%Y-%m-%d %H:%M:%S")` | parse to epoch |
| `round(x, n)` | rounding |
| `len(field)` | string length |
| `substr(field, start, len)` | substring |
| `split(field, ",")` + `mvexpand` | split delimited string into multivalue |
| `tonumber(field)` | cast string→number for math |
| `null()` | explicit null assignment |

---

## 11. Lookups & Enrichment

```spl
" Enrich service name with owning team from a CSV lookup
index=*_prod level=ERROR
| lookup service_owner_lookup.csv service OUTPUT team, oncall_slack_channel
| stats count by service, team, oncall_slack_channel
```

```spl
" KV store lookup for dynamic data (feature flags, config versions)
| lookup feature_flags_kv flag_name OUTPUT enabled
```

Create/manage lookups: **Settings → Lookups → Lookup table files** (upload CSV) then **Lookup definitions** to name it.

---

## 12. Alerting-Style Queries (for saved searches / alerts)

```spl
" Alert: error rate > 5% in last 5 min, per service
index=*_prod sourcetype=springboot_json earliest=-5m
| stats count(eval(level="ERROR")) as errors, count as total by service
| eval error_rate=round(errors/total*100,2)
| where error_rate > 5
```

```spl
" Alert: p99 latency breach
index=order_prod earliest=-5m
| stats p99(duration_ms) as p99 by http_path
| where p99 > 2000
```

```spl
" Alert: specific exception spike vs baseline (rolling)
index=order_prod "OutOfMemoryError" earliest=-15m
| stats count
| where count > 5
```

---

## 13. Regex Quick Reference (Java-log flavored)

| Goal | Pattern |
|---|---|
| UUID / traceId (32 hex) | `[a-f0-9]{32}` |
| UUID with dashes | `[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}` |
| Java exception class | `[\w\.]+Exception` |
| Duration in ms | `(\d+)\s?ms` |
| IPv4 | `\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}` |
| HTTP status | `\b[1-5]\d{2}\b` |
| logfmt key=value | `(\w+)=("[^"]*"|\S+)` |
| Kubernetes pod name | `[\w-]+-[a-f0-9]{8,10}-[a-z0-9]{5}` |
| Stack trace "Caused by" chain | `Caused by:\s*([\w\.]+):?\s*(.*)` |

Test regex live using the **Field Extractor UI** (right-click field → Extract Fields) before hardcoding `rex` — faster feedback loop.

---

## 14. Building a Reusable Dashboard Panel Set (for your service)

Minimum panels every service dashboard should have:
1. **Request rate** (`timechart count by http_path`)
2. **Error rate %** (as in §8)
3. **Latency percentiles** (p50/p90/p99 timechart)
4. **Top 10 error messages/exceptions**
5. **Traffic by pod/instance** (to catch uneven load / bad node)
6. **Dependency call health** (downstream service error/latency breakdown)
7. **Saturation indicators** (thread pool, connection pool, queue depth if logged)

```spl
" Reusable base search (set as a "base" search, others post-process it)
index=order_prod sourcetype=springboot_json
```

Use **Splunk macros** for repeated snippets:
```
" Settings > Advanced Search > Search Macros
" Macro name: get_errors(1)
index=$service$ level=ERROR
```
Call with: `` `get_errors("order_prod")` ``

---

## 15. Common Pitfalls & Gotchas

- **`transaction` silently drops incomplete transactions** past `maxspan`/`maxpause` — don't trust duration numbers blindly, verify with `stats` counts.
- **Multivalue fields from JSON arrays** — `stats count` will double count unless you `mvexpand` intentionally or dedupe.
- **Timezone mismatch**: Splunk displays in the searcher's browser TZ by default; verify `_time` vs `@timestamp` in raw JSON aren't offset — use `| eval _time=strptime(...)` if ingestion time ≠ event time.
- **`_indextime` vs `_time`**: `_time` is when the event occurred (from the log), `_indextime` is when Splunk indexed it — a big gap = ingestion lag, worth checking during "missing logs" complaints.
- **Sourcetype sprawl**: microservices teams often create inconsistent sourcetypes (`app-json`, `springboot`, `service-x-log`) — always run a quick:
  ```spl
  index=order_prod | stats count by sourcetype
  ```
  before assuming your search covers everything.
- **Subsearch default limits**: subsearches (used implicitly by `join`) cap at 50,000 results / 60s by default — silently truncated results look like "missing data."
- **High-cardinality `by` clauses** (e.g. `by traceId`) in `stats`/`chart` over huge windows can blow up memory — bound the time range first.
- **Async/thread-pool boundaries breaking MDC propagation** — if traceId vanishes mid-trace, it's usually a missing `TaskDecorator` or a Kafka listener not extracting trace headers, not a logging bug.
- **Missing `spanId`/`parentSpanId` correlation** — you often need to reconstruct order using `_time` + `logger_name` sequence rather than trusting parent/child fields if the tracing library version is inconsistent across services.

---

## 16. Fast Daily Habits (muscle memory)

- Start every RCA with `index=... earliest=-Xh | stats count by sourcetype` to sanity-check you're looking at the right data.
- Keep a **personal macro library**: `get_trace(traceId)`, `get_errors(service)`, `get_latency_pctiles(service)`.
- Save frequent searches as **Saved Searches**, not just history — history rolls off.
- Use `| eval _raw=message` sparingly for cleaner display without losing original `_raw` (or keep both: `| eval display_msg=message`).
- Pin the **fields sidebar** interesting fields (`traceId`, `service`, `level`, `http_status`) so they always show as columns.
- When stuck, `| head 1 | table *` or `| fieldsummary` to rediscover the actual field names — logging fields drift across service versions.

---

### Quick Reference: The One Query You'll Run Most

```spl
index=*_prod traceId="<PASTE_TRACE_ID>"
| table _time, service, spanId, logger_name, level, http_status, duration_ms, message
| sort _time
```

Keep this one saved as an alert-independent **"Trace Lookup"** dashboard with a `traceId` input token — it will save you more time than anything else in this doc.

---

## 17. Verified Guidance From Recognized Splunk & Java Experts (2026-current, sourced)

Everything below is drawn from **official Splunk documentation, Splunk's own engineering blog, official Spring Boot docs, and named practitioner write-ups** — checked against 2026-current versions so it stays compatible with what you'll actually see in a modern Splunk Enterprise/Cloud + Spring Boot 3.x stack. No claims here are invented; each point links to its source so you can verify it yourself.

### From Splunk's official documentation (help.splunk.com, current 2026 releases)
- The single biggest lever for search speed is narrowing the time range as tightly as possible instead of relying on the default lookback window, and understanding which indexes/sources/sourcetypes actually hold your data before writing filters. This is exactly why §2 of this cheatsheet insists on `index=`/`sourcetype=` before anything else.
- Dropping unneeded fields early with `fields` measurably reduces the work Splunk does downstream — confirms the `| fields` habit in §2 isn't just folklore, it's Splunk's own documented optimization technique.
- Splunk explicitly documents restructuring searches so filtering criteria happen *before* expensive commands like `lookup`/`eval` run against the full result set, rather than filtering afterward — i.e., push `where`/search-term filtering as early in the pipe chain as possible.
- Source: [help.splunk.com — About search optimization](https://help.splunk.com/en/splunk-cloud-platform/search/search-manual/10.4.2604/optimizing-searches/about-search-optimization) · [Quick tips for optimization](https://help.splunk.com/en/splunk-enterprise/search/search-manual/10.2/optimize-searches/quick-tips-for-optimization)

### From Splunk's official "Clara-fication" engineering blog (by Clara Merriman, Senior Splunk Engineer, and Martin Müller, Principal Consultant — also a .conf20 session, TRU1143C)
- The Job Inspector can be configured (via a `limits.conf` stanza) to surface debug messaging plus subsearch/lispy output directly in the top message bar, and separately to show per-peer execution cost breakdown — invaluable when an RCA search feels slow and you need to know *which indexer peer* is the bottleneck, not just that the search is slow overall.
- When benchmarking whether a search rewrite actually helped, the recommended practice is to compare scanCount and results/events-per-second between the original and optimized versions across multiple runs, rather than trusting a single run's wall-clock time.
- Source: [Splunk > Clara-fication: Job Inspector](https://www.splunk.com/en_us/blog/tips-and-tricks/splunk-clara-fication-job-inspector.html) · [Splunk > Clara-fication: Search Best Practices](https://www.splunk.com/en_us/blog/customers/splunk-clara-fication-search-best-practices.html)

### From a senior Splunk performance-tuning practitioner (Gareth Anderson, actively maintained Medium series, updated as recently as July 2026)
- At scale, admins are encouraged to enforce guardrails rather than rely purely on individual users self-optimizing — using Splunk's workload management (WLM) admission rules to block runaway searches, and `srchMaxTime` in `authorize.conf` to cap total search runtime.
- **Practical takeaway for you as an app developer**: if your ad-hoc RCA search gets killed mid-run in a shared prod Splunk environment, it's very likely hitting an admin-enforced WLM/`srchMaxTime` guardrail, not a bug — narrow your time range and index scope rather than fighting it.
- Source: [Splunk performance tuning tips — search head tier, Gareth Anderson](https://medium.com/@gjanders03/splunk-performance-tuning-tips-search-head-tier-d211ad679cc0)

### From official Spring Boot reference documentation (docs.spring.io — Spring Boot 3.x, current)
- Spring Boot auto-populates a correlation ID in log output by default when Micrometer Tracing is in use, built from the MDC `traceId` and `spanId` values, and this format is fully customizable via the `logging.pattern.correlation` property — so if your team's logs don't show `[traceId-spanId]`, it's a config gap, not a Splunk ingestion problem.
- This confirms and refines §4 of this cheatsheet: the exact MDC keys to expect are literally `traceId` and `spanId` (lowercase, camelCase) when using current Micrometer Tracing — not the older Sleuth-era `X-B3-TraceId` header-casing, which is now legacy.
- Source: [Spring Boot Reference — Tracing](https://docs.spring.io/spring-boot/reference/actuator/tracing.html)

### From current Spring/observability practitioner write-ups (2025–2026)
- Spring Boot 3's default context-propagation format is W3C Trace Context (the `traceparent`/`tracestate` headers), with B3 (Zipkin-style `X-B3-*` headers) supported as an alternative — but whichever standard is chosen, every service in the mesh must use the same one consistently, or trace continuity breaks silently at the boundary. This is the single most common root cause behind the "broken trace" scenario in §7A query #4 of this cheatsheet.
- Sampling probability is commonly reduced from 100% in lower environments down to something like 10% in production to control tracing overhead and data volume — which is *exactly* why, in prod, you'll sometimes find a `traceId` in your app logs with no matching spans in your tracing backend (Splunk APM/Tempo/Jaeger/Zipkin): the log was sampled-in for logging but the span itself was sampled-out for tracing. Don't assume a tracing bug — check the sampling config first.
- Custom MDC fields (userId, tenantId, orderId, etc.) remain valuable alongside auto-injected traceId/spanId because they carry business context that pure tracing IDs can't convey — validates keeping `MDC.orderId`-style fields (§4) even in a fully Micrometer/OTel-instrumented stack.
- Sources: [Distributed Tracing in Spring Boot 3 — Last9](https://last9.io/blog/distributed-tracing-with-spring-boot/) · [Implementing Micrometer Tracing in Spring Boot — Anil R](https://medium.com/dev-spring/implementing-micrometer-tracing-in-spring-boot-dead0968b4fa) · [Logging in Distributed Systems — Priya Srivastava](https://medium.com/@priyasrivastava18official/logging-in-distributed-sytem-from-basic-logs-to-distributed-traces-your-spring-boot-guide-2ae15ed4903c)

### Recognized names/resources worth following directly (community-vetted, not just this doc's summary)
- **Splunk's official blog** maintains a dedicated tips-and-tricks series; the "Clara-fication" series specifically (Clara Merriman) is the most consistently cited source for search-optimization mechanics among senior Splunk admins.
- **Splunk Cheatsheets by Aplura** and the **Splunk blog by Mason Morales** are frequently pointed to inside Splunk's own community content as high-value independent references for knowledge-object management and admin-level tuning.
- **docs.spring.io "Actuator: Tracing"** is the canonical, always-current source for Micrometer Tracing configuration — treat any blog post (including the ones cited above) as secondary to this when Spring Boot version specifics matter, since micro-details shift between minor versions.
- For the theoretical foundation of the traceId/spanId/parent-child model itself, the original reference point across the industry (including Splunk APM, Jaeger, Zipkin, and OpenTelemetry) is Google's 2010 **"Dapper" paper** on large-scale distributed tracing — useful background if you want to understand *why* the span-tree model works the way it does, independent of any specific vendor.

> **Verification note**: the Splunk doc pages above are dated 2026 in their own "last updated" metadata, and the Spring Boot/Micrometer sources reflect the current Spring Boot 3.x + Micrometer Tracing (post-Sleuth) stack, not the deprecated Spring Cloud Sleuth approach. If your codebase still shows `spring-cloud-starter-sleuth` dependencies, that's a legacy setup worth flagging for migration — Sleuth is no longer the recommended path.
