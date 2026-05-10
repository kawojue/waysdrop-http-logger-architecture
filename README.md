# HTTP Request Logger Architecture

## Scope

This document explains how API HTTP traffic is captured, buffered outside the database, deduplicated, and persisted in batches. The goal is a fast request path: the hot pipeline only enqueues serializable snapshots to an in-memory store; durable logging and identity resolution happen asynchronously on a fixed schedule.

---

## 1) System Layers

The logger is organized into three cooperating layers:

- **Capture Layer**: wraps each HTTP response lifecycle, records metadata and optional bodies, and prepares a payload for buffering without awaiting the database.
- **Buffer Layer**: holds ordered JSON lines in a Redis list so producers never block on Postgres and spikes do not translate into one insert per request.
- **Persistence Layer**: drains the list in bounded chunks, merges semantically duplicate lines inside each batch, and applies a single upsert per logical key to the logs table.

This separation keeps user-visible latency low while controlling write amplification and index pressure on the relational store.

---

## 2) Capture Semantics

### What gets logged

For each non-omitted route, the capture layer records method, normalized path, full URL (including protocol and host), client IP, user agent, timing (high-resolution duration formatted for humans), HTTP status, optional query string, device parsing (OS, browser, device class, CPU hint where available), request and response size labels, and optional structured request and response bodies. Handler metadata may attach a human-readable event title for audit trails.

### What is deliberately skipped

A fixed omit list excludes noise and long-lived streams: health checks, metrics, landing stubs, certain map streams, permission probes, and similar endpoints. Read-only maintenance mode allows GET traffic to bypass capture-side mutations while still restricting mutating verbs elsewhere.

### Response body capture

The stack monkey-patches the response object so that JSON, `send`, and streaming (`write`/`end`) paths funnel into a single semantic snapshot when possible. Parse failures fall back to an opaque wrapper so the row still reflects that something was returned.

### Sensitive data

Known secret patterns (passwords, tokens, card fields, OTPs, and related keys) are stripped from request and response bodies before buffering. Success GET responses may drop response bodies entirely when configured, to avoid caching large read payloads in the log store.

### When capture runs

Work that serializes the log line and touches the buffer is scheduled **after** the response pipeline completes, so the handler does not wait on Redis or auth side effects tied to logging. Errors still produce a synthetic response body with status and stack material for diagnostics, within the same sanitization rules.

---

## 3) Authentication Bridge

Buffered lines carry two pieces of state: the log row as captured, and a small token envelope used only downstream.

- **Bearer-style tokens** are verified when present on the request; successful verification marks the auth flavor as JWT and stores the subject for later identity resolution.
- **API keys** resolve to owning metadata; the log records the key id and auth flavor. Optional guard shortcuts (key id and owner already validated upstream) avoid a duplicate key lookup. Metering hooks tied to API key traffic—quota increments, last-used timestamps, optional usage de-duplication fingerprints, and asynchronous billing taps—run on this path when appropriate, but those concerns are orthogonal to flush batching: they are still per-request work, not part of the database batch insert.

- **Sentry** receives user context when a subject is known, so crashes correlate with the same identity inferred for the log row.

The buffer stores a JSON line pairing the log snapshot with the decoded subject (if any). Database primary keys for users or members are **not** written on the hot path; they are inferred in the persistence layer.

---

## 4) Buffer Layer Design

### Structure

New lines are pushed on one end of a Redis list; consumers drain from the opposite end in FIFO order. That ordering guarantees fairness under contention: oldest observations leave the buffer first.

### Atomic drain

Draining uses a single atomic script: take up to **N** entries in one shot and trim the list in the same operation so two workers cannot double-consume or drop rows between read and delete.

### Operational modes

When the deployment is configured for read-only behavior, list writes become no-ops so observability does not fight intentional freeze windows. Normal mode uses a configurable key namespace so multiple environments can coexist without collision.

### Durability tradeoff

Buffered data lives in Redis until flushed. Loss of that store between push and successful drain loses those lines; conversely, survives process restarts of the API tier. This is an intentional trade for latency and simplicity versus strict write-ahead to Postgres on every request.

---

## 5) Scheduled Flush

### Job model

A repeatable queue job fires on a wall-clock interval (minimum spacing enforced in configuration). Registration is idempotent: only one repeatable definition is kept per interval fingerprint so deploys do not stack duplicate schedulers.

### Worker behavior

The consumer runs flushes with **single-threaded concurrency** for this job type to avoid fighting itself. Before work, it attempts a short-lived distributed lock; if another instance holds it, the run exits quietly and the next tick retries. The lock TTL bounds stuck workers without blocking the pipeline forever.

### Drain loop

While the lock is held, the worker repeatedly drains up to a configured batch size, hands each batch to the persistence routine, and continues until a drain returns empty. A single scheduled invocation can therefore clear a backlog accumulated during an outage or slow database window.

---

## 6) Persistence: Identity and Merge

### Parsing and revival

Each buffered line is parsed defensively; malformed lines are skipped. Date fields that serialized as strings are revived into real timestamps before SQL generation.

### Identity resolution

Distinct token subjects across the batch are collected once. Two batched queries resolve which subjects are platform users versus members. Each log row then receives the appropriate foreign keys. Subjects that match neither set remain anonymous at the relational level (subject still present only inside the ephemeral buffer until merged).

### In-batch deduplication key

Rows are grouped by the tuple: client IP, HTTP method, full URL, status code, and resolved user or member id (empty string when absent). This mirrors the uniqueness rule enforced in the database so repeated traffic collapses to one row per burst window.

### Merge behavior

Within each group, the **last** observation wins for scalar and JSON columns—bodies, timing, titles, auth metadata, and so on—matching intuitive “latest state of this signature” semantics. A counter **`triggered`** accumulates how many original HTTP events folded into that group; on insert it seeds the count, on conflict it **adds** the new delta. That preserves volume signal without storing one physical row per click during floods.

### Database write shape

Merged rows are written in chunks with raw SQL for efficiency. Each chunk uses a multi-row insert with an **upsert** conflict target aligned to the expression unique index (IP, method, full URL, status, coalesced user id, coalesced member id). On conflict, scalars refresh from the new payload, **`triggered`** increments by the batch delta, timestamps update, and a **job identifier** field records which flush execution last touched the row (useful for tracing back-pressure episodes, not for per-request correlation).

### Deadlock resilience

Concurrent flushes from other writers touching overlapping rows can cause transient deadlocks; the executor retries with bounded exponential backoff and jitter before surfacing failure.

---

## 7) Uniqueness in the Database

Before the expression index existed, duplicate logical rows could accumulate. A preparatory migration collapses historical duplicates by combining **`triggered`** and keeping a survivor row, then installs the unique index so the upsert conflict target is enforceable. The ORM schema may not declare that expression index explicitly; it is a migration-level contract the flush SQL depends on.

---

## 8) Configuration Surface

Tune behavior with environment-driven settings (with safe clamps and defaults in the shared config module):

- **`LOG_BATCH_SIZE`** — Maximum lines pulled from Redis per drain call; also reused elsewhere for log retention batching. Bounded to prevent accidental multi-thousand megabatches.
- **`LOG_FLUSH_INTERVAL_MS`** — Repeatable job cadence; floored so schedules cannot pulse faster than a sane minimum.
- **`LOG_BUFFER_REDIS_KEY`** — Logical list name for the buffer.

Related API metering knobs (usage de-duplication window and optional request-body digest inclusion) affect **who** gets charged or metered when a key is present, not the log merge key itself; they still run during capture so operators should read billing documentation alongside this one.

---

## 9) Consistency and Latency Characteristics

### Eventual visibility

Operators see new rows only after a successful flush. Worst-case lag is roughly the flush interval plus queue wait and database time. This is unsuitable for strict real-time per-request forensic dashboards without an additional streaming path.

### Count semantics

**`triggered`** approximates “how many HTTP exchanges matched this fingerprint since the row materialized,” not a globally perfect counter across all time if archival or pruning occurs. It is best interpreted as operational pressure within the retention window.

### Ordering across merges

The last writer in a batch defines field snapshots for that flush; rapid alternating clients behind the same NAT sharing identical URLs and statuses will collapse in ways that reflect final observed state, not a time series.

---

## 10) Failure Modes and Guards

- **Redis unavailable on push**: capture logs a warning and drops the line; the request itself already succeeded.
- **Flush lock contention**: secondary instances skip silently; the next interval retries.
- **Partial parse failures inside a batch**: invalid lines are ignored; valid lines still persist.
- **Database deadlock**: automatic retry with cap; persistent failure surfaces in worker logs.
- **Read-only server mode**: no buffer writes, so no phantom durability promises during freeze.

---

## 11) Operational Checklist

Monitor:

- Redis list length growth (backlog depth) and age of head entry.
- Flush job success/failure rates and duration.
- Postgres deadlock or lock wait metrics on the logs table.
- Ratio of **`triggered`** growth to insert rate (high collapse ratio implies heavy duplicate traffic or scraping).

Inspect:

- Repeatable job registry for duplicate flush definitions after bad deploys.
- Lock key TTL versus longest observed flush duration.

---

## 12) End-to-End Summary

1. Request enters the capture interceptor unless the route is omitted or the server is in a reduced logging mode.
2. Response completes; metadata and optional bodies are sanitized.
3. Auth side effects and Sentry context run; a compact envelope with subject is formed.
4. A JSON line is pushed to a Redis list FIFO buffer.
5. On schedule, one flush worker drains batches atomically, resolves identities, merges duplicates inside each batch, and upserts chunks into Postgres with counter increments on conflict.
6. Surviving historical duplicates are reconciled via the expression unique index and preparatory data merges.

Overall, the architecture trades immediate relational durability for predictable API latency and bounded database write volume, while still converging on one authoritative row per logical traffic signature and preserving aggregate volume through the **`triggered`** counter.
