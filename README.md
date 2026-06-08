# MarkDesk: Architecture Overview

**AI-powered trademark prosecution platform.** Built and operated solo, covering data ingestion, production ML inference, and observability.

Live at [markdesk.wolff.sh](https://markdesk.wolff.sh)

---

## What It Does

MarkDesk helps small business owners file USPTO trademark applications and respond to office actions (examiner rejections). The platform:

- Ingests the full USPTO trademark register (27M+ records) from government bulk XML data
- Provides real-time clearance search with phonetic similarity matching
- Classifies marks against the USPTO Acceptable Identification of Goods and Services Manual
- Drafts office action responses via a multi-stage AI pipeline with attorney review gates
- Monitors trademark status changes and sends deadline alerts

---

## System Architecture

```mermaid
graph TB
    subgraph External["External Data Sources"]
        USPTO["USPTO Open Data Portal<br/>(Bulk XML · 27M records)"]
        STRIPE["Stripe<br/>(Payments + Webhooks)"]
    end

    subgraph Ingestion["Data Ingestion Layer"]
        DAILY["Daily Sync Job<br/>(cron · idempotent)"]
        PARSER["Streaming XML Parser<br/>(SAX · line-by-line)"]
        PHONETIC["Phonetic Index Builder<br/>(Double Metaphone)"]
    end

    subgraph App["Application Layer (Next.js)"]
        API["API Routes"]
        MW["Middleware<br/>(Auth · Rate Limit)"]
        QUEUE["Async Job Queue<br/>(SQLite-backed)"]
        DELIVERY["Delivery Processor"]
    end

    subgraph AI["AI Inference Pipeline"]
        S1["Stage 1: Context Enrichment"]
        S2["Stage 2: Parallel Drafting"]
        S3["Stage 3: Validation"]
        S4["Stage 4: Assembly"]
    end

    subgraph Data["Data Layer"]
        SQLITE["SQLite (WAL mode)<br/>27M records · 46GB"]
        FTS["FTS5 Full-Text Index"]
        PHIDX["Phonetic Index<br/>(Double Metaphone codes)"]
    end

    subgraph Infra["Infrastructure & Observability"]
        LITESTREAM["Litestream<br/>(WAL → S3 replication)"]
        CW["CloudWatch<br/>(Metrics + Alarms)"]
        S3["S3<br/>(Backup + State)"]
        TF["Terraform<br/>(IaC)"]
        GHA["GitHub Actions<br/>(CI/CD)"]
    end

    USPTO -->|"Daily XML (1GB/day)"| DAILY
    DAILY --> PARSER
    PARSER -->|"Batch INSERT OR REPLACE"| SQLITE
    PARSER --> PHONETIC
    PHONETIC --> PHIDX

    STRIPE -->|"Webhook"| API
    API --> MW
    MW --> QUEUE
    QUEUE -->|"Claim + Process"| DELIVERY
    DELIVERY --> S1

    S1 --> S2
    S2 --> S3
    S3 --> S4

    SQLITE --- FTS
    SQLITE --- PHIDX
    SQLITE -->|"Continuous WAL replication"| LITESTREAM
    LITESTREAM --> S3

    API -->|"Search queries"| FTS
    API -->|"Phonetic lookup"| PHIDX

    CW -.->|"Alarms → SNS → Email"| API
    GHA -->|"Deploy on push"| App
    TF -->|"Manages"| Infra
```

---

## Data Ingestion Pipeline

The USPTO publishes the full US trademark register as bulk XML. No search API exists. I built an ingestion pipeline that maintains a local replica with <48h freshness.

```mermaid
flowchart LR
    subgraph Source["USPTO Open Data Portal"]
        ANNUAL["Annual Backfile<br/>(22GB · 177 files · 1884–present)"]
        DAILY["Daily Deltas<br/>(~60MB zip · ~1GB XML · ~63K records/day)"]
    end

    subgraph Ingest["Ingestion"]
        FETCH["Fetch Latest<br/>(API key auth · follow redirects)"]
        MARKER["Idempotency Check<br/>(.done_filename marker)"]
        DECOMP["Decompress<br/>(zip → XML)"]
        SAX["SAX Stream Parser<br/>(line-by-line · no DOM)"]
    end

    subgraph Transform["Transform"]
        CASE["Extract Case File<br/>(40+ fields per record)"]
        PHON["Compute Phonetic Codes<br/>(Double Metaphone)"]
        PROS["Extract Prosecution History<br/>(event codes → office actions)"]
        CLASS["Extract Classifications<br/>(Nice classes + descriptions)"]
    end

    subgraph Load["Load (SQLite · WAL)"]
        BATCH["Batch Transaction<br/>(1000 records/tx)"]
        UPSERT["INSERT OR REPLACE<br/>(serial_number is PK)"]
        IDX["Index Rebuild<br/>(FTS5 · phonetic · status)"]
    end

    subgraph Guards["Operational Guards"]
        DISK["Disk Space Check<br/>(abort if <2GB free)"]
        MAINT["Maintenance Lock<br/>(skip if DDL in progress)"]
        FRESH["Freshness Metric<br/>(hours since newest record)"]
    end

    DAILY --> FETCH
    FETCH --> MARKER
    MARKER -->|"Already imported"| SKIP["Skip (exit 0)"]
    MARKER -->|"New file"| DECOMP
    DECOMP --> SAX
    SAX --> CASE
    CASE --> PHON
    CASE --> PROS
    CASE --> CLASS
    PHON --> BATCH
    PROS --> BATCH
    CLASS --> BATCH
    BATCH --> UPSERT
    UPSERT --> IDX

    DISK -.-> FETCH
    MAINT -.-> FETCH
    FRESH -.-> UPSERT
```

**Design decisions:**
- **SAX over DOM**: Daily XML files are ~1GB. DOM parsing would OOM a 2GB instance. Line-by-line streaming keeps RSS under 200MB.
- **Idempotent markers**: `.done_<filename>` files prevent re-import on cron retry or server restart. The check is a single `stat()`, no DB query needed.
- **Batch transactions**: 1000-record SQLite transactions balance write throughput (~50K inserts/sec) against WAL growth (Litestream must keep up).
- **Maintenance lock**: Long-running DDL (CREATE INDEX on 46GB DB) and daily imports cannot run concurrently because both saturate disk I/O on the single-disk instance. A lock file prevents overlap.

---

## Multi-Stage AI Inference Pipeline

Office action responses require structured legal arguments. A single LLM call can't reliably produce a complete, validated response, so the pipeline is decomposed into four stages with distinct failure modes and retry semantics.

```mermaid
flowchart TB
    subgraph Trigger["Trigger"]
        WEBHOOK["Stripe Webhook<br/>(checkout.session.completed)"]
        ENQUEUE["Enqueue Job<br/>(type: oa_response)"]
    end

    subgraph S1["Stage 1 — Context Enrichment"]
        PDF["Parse OA PDF<br/>(Claude Vision)"]
        CLASSIFY["Classify Refusal Types<br/>(taxonomy mapping)"]
        CITATIONS["Resolve Cited Marks<br/>(DB lookup)"]
        HISTORY["Fetch Prosecution History"]
    end

    subgraph S2["Stage 2 — Parallel Drafting"]
        FAN["Fan-out per Refusal<br/>(p-limit concurrency cap)"]
        DRAFT1["Draft: Refusal A"]
        DRAFT2["Draft: Refusal B"]
        DRAFTN["Draft: Refusal N"]
        RETRY["Retry (exponential)<br/>[1s, 3s, 9s]"]
        SYNTH["Synthetic Placeholder<br/>[ATTORNEY INPUT NEEDED]"]
    end

    subgraph S3["Stage 3 — Validation"]
        VALIDATE["Structural Validation<br/>(citations · sections · format)"]
        REGEN["Corrective Regeneration<br/>(failed sections only)"]
        REVAL["Re-validate (attempt 2)"]
    end

    subgraph S4["Stage 4 — Assembly"]
        ASSEMBLE["Deterministic Assembly<br/>(no LLM calls)"]
        RENDER["Render PDF"]
        REVIEW["→ Attorney Review Queue"]
    end

    subgraph Cost["Cost Controls"]
        TRACK["Per-job Token Tracking<br/>(input + output)"]
        WARN["Warn at $0.50/draft"]
        CAP["Hard cap at $1.00/draft"]
    end

    WEBHOOK --> ENQUEUE
    ENQUEUE --> PDF
    PDF --> CLASSIFY
    CLASSIFY --> CITATIONS
    CITATIONS --> HISTORY
    HISTORY --> FAN

    FAN --> DRAFT1
    FAN --> DRAFT2
    FAN --> DRAFTN

    DRAFT1 -->|"Success"| VALIDATE
    DRAFT2 -->|"Success"| VALIDATE
    DRAFTN -->|"Success"| VALIDATE

    DRAFT1 -->|"Failure"| RETRY
    RETRY -->|"Exhausted"| SYNTH
    SYNTH --> VALIDATE

    VALIDATE -->|"Pass"| ASSEMBLE
    VALIDATE -->|"Fail (attempt 1)"| REGEN
    REGEN --> REVAL
    REVAL -->|"Pass or Fail"| ASSEMBLE

    ASSEMBLE --> RENDER
    RENDER --> REVIEW

    TRACK -.-> DRAFT1
    TRACK -.-> DRAFT2
    WARN -.-> TRACK
    CAP -.-> TRACK
```

**Design decisions:**
- **Parallel fan-out with concurrency limit**: Multiple refusals are drafted concurrently (`p-limit` semaphore) to stay within provider rate limits. `Promise.allSettled` ensures one failure doesn't discard completed work.
- **Corrective regeneration**: On validation failure, only the *failing sections* are regenerated with the validation issues injected as corrective context. One regeneration attempt maximum to prevent infinite loops.
- **Synthetic placeholders**: If generation fails after retries, a well-formed placeholder fills the slot. The document is still structurally valid and the attorney review step catches the gap. Graceful degradation over hard failure.
- **Cost tracking per job**: Every inference records input/output tokens and computes cost at the model's per-token rate. A warn threshold at $0.50 catches prompt bloat before the hard ceiling. Costs are queryable per-order for margin analysis.
- **Deterministic assembly**: Stage 4 never calls an LLM. It renders the validated drafts into the final document format. This separation means assembly is fast, cheap, and reproducible.

---

## Async Job Queue & Delivery

Payment events trigger async work (AI generation, PDF rendering, email delivery). The queue is SQLite-backed with at-least-once semantics.

```mermaid
stateDiagram-v2
    [*] --> Pending: enqueue()
    Pending --> Processing: claimNextJob()
    Processing --> Complete: completeJob()
    Processing --> Pending: Transient failure<br/>(attempts < max)
    Processing --> Failed: Permanent failure<br/>(attempts >= max)
    Failed --> [*]
    Complete --> [*]

    note right of Processing
        Exponential backoff:
        [5s, 30s, 2min, 10min, 1hr]
        
        Claim semantics:
        SELECT ... ORDER BY created_at
        UPDATE status = processing
        (single transaction)
    end note

    note right of Failed
        Alarm fires on
        DeliveryFailures > 0
        (5-min window)
    end note
```

```mermaid
flowchart LR
    subgraph Queue["Job Lifecycle"]
        POLL["Poll Loop<br/>(configurable interval)"]
        CLAIM["Claim Oldest Ready Job<br/>(atomic SELECT + UPDATE)"]
        EXEC["Execute<br/>(type-dispatched)"]
        BACKOFF["Compute next_retry_at<br/>(exponential delays)"]
    end

    subgraph Types["Job Types"]
        APP["application_delivery<br/>(AI classify → PDF → email)"]
        OA["oa_response<br/>(4-stage pipeline → PDF → review queue)"]
        MON["monitoring_setup<br/>(subscription activation)"]
    end

    subgraph Side["Side Effects"]
        METRICS["Record AI usage<br/>(tokens, cost, model)"]
        STATUS["Update order status"]
        RETENTION["Retention sweep<br/>(hourly, declarative policies)"]
    end

    POLL --> CLAIM
    CLAIM --> EXEC
    EXEC -->|"Success"| STATUS
    EXEC -->|"Failure"| BACKOFF
    BACKOFF --> POLL

    EXEC --> APP
    EXEC --> OA
    EXEC --> MON

    EXEC --> METRICS
    POLL -->|"Once/hour"| RETENTION
```

**Design decisions:**
- **SQLite as queue**: At current scale (<100 jobs/day), a dedicated message broker adds operational overhead without benefit. The queue table lives in the same DB as the application, so atomic cross-table transactions (order status + queue insert) are trivial.
- **At-least-once delivery**: Jobs are idempotent. Re-processing a delivery just re-sends an email. The customer gets a duplicate at worst, never a lost order.
- **Declarative retention**: Completed/failed jobs are swept after 30 days. Retention policies are declared as a data structure, not scattered across codebase.

---

## Observability & Operational Maturity

Single-person operation demands automated detection. The system publishes 6 CloudWatch metrics (within free tier) and fires alarms for 7 failure modes.

```mermaid
flowchart TB
    subgraph Collection["Metric Collection (cron · every 5 min)"]
        HEALTH["HTTP health check<br/>(binary: 1=up, 0=down)"]
        MEM["Node.js RSS<br/>(MB)"]
        DISK["Disk usage<br/>(%)"]
        QUEUE_M["Queue depth<br/>(pending + processing)"]
        FAIL["Delivery failures<br/>(permanently failed jobs)"]
        FRESH["Data freshness<br/>(hours since newest record)"]
    end

    subgraph Publish["CloudWatch"]
        NS["Namespace: MarkDesk"]
        PUT["PutMetricData<br/>(6 metrics · free tier)"]
    end

    subgraph Alarms["Alarms (Terraform-managed)"]
        A1["health-check-failed<br/>(Minimum < 1 for 15min)"]
        A2["disk-usage-high<br/>(Maximum > 90% for 10min)"]
        A3["memory-high<br/>(Maximum > 1500MB for 15min)"]
        A4["metrics-absent<br/>(SampleCount < 1 for 20min)"]
        A5["data-stale<br/>(Minimum > 96h for 3h)"]
        A6["delivery-failures<br/>(Sum ≥ 1 in 5min)"]
        A7["queue-backup<br/>(Maximum > 5 for 10min)"]
    end

    subgraph Response["Response"]
        SNS["SNS Topic"]
        EMAIL["Email Alert"]
    end

    subgraph Backup["Continuous Backup"]
        WAL["SQLite WAL"]
        LITE["Litestream<br/>(v0.3.13)"]
        S3B["S3 Bucket<br/>(lifecycle: 30d retention)"]
    end

    HEALTH --> PUT
    MEM --> PUT
    DISK --> PUT
    QUEUE_M --> PUT
    FAIL --> PUT
    FRESH --> PUT

    PUT --> NS
    NS --> A1
    NS --> A2
    NS --> A3
    NS --> A4
    NS --> A5
    NS --> A6
    NS --> A7

    A1 --> SNS
    A2 --> SNS
    A3 --> SNS
    A4 --> SNS
    A5 --> SNS
    A6 --> SNS
    A7 --> SNS
    SNS --> EMAIL

    WAL -->|"Continuous replication"| LITE
    LITE --> S3B
```

**Design decisions:**
- **Metric budget discipline**: Only metrics consumed by alarms are shipped to CloudWatch. Diagnostic metrics (DB size, mark count, AI cost) are logged locally but not published. Avoids the $0.30/metric/month cost creep.
- **`treat_missing_data = breaching`** on health check and metrics-absent: If the cron stops running (server down, OOM kill), silence itself is the alarm signal. No "silent failure" gap.
- **Litestream over periodic snapshots**: WAL-based continuous replication means RPO is seconds, not hours. A weekly full-DB copy (`aws s3 cp` of 46GB) was eliminated after it caused disk contention and SIGBUS crashes. The continuous WAL approach is both safer and cheaper.
- **Data freshness SLA**: USPTO publishes no data Fri–Mon. The 96-hour threshold accounts for weekends without false-alarming. Three consecutive hourly breaches are required to fire, which prevents transient import delays from paging.

---

## CI/CD & Deploy Pipeline

Push-to-deploy with structural guards, platform-aware native module handling, and post-deploy verification.

```mermaid
flowchart LR
    subgraph Trigger["Trigger"]
        PUSH["Push to mainline<br/>(src/ or public/ changed)"]
        PR["Pull Request"]
    end

    subgraph CI["CI Checks (PR only)"]
        UNIT["Unit Tests<br/>(vitest)"]
        GUARD["Structural Guards<br/>(pattern enforcement)"]
    end

    subgraph Build["Build"]
        NEXT["next build<br/>(standalone output)"]
        BUNDLE["Bundle monitor script<br/>(CommonJS · no tsx dep)"]
        STATIC["Copy static assets<br/>(public/ + .next/static/)"]
    end

    subgraph Deploy["Deploy"]
        RSYNC["rsync to Lightsail<br/>(exclude: data/, .env, native builds)"]
        NATIVE["Rebuild better-sqlite3<br/>(macOS→Linux binary mismatch)"]
        RESTART["systemctl restart"]
    end

    subgraph Verify["Post-Deploy Verification"]
        WARM["Cache warmup<br/>(mmap 46GB DB)"]
        SMOKE["Smoke tests vs prod<br/>(Playwright)"]
        WARN_F["Warning annotation<br/>(on smoke failure)"]
    end

    PR --> UNIT
    PR --> GUARD
    PUSH --> NEXT
    NEXT --> BUNDLE
    BUNDLE --> STATIC
    STATIC --> RSYNC
    RSYNC --> NATIVE
    NATIVE --> RESTART
    RESTART --> WARM
    WARM --> SMOKE
    SMOKE -->|"Failure"| WARN_F
```

**Design decisions:**
- **Structural guards**: Automated tests that grep the source tree for anti-patterns (re-inlined utilities, duplicate implementations, unauthorized constants). They enforce consolidation not through code review discipline, but through CI failure.
- **Native module rebuild on deploy**: `better-sqlite3` compiles a C++ addon. Build machine (macOS/GitHub runner) and server (Ubuntu) have different ABIs. The deploy rebuilds the native module in-place rather than shipping the whole `node_modules/`.
- **Cache warmup before smoke**: A 46GB memory-mapped SQLite DB has a cold start where first queries hit disk. The deploy warms the search path explicitly so smoke tests don't false-fail on timeout.
- **Non-blocking smoke failure**: Smoke tests run against live prod but don't roll back on failure (a rollback is riskier than investigating). Instead, a GitHub annotation flags the issue for human review.

---

## Search Architecture

Clearance search requires both exact text matching and phonetic similarity detection (trademarks that *sound alike* can conflict even if spelled differently).

```mermaid
flowchart TB
    subgraph Query["User Query: 'Kavatina'"]
        INPUT["Raw input"]
    end

    subgraph Parallel["Parallel Search Paths"]
        FTS["FTS5 Full-Text Search<br/>(exact + prefix matching)"]
        PHON["Phonetic Lookup<br/>(Double Metaphone → index scan)"]
    end

    subgraph Scoring["Reciprocal Rank Fusion"]
        RRF["RRF Score<br/>(k=60)"]
        JW["Jaro-Winkler Similarity<br/>(string distance)"]
        STATUS["Status Weight<br/>(Registered > Pending > Dead)"]
        RECENCY["Recency Weight<br/>(newer filings ranked higher)"]
    end

    subgraph Ranking["Final Ranking"]
        TIER["Tier Sort<br/>(status-based: live > pending > dead)"]
        SCORE["Score Sort<br/>(within tier)"]
        RESULTS["Top-K Results<br/>(with conflict risk indicators)"]
    end

    INPUT --> FTS
    INPUT --> PHON

    FTS -->|"rank position"| RRF
    PHON -->|"rank position"| RRF

    RRF --> TIER
    JW --> TIER
    STATUS --> TIER
    RECENCY --> TIER

    TIER --> SCORE
    SCORE --> RESULTS
```

**Design decisions:**
- **Dual-path search**: Exact text match catches direct conflicts; phonetic match catches sound-alikes ("Kavatina" ↔ "Cavatina"). Both are required for proper clearance. Missing either path creates false confidence.
- **Reciprocal Rank Fusion**: Merges ranked lists from different retrieval methods without needing score normalization. Simple, effective, well-studied in IR literature.
- **Status-tiered ranking**: A live registered mark (status 800) is a hard blocker regardless of textual similarity. Tier is the primary sort; score only breaks ties within a tier. This matches how trademark attorneys actually assess risk.
- **Double Metaphone over Soundex**: Soundex is English-only and loses too much information. Double Metaphone handles multi-language phonetics and produces primary + alternate codes, catching more potential conflicts.

---

## Infrastructure as Code

All cloud resources are Terraform-managed with state locking (S3 + DynamoDB), scoped IAM for CI, and prevent-destroy on critical resources.

```mermaid
flowchart TB
    subgraph TF["Terraform (infra/)"]
        STATE["Remote State<br/>(S3 + DynamoDB lock)"]
        PLAN["Plan on PR<br/>(GitHub Actions)"]
        APPLY["Apply on merge<br/>(manual trigger)"]
    end

    subgraph Resources["Managed Resources"]
        INSTANCE["Lightsail Instance<br/>(Ubuntu 22.04)"]
        IP["Static IP"]
        PORTS["Firewall Rules<br/>(22, 80, 443)"]
        KEY["SSH Key Pair"]
        BUCKET["S3 Backup Bucket<br/>(lifecycle: 30d)"]
        ALARMS_R["7 CloudWatch Alarms"]
        SNS_R["SNS Alert Topic"]
        IAM["CI IAM User<br/>(scoped policy · prevent_destroy)"]
    end

    subgraph Security["IAM Design"]
        SCOPED["Resource-scoped Actions<br/>(no wildcard ARNs)"]
        PREFIXED["Name-prefixed Resources<br/>(markdesk-* only)"]
        READONLY["Separate readonly profiles<br/>(for monitoring)"]
        NODELETE["prevent_destroy on<br/>IAM + state resources"]
    end

    TF --> Resources
    STATE --> TF
    PLAN --> TF
    APPLY --> TF

    IAM --> SCOPED
    IAM --> PREFIXED
    IAM --> READONLY
    IAM --> NODELETE
```

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| App | Next.js 15 / TypeScript | SSR + API routes in one deployable, standalone output mode for self-hosting |
| Database | SQLite (WAL) + better-sqlite3 | 27M records, read-heavy, single-writer. No connection pooling overhead. |
| Search | FTS5 + Double Metaphone | Sub-50ms searches across 27M records with phonetic similarity |
| AI | Claude API (Anthropic SDK) | Vision for PDF parsing, text generation for drafting, structured output for classification |
| Payments | Stripe (Checkout + Webhooks) | Idempotent webhook handling with transactional order creation |
| Email | Resend | Transactional delivery notifications + monitoring alerts |
| Infra | AWS Lightsail + Terraform | Single-instance simplicity with IaC discipline |
| Backup | Litestream → S3 | Continuous WAL replication, seconds RPO |
| CI/CD | GitHub Actions | Push-to-deploy with structural guards + post-deploy smoke |
| Observability | CloudWatch (custom metrics + alarms) | 6 metrics, 7 alarms, free tier |
| Reverse Proxy | Caddy | Auto-HTTPS via Let's Encrypt, zero config TLS |

---

## Lessons Learned (Operational)

A few hard-won lessons from operating this solo:

1. **Don't `aws s3 cp` a live WAL-mode database.** Multi-hour uploads get BadDigest errors from concurrent checkpoints. Litestream's continuous replication solves this properly.

2. **Silence is a signal.** `treat_missing_data = breaching` on health metrics means a dead cron (OOM, disk full, crashed agent) alarms immediately instead of going unnoticed for hours.

3. **DDL on a single-disk instance is surgery.** A `CREATE INDEX` on 46GB competes for I/O with the live app's mmap'd reads. I wrote a DDL runbook with pre-checks (cron pause, deploy hold, disk headroom) after a SIGBUS outage taught me the hard way.

4. **Native binary mismatches are silent killers.** `better-sqlite3` built on macOS won't load on Linux. The deploy pipeline rebuilds it in-place on the server, not during the build step.

5. **Metric budget matters at small scale too.** CloudWatch charges $0.30/metric/month beyond 10. Only shipping alarmed metrics keeps the bill at $0 while diagnostic data stays in local logs.

---

## Contact

**Matthew Wolff** · [GitHub](https://github.com/matthewwolff) · [Site](https://matthewwolff.github.io)
