# Performance and Scalability Analysis

**System:** Event-Driven Fiscal Document Processing (Brazilian SEFAZ Invoices)
**Date:** 2026-03-19
**Scope:** Pre-implementation architectural review

---

## Executive Summary

The architecture as described has five issues that will constrain throughput or cause failures under real load before any optimization work becomes necessary. The Gemini API rate limit is a hard ceiling that the architecture does not account for at all — adding more workers without a rate limiter does not increase throughput, it increases retry and DLQ pressure. The synchronous Classifier in the request path will degrade API response time proportionally with batch size. Worker co-location prevents independent scaling of the three processing branches. There is no batch completion signal, so the frontend polls without bound. PDF documents that fall into the scanned path pay the full cost twice (once for pdfplumber, once for the image pipeline). These are not edge cases; they are the expected behavior of the system under normal use.

The findings below are ordered by severity within each topic area.

---

## Finding 1 — Gemini API Rate Limits as Throughput Ceiling

**Severity: Critical**
**Topic: Throughput Bottlenecks**

### Problem

The architecture routes all image documents and all scanned PDFs through the Gemini API. There is no client-side rate limiter described anywhere. Gemini API quotas are per-project and expressed as requests-per-minute (RPM) and tokens-per-minute (TPM). As of early 2026, Flash-tier models enforce quotas in the range of 1,000–2,000 RPM for paid tiers, with per-second burst limits that are lower.

Without a client-side token bucket or semaphore, each Celery worker calls Gemini independently. When the fleet scales horizontally to consume the queue faster, all workers hit the rate limit simultaneously. The result is a cascade of 429 responses, Celery retries with exponential backoff (if configured), queue re-insertion, and eventually DLQ exhaustion. Adding workers in this configuration makes the throughput problem worse, not better, because:

1. More concurrent 429s exhaust retries faster.
2. Backoff storms increase effective latency per document.
3. Re-queued tasks from retries inflate queue depth, triggering further scaling events.

**Estimated impact:** At 1,000 RPM with a 5-second average Gemini call latency (including preprocessing), the maximum sustainable throughput for image/scanned-PDF documents is approximately **16–17 documents per second**, or roughly **1,000 documents per minute**, regardless of fleet size. A batch of 1,000 image documents takes at minimum 60 seconds end-to-end at this ceiling with zero contention. With retry overhead from uncontrolled concurrency, effective throughput can drop to 30–50% of theoretical maximum.

### Recommendation

Implement a distributed rate limiter in Redis using the token bucket algorithm, applied globally across the entire worker fleet before any Gemini call is made. A single Redis key holds the bucket state; workers atomically check and decrement the token count using a Lua script (to ensure atomicity). Workers that find the bucket empty wait and retry with a short sleep rather than submitting a task to Celery's retry mechanism.

```
Redis key: gemini:rate_limiter:tokens
Refill rate: N tokens per second (set to 90% of Gemini quota to leave headroom)
Burst cap: N tokens (set to Gemini's per-minute quota)
```

This approach allows the fleet to scale horizontally without generating 429s, because the rate is enforced before the HTTP call is attempted. The Celery retry mechanism then remains reserved for genuine transient failures (network errors, 500s) rather than predictable quota exhaustion.

Additionally, instrument Gemini call latency, 429 rate, and effective throughput as separate metrics from queue throughput. The queue drain rate and the Gemini throughput rate are different numbers and must be monitored independently.

---

## Finding 2 — Synchronous Classifier in the Request Path

**Severity: High**
**Topic: Latency in the Request Path**

### Problem

The Classifier is described as a "lightweight synchronous module inside the invoice-api" that performs magic byte inspection. Magic byte inspection requires reading the first N bytes of each file. The files have already been uploaded to Supabase Object Storage (S3-compatible) by the time the API receives the file URLs. Therefore, the Classifier must issue a ranged GET request (`Range: bytes=0-511` or similar) to S3 for each file in the batch before the API can respond to the client.

This introduces network I/O in the synchronous request path. The Hono API runs serverless on Vercel, which means:

1. Each request handler awaits all Classifier calls before publishing to Redis and returning a response.
2. For a batch of N files, the API issues N sequential or concurrent S3 ranged GETs before responding.
3. Vercel serverless functions have execution time limits (typically 10–60 seconds depending on plan tier).

**Estimated impact per file:**
- S3 ranged GET cold (cross-region): 80–200 ms
- S3 ranged GET warm (same-region): 20–60 ms
- Assuming same-region deployment: 30 ms average per file

For a batch of 100 files processed sequentially: **3 seconds added to API response time.**
For 100 files processed concurrently (Promise.all): constrained by S3 connection concurrency limits and Vercel's outbound connection pool. At 20 concurrent connections, the 100-file batch takes roughly 5 ceiling divisions × 30 ms = **150 ms**, but with connection establishment overhead and S3 throttling at burst, real numbers will be 300–800 ms.

For 1,000-file batches: with sequential processing, this exceeds Vercel's function timeout. With concurrent processing, S3 burst throttling becomes a significant factor, and the API becomes unresponsive for several seconds.

**The deeper problem:** the client is blocked waiting for a response while the API performs file inspection. This is unnecessary — the client only needs a batch ID and a confirmation that tasks were accepted. The classification result is only needed by the worker, not by the client at submission time.

### Recommendation

Move classification out of the synchronous request path entirely. The API should:

1. Accept the batch, generate a batch ID, write a pending record to PostgreSQL, and respond to the client immediately (< 50 ms response time).
2. Publish an unclassified task to Redis containing the file URL and batch ID.
3. Have the worker perform magic byte inspection as the first step of processing, before routing to the appropriate branch.

This changes the Classifier from a synchronous API module into the first stage of the worker pipeline. The response time for a 1,000-file batch submission drops from potentially 30+ seconds to under 100 ms. Workers already download the file for processing; the ranged GET for magic bytes can be folded into that download or performed as a cheap first step.

If pre-classification is strongly desired for routing optimization (e.g., to publish directly to branch-specific queues), it should be done asynchronously after the API has already responded — a background task spawned by the API handler, not a blocking call.

---

## Finding 3 — Worker Co-location Prevents Independent Scaling

**Severity: High**
**Topic: Scaling Strategies**

### Problem

All three processing branches (image, XML, PDF) run in the same ECS task definition and Celery worker pool. Their resource profiles are fundamentally different:

| Branch | CPU profile | Memory profile | I/O profile | Blocking factor |
|--------|-------------|----------------|-------------|-----------------|
| Image  | Medium (resize, contrast) | Medium (decoded bitmap in memory) | High (S3 download, Gemini HTTP) | Gemini API latency |
| XML    | Low (XSD parse, XPath) | Low (text only) | Low (S3 download) | None (CPU-bound, fast) |
| PDF    | High (pdfplumber, pdf2image render) | High (page rasterization) | High (S3 download) | pdf2image CPU |

The image branch workers spend most of their time waiting on Gemini HTTP responses. The XML branch workers are fast and CPU/memory light. The PDF branch workers are CPU-heavy when rasterizing pages.

When the queue has a mix of document types, all worker processes are competing for the same ECS task CPU and memory allocation. A surge of large scanned PDFs will consume all available memory in the shared task, starving XML workers that could otherwise drain their queue instantly. Scaling ECS tasks to handle PDF load scales all branches equally, which is wasteful.

**Estimated impact:** In a mixed batch of 33% image, 33% XML, 33% PDF: XML documents that would take < 1 second each may queue behind PDF rasterization jobs that take 5–15 seconds each and consume significantly more memory. XML throughput is artificially constrained to PDF throughput. Overprovisioning for PDF load increases cost for the XML branch by 5–10x relative to its actual resource needs.

### Recommendation

Split the single worker fleet into three separate Celery applications (or at minimum three separate Celery queues with dedicated worker pools scaled independently):

- `queue:image` → image workers (I/O-bound, optimize for concurrency; gevent or asyncio pool, 10–20 concurrent tasks per worker)
- `queue:xml` → XML workers (CPU-bound but fast; prefork pool, 4–8 processes per worker)
- `queue:pdf` → PDF workers (CPU/memory-heavy; prefork pool, 2–4 processes per worker, higher memory allocation per ECS task)

ECS scaling policies should be defined per queue depth:
- Image queue: scale on `ApproximateNumberOfMessagesVisible > 50`, limited by Gemini rate (see Finding 1)
- XML queue: scale on `ApproximateNumberOfMessagesVisible > 200` (fast drains, less aggressive scaling)
- PDF queue: scale on `ApproximateNumberOfMessagesVisible > 20` (expensive tasks, conservative scaling)

This also allows the PDF branch to be scaled to zero during off-hours if PDF volume is low, without affecting image or XML throughput.

---

## Finding 4 — No Batch Completion Detection

**Severity: High**
**Topic: Throughput Bottlenecks / Database Performance**

### Problem

The frontend polls the invoice-api for batch status. The API queries PostgreSQL to determine how many documents in the batch have a terminal status (completed or failed). There is no described mechanism to detect when all documents in a batch have reached a terminal state and notify the frontend.

This creates two interrelated problems:

**A. Unbounded polling:** The frontend does not know when to stop polling. If polling interval is 2 seconds, a 1,000-document batch that takes 10 minutes to process generates 300 polling requests per client session. For 100 concurrent users, that is 30,000 status queries over 10 minutes — all hitting PostgreSQL through the API.

**B. Status query cost:** Each status query requires a COUNT or row scan over the `documents` table filtered by `batch_id`. Without the right index, this is a sequential scan. With an index on `(batch_id, status)`, it is an index scan but still executes N times per poll interval per client. At high document volume, the `documents` table grows large, and even indexed queries add up.

**Estimated impact:** 100 concurrent users × 1 poll/2 seconds × 600 seconds = 30,000 queries against PostgreSQL during a 10-minute batch window. If each query takes 5 ms (indexed), that is 150 query-seconds of PostgreSQL time, which is not catastrophic but is entirely unnecessary work.

### Recommendation

**For batch completion detection:** Add a `batch_summary` table or a materialized counter per batch that tracks `(total_documents, completed_count, failed_count)`. Workers increment these counters atomically (PostgreSQL `UPDATE batch SET completed_count = completed_count + 1 WHERE id = $1`) rather than having the API recount on each poll. When `completed_count + failed_count = total_documents`, the batch is complete. This reduces each status poll to a single row read on the summary table rather than an aggregate scan over potentially thousands of document rows.

**For reducing poll frequency:** Implement SSE (already mentioned as an option in the architecture). When a batch summary counter crosses the completion threshold, the API server can push a completion event to the waiting SSE connection. This eliminates polling entirely for completion detection. Polling should only be used as a fallback.

**Index design for the documents table:**
```sql
CREATE INDEX idx_documents_batch_status ON documents (batch_id, status);
CREATE INDEX idx_documents_batch_id ON documents (batch_id) INCLUDE (status, updated_at);
```

If using SSE on Vercel serverless, note that SSE connections on serverless functions are constrained by function execution timeouts. For long-running batches, the SSE connection must reconnect or be handled by a persistent process (e.g., Railway or a dedicated Vercel Edge Function with streaming).

---

## Finding 5 — PDF Double-Processing Cost

**Severity: High**
**Topic: PDF Double-Processing Cost**

### Problem

The PDF branch attempts text extraction with pdfplumber first, then falls back to pdf2image + image pipeline if confidence is below threshold. For scanned PDFs, this means:

1. **S3 download** — full PDF file downloaded once (100 KB to 50 MB typical for SEFAZ documents)
2. **pdfplumber extraction** — PDF is parsed, text layer read; for scanned PDFs with no text layer, this returns empty or near-empty text quickly but still incurs PDF parsing overhead
3. **pdf2image rasterization** — each page is rendered to a bitmap (memory-intensive, CPU-intensive)
4. **Image preprocessing** — resize, contrast enhancement per page frame
5. **Gemini API call** — per page frame or per document

For a typical SEFAZ NFe (Nota Fiscal Eletrônica), the PDF is 1–4 pages. For NF-e with attached DANFE (document auxiliary), it may be up to 10 pages. Each page rasterized at 200 DPI produces a ~2–5 MB uncompressed bitmap before the resize step.

**Estimated cost for a 4-page scanned PDF:**
- pdfplumber parse: 200–500 ms (wasted for scanned)
- pdf2image rasterization (4 pages at 200 DPI): 2–6 seconds
- Image preprocessing (4 pages): 400 ms
- Gemini API calls (4 pages, serialized due to rate limit): 4 × 3–5 seconds = 12–20 seconds
- **Total: 15–27 seconds per scanned PDF document**

For a 100-document batch of all scanned PDFs, at the Gemini rate limit ceiling (Finding 1), this can take 30–40 minutes end-to-end.

**The wasted work:** pdfplumber always runs first, even for files that are obviously scanned. For PDFs generated by scanners, the text layer is absent or contains only garbled OCR artifacts. A 200–500 ms pdfplumber parse adds latency to every scanned PDF with no benefit.

### Recommendation

**Short-term:** Check whether the pdfplumber extraction is genuinely necessary before rasterization. If the PDF has zero text content (page.extract_text() returns None or empty string on the first page), skip pdfplumber entirely and route directly to pdf2image. This avoids full-document pdfplumber parsing for clearly scanned files.

**Medium-term:** Add a PDF metadata inspection step before full processing. pdfplumber (or pikepdf/PyPDF2) can read the PDF's `Producer` and `Creator` metadata fields and check whether any page has a text stream object, without rendering or extracting all pages. Scanner-produced PDFs have no text streams. This inspection takes < 50 ms and eliminates the wasted extraction attempt for the majority of scanned documents.

**On multi-page Gemini calls:** Evaluate whether all pages of a multi-page DANFE must be submitted to Gemini individually or whether the relevant fiscal fields (CNPJ, total value, item lines) appear on a known subset of pages. For standard DANFE format, the structured data is on page 1–2. Processing only those pages reduces Gemini calls proportionally.

---

## Finding 6 — Celery Pool Configuration for Mixed Workloads

**Severity: High**
**Topic: Worker Resource Utilization**

### Problem

Celery supports multiple execution pool backends: `prefork` (multiprocessing), `gevent` (greenlet-based cooperative concurrency), `eventlet`, and `solo`. The correct choice depends on whether tasks are CPU-bound or I/O-bound.

| Branch | Bound by | Correct pool | Concurrency setting |
|--------|----------|--------------|---------------------|
| Image  | I/O (Gemini HTTP wait) | gevent or threads | 10–20 per worker |
| XML    | CPU (lxml parsing) | prefork | N_CPU per worker |
| PDF    | CPU (rasterization) | prefork | 2–4 per worker (memory-limited) |

If all three branches run in the same Celery worker with a single pool type, one of these is wrong:

- Using `prefork` for image workers: each process blocks on Gemini HTTP with no I/O multiplexing. A 4-CPU ECS task running 4 prefork workers can only process 4 concurrent Gemini calls. With 5-second Gemini latency, that is 48 documents per minute per task — far below the Gemini API quota ceiling.
- Using `gevent` for XML workers: greenlet cooperative scheduling is fine for XPath extraction, but lxml releases the GIL only partially. Some lxml operations will block gevent event loop, causing other greenlets to stall.
- Using `gevent` for PDF workers: pdf2image calls subprocess (Ghostscript/Poppler); these are synchronous subprocess calls that block the entire gevent event loop thread.

**Estimated throughput loss:** With `prefork` and 4 workers per task for image processing: 48 docs/min per ECS task. With `gevent` and 16 concurrent greenlets per task: 192 docs/min per ECS task — a 4x throughput improvement for the same ECS resources, while staying within the same Gemini rate limit envelope (rate limiting is still necessary per Finding 1).

### Recommendation

After splitting the fleet per Finding 3:

- **Image workers**: Use `--pool=gevent` with `--concurrency=16` (or higher, test against actual Gemini latency). Gevent patches the `requests` and `httpx` libraries used for Gemini HTTP calls, enabling I/O multiplexing. Each ECS task effectively handles many concurrent in-flight Gemini requests.
- **XML workers**: Use `--pool=prefork` with `--concurrency=$(nproc)`. lxml's XSD validation and XPath extraction are CPU-bound; prefork gives true parallelism.
- **PDF workers**: Use `--pool=prefork` with `--concurrency=2`. The bottleneck is pdf2image (Poppler/Ghostscript subprocess); limiting concurrency prevents memory exhaustion from simultaneous page rasterization.

---

## Finding 7 — Database Connection Pooling and Write Contention

**Severity: Medium**
**Topic: Database Performance**

### Problem

Multiple ECS tasks, each running multiple Celery worker processes, all write to Supabase PostgreSQL directly. Celery workers are typically configured with a database connection per worker process. With:

- 10 ECS tasks × 4 prefork workers per task = 40 simultaneous connections (conservative scenario)
- 20 ECS tasks × 16 gevent workers per task = 320 logical connections (aggressive scaling scenario)

Supabase PostgreSQL on standard tiers supports 60–200 direct connections (varies by plan). PostgreSQL itself handles up to ~500 connections before connection overhead becomes a significant performance factor. Beyond that, connection establishment and maintenance consume memory and CPU that would otherwise serve queries.

**Write patterns:** All three worker branches write to the same `documents` table on task completion. During a large batch, this table receives concurrent INSERT and UPDATE statements from all workers simultaneously. If the status polling query (Finding 4) is running an aggregate scan against the same table at the same time, table-level lock contention is possible on older PostgreSQL versions, and even on modern versions, hot block contention on heavily-written pages degrades throughput.

**Estimated impact:** At 100 concurrent worker processes writing at 1 write/5 seconds each, the table receives 20 writes/second — well within PostgreSQL's capability. At 500 workers, 100 writes/second with concurrent reads creates measurable contention, particularly if vacuum is not keeping up with tuple churn from status updates.

### Recommendation

**Connection pooling:** Deploy PgBouncer in transaction pooling mode between the worker fleet and PostgreSQL. This multiplexes many application connections over a small number of actual PostgreSQL connections. With transaction pooling, each worker acquires a connection only for the duration of a single transaction, then releases it. 500 logical connections map to 20–50 actual PostgreSQL connections. Supabase provides a built-in PgBouncer endpoint; use it.

**Write design:** Separate the status update (frequent, small) from the data write (infrequent, large). Use two operations:
1. On task start: `UPDATE documents SET status='processing' WHERE id=$1` — small, fast
2. On task complete: `INSERT INTO extracted_data (...) VALUES (...); UPDATE documents SET status='completed' WHERE id=$1` — larger, but infrequent

**Partitioning:** If volume is expected to exceed 1M documents, partition the `documents` table by `created_at` (range partitioning by month) or by `batch_id` (hash partitioning). Status queries filtered by `batch_id` then hit a smaller partition.

**Avoid UPDATE churn for status:** Frequent status updates (`pending → processing → completed`) generate dead tuples that PostgreSQL must vacuum. Use `FILLFACTOR=70` on the `documents` table to reserve space for in-place updates (HOT updates), reducing dead tuple accumulation.

---

## Finding 8 — Queue Depth and Redis Memory Pressure

**Severity: Medium**
**Topic: Queue Depth and Backpressure**

### Problem

Celery by default stores task messages in Redis as serialized JSON or pickle blobs. Each message contains the task name, arguments (including file URLs and metadata), and routing information. For this system, each message is small (file URL + batch ID + document ID), perhaps 200–500 bytes.

At 1,000 queued documents, total Redis memory for task payloads is roughly 500 KB — negligible. At 1,000,000 queued documents, it is 500 MB — significant for ElastiCache instances.

The deeper issue is **backpressure**. There is no described mechanism to limit how fast the API publishes tasks to Redis relative to how fast workers consume them. If the API accepts a 100,000-document batch in 5 seconds (plausible for a bulk upload scenario) and workers process at 1,000 documents/minute, the queue will hold ~99,000 tasks for ~100 minutes. During this period, any Redis failover event (ElastiCache maintenance, node replacement) loses all queued tasks unless persistence is configured.

**Estimated memory for 100,000 tasks at 500 bytes each:** 50 MB. For a cache.t3.medium (3.09 GB), this is manageable. However, ElastiCache Redis by default is an in-memory store with no persistence. Task loss on node restart is a real risk.

### Recommendation

**Redis persistence:** Enable AOF (Append-Only File) persistence on ElastiCache with `appendfsync everysec`. This adds 1–2 ms write latency to Redis operations but protects queued tasks from loss on node restart. Without this, a Redis node replacement during a large batch silently loses all queued tasks with no error to the client.

**Queue length monitoring and backpressure:** Add a metric for `llen queue:image`, `llen queue:xml`, `llen queue:pdf` as CloudWatch custom metrics. Set an alarm at a configurable high-water mark (e.g., 10,000 tasks per queue). When the high-water mark is reached, the API should return HTTP 429 to new batch submissions rather than continuing to enqueue, giving operators visibility before queue depth becomes unmanageable.

**Soft rate limiting at ingestion:** The API should track in-flight + queued task count (using a Redis counter incremented on enqueue, decremented on worker completion) and reject submissions that would exceed a configured maximum. This creates backpressure at the correct point — the ingestion interface — rather than allowing the queue to grow without bound.

---

## Finding 9 — Cold Start and Serverless Availability

**Severity: Medium**
**Topic: Cold Start and Availability**

### Problem

The Hono API runs serverless on Vercel. Serverless functions have cold start latency when no instance is warm. For a TypeScript/Node.js function on Vercel, cold start is typically 200–800 ms. For Python (if FastAPI is chosen), cold start is 500 ms–2 seconds due to heavier runtime initialization.

**Cold start impact on batch submission:** If the API has not received a request in the last few minutes (e.g., first morning upload), the first request in a batch submission may experience a cold start delay. This is a single event per session, not per-request, so for large batches it is a one-time 200–800 ms penalty — acceptable.

**Cold start impact on status polling:** If the API scales to zero between polls (Vercel aggressively recycles idle instances), each poll may incur a cold start. With 2-second polling intervals and aggressive recycling, every poll could cold-start. This is the more dangerous scenario: a client polling every 2 seconds for 10 minutes makes 300 requests; if each cold-starts, total latency added is 300 × 500 ms = 150 seconds of wasted waiting.

**Worker cold start (ECS):** ECS task startup time includes container image pull (30–120 seconds for first pull; cached thereafter), container initialization, and Celery worker startup (5–15 seconds for Python + dependency import). If autoscaling triggers a new ECS task in response to queue depth, there is a 45–135 second gap before the new worker begins consuming tasks. During high-burst scenarios, this lag means the queue grows unchecked for 1–2 minutes after a scaling event triggers.

### Recommendation

**For Vercel cold starts:** Use Vercel's "fluid compute" or "always-on" instance options for the status polling endpoint specifically. The batch submission endpoint tolerates a single cold start; the polling endpoint does not. Alternatively, keep the polling interval at 5–10 seconds (rather than 2 seconds) to reduce instance recycling pressure.

**For ECS worker warm pool:** Configure an ECS capacity provider with a minimum running task count of at least 1 per queue type. This ensures at least one worker per branch is always running and available, eliminating cold starts for moderate load. Autoscaling adds tasks above this floor when queue depth grows.

**Pre-warm strategy for predictable load:** If document upload volume follows a daily pattern (business hours), use scheduled ECS scaling to pre-warm the worker fleet 10–15 minutes before expected peak load, avoiding the ECS startup latency window during burst onset.

---

## Finding 10 — Storage I/O for Large Files

**Severity: Medium**
**Topic: Storage I/O**

### Problem

Every worker downloads the full file from Supabase Object Storage before processing. The file is downloaded to the worker's local ephemeral filesystem or to memory. For large files, this is significant:

| File type | Typical size range | Download time (100 Mbps) |
|-----------|-------------------|--------------------------|
| XML (NFe) | 5–50 KB | < 10 ms |
| Image (thermal photo) | 500 KB–5 MB | 40–400 ms |
| PDF (DANFE) | 200 KB–20 MB | 16–1,600 ms |
| PDF (multi-page scanned) | 2–50 MB | 160 ms–4 s |

For image and PDF workers, S3 download time is a non-trivial fraction of total task time. For a 10 MB scanned PDF on a worker with 100 Mbps egress, download alone takes 800 ms before any processing begins.

**ECS egress costs:** AWS charges for data transfer out of ECS to external S3 (Supabase Object Storage is external to AWS). For high-volume processing, this cost compounds. At 1,000 documents/day averaging 2 MB each, monthly egress is ~60 GB — minor. At 100,000 documents/day, it is ~6 TB/month — significant cost.

**The Classifier's ranged GET** (if kept in the API path per current design) adds additional S3 read operations per file beyond the worker download. Each file is read twice: once for classification (header bytes) and once for processing (full file). At scale, this doubles the S3 API request count.

### Recommendation

**Co-locate workers with storage:** If feasible, run ECS workers in the same AWS region as Supabase's Object Storage backend (Supabase uses AWS S3 under the hood; the region is configurable). Same-region transfers avoid inter-region data transfer charges and reduce download latency to 10–50 ms for most file sizes.

**Stream processing instead of full download:** For the XML branch, replace full-file download with streaming parse: open an HTTP stream from S3, pipe it to lxml's `iterparse`. This reduces peak memory usage and eliminates the write-to-disk step. lxml supports streaming validation against XSD schemas.

**For PDF/image workers:** Download directly to a memory buffer (BytesIO) rather than writing to ephemeral disk, then pass the buffer to pdfplumber or Pillow. This eliminates one filesystem I/O round-trip per task. For files > 50 MB, disk-backed processing is still appropriate to avoid memory pressure.

**Eliminate the duplicate read:** If the Classifier is moved to the worker (per Finding 2's recommendation), each file is read only once — during worker processing. The API-path ranged GET is eliminated entirely.

---

## Finding 11 — Batch Size Effects

**Severity: Medium**
**Topic: Batch Size Effects**

### Problem

The system's behavior changes qualitatively at different batch sizes. Three regimes are relevant:

**Small batch (1–10 files):**
- API latency: Dominated by Classifier S3 I/O (if synchronous). For 1 file, this adds 20–60 ms, acceptable.
- Worker pickup: Depends on worker availability. If a worker is idle, task is consumed within 1–2 seconds. If workers are scaling up, 45–135 second ECS cold start.
- End-to-end latency: 5–30 seconds for image/PDF. < 2 seconds for XML.
- Verdict: Acceptable, but cold start behavior is jarring for interactive users who expect fast feedback.

**Medium batch (100 files):**
- API latency: If Classifier is synchronous, 3–8 seconds added per Finding 2. Potentially approaches Vercel timeout.
- Queue ingestion: All 100 tasks published to Redis in < 1 second (Redis publish is fast). Workers begin consuming immediately if warm.
- Processing time at rate limit ceiling: ~6–10 minutes for image/PDF heavy batch.
- Verdict: Works, but synchronous Classifier becomes a notable problem at this scale.

**Large batch (1,000 files):**
- API latency: Synchronous Classifier at 1,000 files exceeds any reasonable Vercel timeout. The API request fails before tasks are published.
- Queue size: 500 KB in Redis — fine.
- Processing time at rate limit ceiling (1,000 images): ~60 minutes minimum.
- Database polling: 100 concurrent users × 300 polls × 10 ms/poll = 300 seconds of DB time.
- ECS scaling: Multiple scale-out events; first event adds 45–135 second startup lag.
- Status response accuracy: With no atomic batch summary counter, the COUNT query becomes expensive as the documents table grows.
- Verdict: System likely fails at API layer due to Classifier timeout. Even if that is fixed, the 60-minute processing window creates significant UX and operational challenges.

**Extreme batch (10,000 files):**
- Without asynchronous Classifier: Not possible (API timeout).
- With async Classifier: API responds in < 100 ms.
- Processing time at rate limit ceiling: 600 minutes (~10 hours) for all-image batch.
- Redis memory: 5 MB for task payloads — fine.
- ECS cost: Multiple scale-out events, sustained for hours.
- Verdict: Architecturally possible with the fixes described, but requires explicit design for long-running batch jobs (progress reporting, partial completion handling, cost caps).

### Recommendation

Design the system explicitly for the batch size range it is expected to support. If batches > 1,000 files are in scope, the following must be addressed before launch:

1. Asynchronous Classifier (Finding 2) — required for > 100 files
2. Distributed Gemini rate limiter (Finding 1) — required for > 500 image/PDF files
3. Atomic batch summary counters (Finding 4) — required for > 100 files with polling
4. Redis AOF persistence (Finding 8) — required for multi-hour processing windows
5. SSE or webhook for completion notification — required for > 10-minute processing windows

If batches > 10,000 files are in scope, evaluate replacing Celery + Redis with AWS SQS + Lambda or AWS Batch, which have native scalability for very large task volumes without manual queue management.

---

## Summary Table

| # | Finding | Severity | Primary Impact |
|---|---------|----------|----------------|
| 1 | No Gemini rate limiter — uncontrolled 429 cascade | **Critical** | Effective throughput drops 50–70% under load |
| 2 | Synchronous Classifier in API request path | **High** | API timeouts at batch size > 100–500 files |
| 3 | Worker co-location prevents independent scaling | **High** | 5–10x cost overrun; XML throughput constrained to PDF rate |
| 4 | No batch completion detection; unbounded polling | **High** | Unnecessary DB load; no completion signal |
| 5 | PDF double-processing for scanned documents | **High** | 15–27 seconds per scanned PDF; compounded by rate limit |
| 6 | Celery pool type mismatch for I/O-bound image workers | **High** | 4x throughput loss for image branch vs optimal pool config |
| 7 | No connection pooling; write contention at scale | **Medium** | PostgreSQL connection exhaustion at > 100 workers |
| 8 | No Redis persistence; no backpressure at ingestion | **Medium** | Task loss on Redis restart; queue grows without bound |
| 9 | Serverless cold starts on polling endpoint | **Medium** | 500 ms latency per poll during aggressive instance recycling |
| 10 | Full-file download before processing; duplicate S3 reads | **Medium** | 800 ms–4 second download overhead for large PDFs |
| 11 | Batch size effects — system behavior changes qualitatively | **Medium** | API failure at > 100–500 files without async Classifier |

---

## Implementation Priority

The three findings that must be resolved before the system will function correctly at any non-trivial load are, in order:

1. **Gemini distributed rate limiter** — without this, scaling makes things worse.
2. **Asynchronous Classifier** — without this, batch submission fails above a threshold batch size.
3. **Worker fleet split by branch** — without this, co-location wastes resources and prevents tuning.

Everything else in this document is a scalability improvement on top of a system that will at least function. The Critical and High findings above represent architectural decisions that, if deferred, will require significant rework after the system is deployed and under load.
