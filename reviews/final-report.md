# Architecture Review: Final Consolidated Report
**Event-Driven Fiscal Document Processing System (Brazilian SEFAZ)**
**Review Date:** 2026-03-19 | **Review Type:** Pre-Implementation Architecture Review

---

## 1. Executive Summary

The system architecture is conceptually sound but is not ready for implementation. Two critical documentation contradictions (the diagram contradicts the written spec) and eleven security/correctness blockers must be resolved before writing production code. The architecture has no authentication design, no state machine, no API contract, and no CI/CD pipeline — all of which are pre-implementation requirements, not post-implementation improvements. The primary recommendation is to replace Hono with FastAPI before any code is written, which is zero-cost at this stage and eliminates an entire class of fragility that would otherwise follow the project permanently.

---

## 2. The Hono vs FastAPI Decision

**Verdict: Replace Hono with FastAPI. Do it now, before any code exists.**

This is the lowest-cost moment to make this change. The recommendation is unanimous across all review phases and is driven by structural incompatibility, not stylistic preference.

### Why Hono fails here

**Celery integration is the core problem.** Celery is a Python task queue. Publishing tasks from TypeScript requires using a third-party, community-maintained library (such as `node-celery-ts`) with no official support, no Celery version guarantees, and no parity with Python's `apply_async` semantics. Any breaking change in Celery's internal wire format (Redis key layout, task envelope structure, header fields) silently breaks TypeScript publishing with no compile-time signal. This is not a theoretical risk — it is the default outcome of crossing language boundaries on an undocumented internal protocol.

**Schema drift between API and workers.** When the API publishes a task payload in TypeScript and the worker receives it in Python, the types are defined in two places in two languages. Every field addition, rename, or type change requires two coordinated edits with no enforcement mechanism. Pydantic models shared across the Python stack eliminate this class of bug entirely.

**The Classifier runs better in Python.** Magic byte inspection is the native domain of `python-magic` (libmagic binding). Replicating this in TypeScript adds a dependency that is less maintained and less complete.

**Vercel is wrong for this workload regardless.** The Classifier performing synchronous S3 I/O in the API request path on a serverless platform with cold starts and a 10-second function timeout will fail on batches larger than ~500 files. This constraint exists whether the API is Hono or FastAPI, but migrating to FastAPI on a long-running server (EC2, ECS, or similar) resolves the constraint cleanly.

### Why FastAPI is the right replacement

| Concern | FastAPI outcome |
|---|---|
| Celery task publishing | Native — `celery.send_task()` is the documented path |
| Shared task schemas | Single Pydantic model, imported by API and worker |
| Classifier | `python-magic` directly, no translation layer |
| Language count | One language for the entire backend |
| Migration cost | Zero — nothing is implemented |
| Async support | Native `async def` with uvicorn |
| API contract | FastAPI auto-generates OpenAPI/JSON Schema |
| Auth integration | OAuth2/JWT helpers built in |

### What to retain from the Hono design

The routing structure, the classification-before-publish flow, and the three-branch worker dispatch model are all correct. Only the implementation language changes.

---

## 3. Findings by Priority

### P0 — Critical (must fix before any code is written)

**Architecture & Documentation**

**C-01** — Diagram contradicts written spec. The architecture diagram and the written documentation describe different systems: different hosting targets (Railway vs AWS ECS), different components (Consumer not in spec), different storage (AWS S3 vs Supabase Storage). All implementation decisions are blocked until these are reconciled into a single authoritative reference.

**C-02** — No API contract exists. There is no OpenAPI spec, no request/response schema, and no defined interface between the Next.js frontend and the API layer. Frontend and backend cannot be developed in parallel without this.

**C-03** — No state machine for batch/document lifecycle. No defined states, no defined transitions, no terminal states. Workers cannot be written without this.

**C-04** — No database schema. No table definition for batches, documents, extracted data, or errors. The system has no schema to write to.

**Security**

**C-05** — No authentication design. The API has no authentication mechanism defined. Any caller can submit batches, read results, or trigger processing. Unacceptable for a system handling regulated fiscal data (CNPJ/CPF under LGPD).

**C-06** — No tenant isolation design. Without it, one tenant can read or trigger processing of another tenant's documents.

**C-07** — SSRF vulnerability in Classifier. The Classifier receives file URLs and fetches them. An attacker can submit `http://169.254.169.254/` (AWS metadata service) or internal VPC addresses. Must validate URL scheme, resolve hostname, and block RFC-1918/link-local ranges before any fetch.

**C-08** — No LGPD data protection strategy. The system stores CPF/CNPJ regulated under Brazil's LGPD. No data classification, retention policy, encryption-at-rest requirement, or right-to-erasure design is documented.

**C-09** — Celery serializer not specified as JSON. The default Celery serializer is pickle. Deserializing pickle from Redis is an unauthenticated remote code execution vector. `task_serializer = "json"` and `accept_content = ["json"]` must be set explicitly.

**C-10** — lxml security flags not specified. XML processing without explicit security flags is vulnerable to XXE and XML bomb attacks. Required flags: `resolve_entities=False`, `no_network=True`, `huge_tree=False`, `load_dtd=False`.

**Framework & CI/CD**

**C-11** — No CI/CD pipeline. For a multi-component system with deployment ordering requirements (migrations before workers), CI/CD is a correctness requirement.

**C-12** — No secrets management design. Database credentials, Redis auth tokens, Gemini API key, and storage credentials have no defined management strategy.

**C-13** — No Gemini client-side rate limiter. Without a token bucket rate limiter shared across workers, scaling workers fills the DLQ instead of increasing throughput. More workers → worse outcomes.

---

### P1 — High (must be resolved before the component is built)

**Architecture**

**H-01** — Retry policy undefined. No retry count, no backoff strategy, no definition of retryable vs terminal errors. The DLQ worker cannot be built without this.

**H-02** — PDF confidence threshold undefined. Metric, threshold value, and measurement method are all undefined.

**H-03** — Classifier S3 I/O is synchronous in the API request path. Will fail on batches larger than ~500 files on Vercel. Must either move to a worker, remove the download (accept metadata from frontend), or run on a long-lived server.

**H-04** — Supabase Object Storage vs AWS S3 ambiguity. Written spec says Supabase; diagram shows AWS S3. These have different access control models and SDK configurations. One must be chosen.

**H-05** — No independent worker scaling design. Three branches with very different throughput profiles (XML is fast, image is slow+expensive) must be separate services with separate queues and autoscaling policies.

**H-06** — No batch completion signal. No defined mechanism for the system to signal that all documents in a batch have been processed.

**H-07** — Scanned PDF double-processing risk. PDFs re-routed from the PDF branch to the image branch may be re-processed from the PDF branch on retry if the in-progress state is not tracked correctly, creating duplicate DB records.

**Security**

**H-08** — No RBAC design. No role definitions (admin, submitter, reader) and no permission model.

**H-09** — No file size or file count limits. No defined maximum per-file size or maximum batch document count. Required to prevent resource exhaustion.

**H-10** — Prompt injection via document content. Document text passed to Gemini can contain instructions that override the extraction prompt. Use structured output mode; do not interpolate document content into instruction text.

**H-11** — Gemini response not validated before DB write. Unexpected Gemini response shapes must be caught by Pydantic validation before writing to the database.

**H-12** — Redis has no authentication or network isolation. Must require AUTH and must not be accessible outside the VPC.

**Framework**

**H-13** — Wrong Celery pool backend for I/O-bound workers. Use `gevent` pool for image workers (I/O-bound, waiting on Gemini). Use `prefork` for PDF workers (CPU-bound, pdfplumber + pdf2image). Mixing them destroys throughput for both.

**H-14** — Celery `acks_late` and `reject_on_worker_lost` not specified. Without these, an ECS task killed mid-execution silently drops the document with no retry and no DLQ entry.

**H-15** — FastAPI connection pools must use lifespan context. All connection pool initialization and teardown must use FastAPI's `lifespan` context manager.

**H-16** — `pdfplumber` must be used as a context manager. Failing to do so leaks file handles and exhausts OS file descriptor limits under load.

**H-17** — `pdf2image` must write to disk, not memory. Specify `output_folder` in `convert_from_path()`. In-memory conversion of multi-page PDFs causes OOM kills on worker containers.

**H-18** — Use `google-genai` SDK, not deprecated `google-generativeai`. The older library is deprecated.

**CI/CD**

**H-19** — No container security scanning in CI. Images must be scanned with `trivy` (fail on CRITICAL CVEs) before deployment.

**H-20** — No database migration strategy. Schema changes must be managed through Alembic with expand-contract migration pattern. No ad-hoc schema changes on a database containing fiscal records.

**H-21** — No local development environment. A `docker-compose` setup bringing up Redis, PostgreSQL, and all worker types is required for development and integration testing.

**H-22** — No queue-based autoscaling. ECS worker scaling must be driven by queue depth (custom CloudWatch metrics), not CPU/memory.

**H-23** — No observability pipeline. `structlog` for structured JSON logging, OpenTelemetry for metrics, Grafana Cloud as backend. Key metrics: DLQ depth, Gemini 429 rate, task duration histograms per worker type.

---

### P2 — Medium (resolve before production-ready)

**M-01** — No ADRs for any technology decision. The Hono→FastAPI decision, Supabase vs S3 choice, Celery pool strategy — all should be recorded in `07. Decisions/`.

**M-02** — No runbook for DLQ manual review. No documented process for reviewing, resubmitting, or permanently failing DLQ entries.

**M-03** — No domain onboarding context. Developers without Brazilian fiscal domain knowledge cannot understand why the system is designed as it is (XPath for XML, contrast enhancement for thermal paper, SEFAZ schema).

**M-04** — No input validation limits documented. File type allowlist, max file size, max batch size not documented.

**M-05** — Polling vs SSE uncommitted. SSE requires persistent connections incompatible with serverless. One must be chosen; Supabase Realtime is a natural fit given Supabase is already in the stack.

**M-06** — No storage access control design. No defined policy for who can read stored files, presigned URL strategy, or URL expiry policy.

---

### P3 — Low (before v1 release)

**L-01** — "Resize to 400KB" is ambiguous. Specify whether this is a hard maximum file size, a target, or a pixel dimension guideline. Include encoding format and quality level.

**L-02** — Use `uv` for Python dependency management. `pip` + `requirements.txt` produces non-reproducible builds. `uv` with a lockfile ensures deterministic environments across CI, containers, and developer machines.

---

## 4. Findings by Category

| Category | P0 Critical | P1 High | P2 Medium | P3 Low | Total |
|---|---|---|---|---|---|
| Architecture / Documentation | 4 | 4 | 4 | 1 | 13 |
| Security | 4 | 5 | 1 | 0 | 10 |
| Framework / Implementation | 2 | 6 | 0 | 1 | 9 |
| CI/CD / Operations | 2 | 5 | 1 | 0 | 8 |
| Performance / Scaling | 1 | 4 | 0 | 0 | 5 |
| **Total** | **13** | **24** | **6** | **2** | **45** |

---

## 5. Pre-Implementation Blockers

These 13 items must be resolved before writing any production code:

1. Reconcile diagram and written spec (C-01)
2. Choose Hono vs FastAPI — this report recommends FastAPI (C-13 context)
3. Choose Supabase Object Storage vs AWS S3 (H-04)
4. Define authentication mechanism (C-05)
5. Define tenant isolation model (C-06)
6. Define document/batch state machine (C-03)
7. Define database schema (C-04)
8. Define API contract / OpenAPI spec (C-02)
9. Define retry policy (H-01)
10. Define PDF confidence threshold (H-02)
11. Define SSRF mitigation strategy (C-07)
12. Define LGPD data protection strategy (C-08)
13. Set up secrets management before any credentials are generated (C-12)

---

## 6. Recommended Implementation Order

**Phase 0 — Foundations (before any feature code)**
1. Resolve all 13 pre-implementation blockers
2. Set up CI pipeline (ruff, mypy, trivy, pip-audit, real Redis+PostgreSQL in CI)
3. Set up secrets management
4. Write `docker-compose` local development environment
5. Write initial database schema and Alembic migration baseline

**Phase 1 — Core infrastructure**
6. FastAPI scaffold with lifespan context, connection pools, OpenAPI spec
7. Authentication middleware — before any endpoint is exposed
8. Celery scaffold: `task_serializer="json"`, `acks_late=True`, `reject_on_worker_lost=True`
9. Separate Celery worker containers (image/IO + PDF/CPU + XML)
10. Shared Pydantic task schemas in a common `invoice-models` package

**Phase 2 — Ingestion layer**
11. Batch submission endpoint with file validation and SSRF-safe URL handling
12. Classifier: python-magic, no S3 re-download, typed task payload output
13. Task publication to per-branch Redis queues
14. Document state write on submission

**Phase 3 — Worker branches (parallelizable after Phase 2)**
15. XML worker: lxml with all 4 security flags, XSD validation, XPath extraction
16. PDF worker: pdfplumber context manager, confidence threshold routing, pdf2image disk output
17. Image worker: download, resize, contrast, Gemini call with rate limiter, Pydantic response validation
18. DLQ worker: state update to failed, error reason persistence, runbook

**Phase 4 — Frontend and observability**
19. Next.js frontend: file upload to storage, status polling (mechanism decided in blockers)
20. Batch completion signal implementation
21. Structured logging, metrics emission, alerting rules
22. Queue-depth autoscaling for ECS worker services

**Phase 5 — Hardening before v1**
23. Security test surface: SSRF, XXE, prompt injection, tenant isolation
24. LGPD compliance: retention jobs, erasure endpoint, encryption-at-rest verification
25. Load test: Gemini rate limiter under horizontal worker scale
26. Container CVE scan integration in CD pipeline

---

## 7. Review Metadata

| Field | Value |
|---|---|
| Review date | 2026-03-19 |
| System reviewed | Event-driven SEFAZ fiscal document processor |
| Review type | Pre-implementation architecture review |
| Phases completed | 4 of 4 |
| Total unique findings | 45 |
| P0 Critical | 13 |
| P1 High | 24 |
| P2 Medium | 6 |
| P3 Low | 2 |
| Primary recommendation | Replace Hono with FastAPI before implementation begins |
| Implementation readiness | **Not ready** — 13 blockers must be resolved first |

---

## Supporting Documents

| Document | Location |
|---|---|
| Security audit (full) | `02. Security/architecture-security-audit.md` |
| Performance analysis (full) | `01. Architecture/performance-scalability-analysis.md` |
| Framework & library review | `01. Architecture/framework-library-review.md` |
| Testing strategy | `03. Testing/testing-strategy.md` |
| CI/CD & DevOps strategy | `04. CI-CD/cicd-devops-strategy.md` |
