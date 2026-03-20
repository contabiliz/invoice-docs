# Framework & Library Review — Fiscal Document Processing System

**Date**: 2026-03-19
**Scope**: Python backend (FastAPI API layer + Celery worker fleet)
**Status**: Pre-implementation review

---

## Summary Table

| # | Area | Severity | Finding |
|---|------|----------|---------|
| 1 | FastAPI lifespan & connection pooling | High | Missing lifespan pattern; pools must be created once per process |
| 2 | Celery serializer | Critical | Must lock to JSON globally; pickle is the compiled default |
| 3 | Celery `acks_late` + `reject_on_worker_lost` | High | Both must be set; without them a worker crash silently drops tasks |
| 4 | Celery pool backends per branch | High | Gevent for image/XML (I/O); prefork for PDF (CPU) via separate queues |
| 5 | Celery result backend | Medium | Disable or use dedicated Redis DB; do not share with broker DB |
| 6 | Celery task naming | Medium | Always set explicit `name=` or tasks break on refactor |
| 7 | pdfplumber memory leak | High | Must be used as a context manager; pages are lazy and hold file handles |
| 8 | pdf2image / poppler in Docker | High | poppler-utils must be installed in the image; use multi-stage build |
| 9 | python-magic system dependency | Medium | Requires `libmagic1` in container; `python-magic-bin` is a wheel-only alternative |
| 10 | lxml XXE hardening | Critical | Already identified; confirmed correct parser flags required |
| 11 | Pillow security | Medium | Pin to ≥10.3.0; avoid `Image.open` on untrusted streams without size cap |
| 12 | Dependency manager | High | Use `uv`; fastest resolver, lock-file by default, Docker-friendly |
| 13 | Code organisation | High | Single monorepo with a `packages/shared` library for Pydantic models |
| 14 | Redis client | Medium | `redis-py` ≥5 for API (async support built-in); `redis-py` sync for workers |
| 15 | PostgreSQL access in workers | Medium | `psycopg3` (psycopg) directly; SQLAlchemy adds unnecessary overhead |
| 16 | Gemini SDK | High | Use `google-genai` (new unified SDK); avoid deprecated `google-generativeai` |
| 17 | Docker base image | Medium | `python:3.12-slim-bookworm`; distroless is impractical with poppler |
| 18 | Python version | Low | 3.12 is correct; key benefit is `TaskGroup` and faster free-threaded prototype |

---

## Detailed Findings

---

### 1. FastAPI — Lifespan Events and Connection Pooling

**Severity**: High

**Current/planned**: Unspecified; risk of creating DB/Redis connections per request.

**Recommended**: Use the `lifespan` context manager (introduced in Starlette 0.20, the default in FastAPI ≥0.93) to create shared connection pools once per process and attach them to `app.state`. Dependency injection functions then read from `app.state` rather than opening new connections.

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
import redis.asyncio as aioredis
import asyncpg

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.redis = aioredis.ConnectionPool.from_url(
        settings.redis_url, max_connections=20
    )
    app.state.pg_pool = await asyncpg.create_pool(
        settings.database_url, min_size=2, max_size=10
    )
    yield
    # Shutdown
    await app.state.redis.aclose()
    await app.state.pg_pool.close()

app = FastAPI(lifespan=lifespan)
```

```python
# app/dependencies.py
from fastapi import Request
import redis.asyncio as aioredis

async def get_redis(request: Request) -> aioredis.Redis:
    return aioredis.Redis(connection_pool=request.app.state.redis_pool)

async def get_db(request: Request) -> asyncpg.Connection:
    async with request.app.state.pg_pool.acquire() as conn:
        yield conn
```

**Additional FastAPI patterns**:

- All endpoints that touch Redis or PostgreSQL must be `async def`. Use `def` only for CPU-bound pure Python work (none here).
- Use `Annotated` dependency injection (Pydantic v2 style) rather than `Depends` as a default argument to keep function signatures clean.
- Split routers into `app/routers/batches.py`, `app/routers/documents.py`. Keep `main.py` as the composition root only.
- Return `Response` with `media_type="application/json"` for 204/202 responses to avoid FastAPI serialising `None`.

---

### 2. Celery Serializer — CRITICAL

**Severity**: Critical

**Current/planned**: Not yet specified.

**Recommended**: Lock the serializer to JSON globally, both on the producer (API) and consumer (worker) side. Pickle is the historical default in older Celery versions and is still present in some configurations.

```python
# celery_app.py (shared config, used by both API and workers)
from celery import Celery

celery_app = Celery("invoice")

celery_app.conf.update(
    broker_url=settings.redis_url,
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],          # reject anything not JSON
    task_always_eager=False,
    task_acks_late=True,              # see finding #3
    task_reject_on_worker_lost=True,  # see finding #3
    worker_prefetch_multiplier=1,     # one task at a time per worker slot
)
```

Explicitly setting `accept_content=["json"]` means the worker will reject any message serialised in another format, preventing deserialization gadget attacks if the Redis broker is compromised.

---

### 3. Celery `acks_late` and `task_reject_on_worker_lost`

**Severity**: High

**Current/planned**: Identified as a requirement in prior phase; confirm implementation.

**Recommended**: Both settings must be enabled. Without `acks_late=True`, Celery acknowledges the task when it is *delivered* to the worker, not when it completes. If the ECS task is killed mid-execution (OOM, spot interruption, deployment), the document is silently lost — no retry, no DLQ entry.

With `task_reject_on_worker_lost=True`, a worker crash causes the broker to redeliver the task rather than losing it. Combined with `acks_late`, this provides at-least-once delivery semantics.

Consequence for idempotency: tasks must be idempotent because they can be retried after partial execution. Database writes should use `INSERT ... ON CONFLICT DO UPDATE` (upsert), not bare `INSERT`.

```python
# tasks/base.py
from celery import Task

class IdempotentTask(Task):
    abstract = True
    acks_late = True
    reject_on_worker_lost = True
    max_retries = 3
    default_retry_delay = 30  # seconds

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        # Write failed status to DB; DLQ worker handles the task record
        pass
```

---

### 4. Celery Pool Backends per Branch

**Severity**: High

**Current/planned**: Identified as a requirement; specific configuration not yet set.

**Recommended**: Deploy two separate worker containers, each consuming a dedicated queue:

| Container | Queue | Pool | Concurrency |
|-----------|-------|------|-------------|
| `worker-io` | `image`, `xml` | `gevent` | 50–200 |
| `worker-cpu` | `pdf` | `prefork` | nproc (e.g., 4) |

```bash
# worker-io container command
celery -A invoice.celery_app worker \
  --queues=image,xml \
  --pool=gevent \
  --concurrency=100 \
  --loglevel=info

# worker-cpu container command
celery -A invoice.celery_app worker \
  --queues=pdf \
  --pool=prefork \
  --concurrency=4 \
  --loglevel=info
```

The `gevent` pool patches the standard library's blocking I/O, making Gemini API calls and S3 downloads non-blocking within a single OS thread. The `prefork` pool gives each PDF worker its own process with its own GIL, allowing genuine parallelism for pdfplumber and pdf2image (both CPU-bound and memory-hungry).

Do not mix pools in one worker. If a gevent worker calls a CPU-bound function (pdfplumber), it blocks the entire event loop for all concurrent tasks in that worker process.

---

### 5. Celery Result Backend

**Severity**: Medium

**Current/planned**: Not specified.

**Recommended**: Disable the result backend entirely, or use a separate Redis database index.

The system already writes all status information to PostgreSQL (the system of record). The Celery result backend stores task return values in Redis, duplicating state and consuming memory for results that are never polled.

```python
celery_app.conf.update(
    result_backend=None,           # disable entirely if results are unused
    # OR use a separate DB to avoid interfering with the broker:
    # result_backend="redis://host/1",  # broker is on DB 0
    task_ignore_result=True,       # per-task override
)
```

If the API needs to check task status (beyond what is in PostgreSQL), use the PostgreSQL status column directly — do not rely on the Celery result backend for user-facing status queries.

---

### 6. Celery Task Naming

**Severity**: Medium

**Current/planned**: Default (module path derived).

**Recommended**: Always set an explicit `name` parameter. Default auto-naming uses the full Python module path. If the package is ever renamed or restructured, all tasks in the broker queue become unroutable — workers will reject messages that reference the old module path.

```python
@celery_app.task(
    base=IdempotentTask,
    name="invoice.tasks.process_image",   # stable, explicit name
    queue="image",
)
def process_image(document_id: str, object_key: str) -> None:
    ...
```

Establish a naming convention: `{service}.tasks.{branch}_{action}`, e.g. `invoice.tasks.image_process`, `invoice.tasks.xml_validate`, `invoice.tasks.pdf_extract`.

---

### 7. pdfplumber Memory Leak

**Severity**: High

**Current/planned**: pdfplumber for PDF text extraction (correct library choice).

**Recommended**: pdfplumber must always be used as a context manager. The `PDF` object holds an open file handle and keeps all page objects in memory. Failing to call `.close()` (either explicitly or via `with`) causes file descriptor leaks that accumulate across tasks within a prefork worker process.

```python
# WRONG — leaks file handles
pdf = pdfplumber.open(local_path)
text = pdf.pages[0].extract_text()

# CORRECT
with pdfplumber.open(local_path) as pdf:
    pages_text = [page.extract_text() or "" for page in pdf.pages]
confidence = compute_confidence(pages_text)
```

**Known gotcha**: `extract_text()` returns `None` (not an empty string) if no text layer is found. Always guard with `or ""`.

**Known gotcha**: pdfplumber uses `pdfminer.six` internally. Encrypted or corrupted PDFs raise `pdfminer.pdfdocument.PDFPasswordIncorrect` or `pdfminer.pdfparser.PDFSyntaxError`. Catch both explicitly in the task and write to the failed status rather than letting the exception propagate to Celery's retry machinery.

```python
from pdfminer.pdfdocument import PDFPasswordIncorrect
from pdfminer.pdfparser import PDFSyntaxError

try:
    with pdfplumber.open(local_path) as pdf:
        ...
except (PDFPasswordIncorrect, PDFSyntaxError) as exc:
    mark_document_failed(document_id, reason=str(exc))
    return  # do not raise; no retry needed for corrupt input
```

pdfplumber is the correct choice over PyMuPDF (fitz) for this use case: it has a cleaner API for text-layer confidence scoring and its AGPL licence is avoided. PyMuPDF's AGPL licence is a concern for a commercial system.

---

### 8. pdf2image / poppler in Docker

**Severity**: High

**Current/planned**: pdf2image for PDF-to-image rasterisation; poppler dependency unspecified.

**Recommended**: pdf2image is a thin wrapper around the `pdftoppm` binary from poppler-utils. The binary must be present in the container. This is the most common source of `PDFInfoNotInstalledError` in production.

```dockerfile
# worker-cpu/Dockerfile  (multi-stage)
FROM python:3.12-slim-bookworm AS base

RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    libmagic1 \
  && rm -rf /var/lib/apt/lists/*

FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM base AS runtime
WORKDIR /app
COPY --from=deps /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
USER nobody
```

**Known gotcha**: `convert_from_path` holds all rasterized PIL Image objects in memory simultaneously by default. For multi-page PDFs this can exhaust memory in prefork workers. Use the `output_folder` parameter to write pages to disk and load them one at a time.

```python
import tempfile
from pdf2image import convert_from_path

with tempfile.TemporaryDirectory() as tmp:
    pages = convert_from_path(
        local_path,
        dpi=200,
        output_folder=tmp,
        fmt="jpeg",
        thread_count=1,     # prefork already provides parallelism
    )
    for page_image in pages:
        route_to_image_pipeline(page_image)
```

---

### 9. python-magic System Dependency

**Severity**: Medium

**Current/planned**: `python-magic` for magic byte inspection.

**Recommended**: `python-magic` requires `libmagic1` (the C library) to be installed in the container. This is straightforward but must be explicit in the Dockerfile (see finding #8 above). Ensure the package installed is `python-magic`, not `filemagic` (a different, unmaintained package with the same import name that appears in search results).

An alternative is `python-magic-bin`, which bundles the libmagic binary as a Python wheel — convenient for local development on Windows/macOS but adds ~2 MB to the container image unnecessarily when the system library is already available.

For the API container (where the Classifier runs), the same `libmagic1` dependency applies — do not overlook it in the API Dockerfile.

```python
# classifier.py
import magic

def classify(data: bytes) -> Literal["xml", "pdf", "img"]:
    # Pass raw bytes, not a file path, to avoid SSRF (prior finding #1)
    mime = magic.from_buffer(data, mime=True)
    if mime == "application/xml" or mime == "text/xml":
        return "xml"
    if mime == "application/pdf":
        return "pdf"
    return "img"
```

Use `magic.from_buffer(data, mime=True)` rather than `magic.from_file(path)` in the Classifier. The classifier should receive the first 1–4 KB of the file content (enough for magic byte detection), not a file path, to keep the SSRF boundary clean.

---

### 10. lxml XXE Hardening — CRITICAL (Confirmed)

**Severity**: Critical

**Current/planned**: Identified in prior phase.

**Recommended**: Already correctly identified. Confirming the required flags and adding the defusedxml alternative consideration.

```python
from lxml import etree

safe_parser = etree.XMLParser(
    resolve_entities=False,
    no_network=True,
    huge_tree=False,    # prevent billion-laughs / quadratic blowup
    load_dtd=False,
)

tree = etree.parse(xml_file_path, safe_parser)
```

`huge_tree=False` (the default) prevents memory exhaustion from deeply nested or extremely large XML documents. `load_dtd=False` should also be set explicitly — without it, `resolve_entities=False` still allows DTD loading which can trigger network requests in some libxml2 versions.

**Alternative**: `defusedxml` wraps lxml/stdlib with safe defaults. However, defusedxml does not support XSD validation or XPath with namespaces as cleanly as lxml, so using lxml directly with explicit safe parser flags is correct for this use case.

---

### 11. Pillow Security and Size Limits

**Severity**: Medium

**Current/planned**: Pillow for image decode, resize, contrast enhancement.

**Recommended**: Pin Pillow to ≥10.3.0. Earlier versions have known decompression bomb vulnerabilities. The `PIL.Image.MAX_IMAGE_PIXELS` default (178 million pixels) should be verified as appropriate; reduce it if thermal paper receipts do not need to exceed a lower threshold.

```python
from PIL import Image

# Set before any Image.open calls in the worker process
Image.MAX_IMAGE_PIXELS = 50_000_000  # ~7000x7000 px; sufficient for receipts

def preprocess_image(raw_bytes: bytes) -> bytes:
    with Image.open(io.BytesIO(raw_bytes)) as img:
        img = img.convert("L")                    # greyscale (thermal paper)
        img = img.resize(target_size, Image.LANCZOS)
        img = ImageOps.autocontrast(img, cutoff=2)
        buf = io.BytesIO()
        img.save(buf, format="JPEG", quality=85)
        return buf.getvalue()
```

Always open images from `io.BytesIO(bytes)` rather than from a file path in the worker — the file has already been downloaded to memory from S3; avoid writing it to disk just to re-read it.

---

### 12. Dependency Manager

**Severity**: High

**Current/planned**: Not yet specified (pip / poetry / uv).

**Recommended**: Use `uv` (Astral). It is the current best-in-class Python package manager for new projects as of 2025–2026:

- Resolver is 10–100x faster than pip/poetry.
- Generates a `uv.lock` lockfile (cross-platform, deterministic).
- `uv sync --frozen` in Docker produces reproducible builds without network re-resolution.
- Supports PEP 517/518/621 (`pyproject.toml`) natively.
- Manages Python version itself (`uv python install 3.12`).
- `uv run` can replace `python -m` in entrypoints.

```toml
# pyproject.toml (API package example)
[project]
name = "invoice-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "pydantic>=2.7",
    "redis>=5.0",
    "asyncpg>=0.29",
    "python-magic>=0.4.27",
    "celery>=5.4",
    "invoice-shared",   # local path dependency
]

[tool.uv.sources]
invoice-shared = { path = "../shared", editable = true }

[dependency-groups]
dev = ["pytest-asyncio", "httpx", "ruff", "mypy"]
```

```dockerfile
# Docker layer caching with uv
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev --no-editable
```

**Do not use Poetry** for this project. Poetry's resolver is significantly slower, its virtual environment handling adds complexity in Docker, and it has known dependency resolution bugs with conflicting extras. pip with `requirements.txt` is acceptable but loses the lock file ergonomics and workspace (monorepo) support that uv provides natively.

---

### 13. Code Organisation — Monorepo with Shared Package

**Severity**: High

**Current/planned**: Not yet specified.

**Recommended**: A single Git repository with a `packages/` workspace containing three Python packages:

```
invoice-docs/               # this repo (docs only)

invoice-backend/            # new implementation repo
├── pyproject.toml          # uv workspace root
├── uv.lock                 # single lock file for all packages
├── packages/
│   ├── shared/
│   │   ├── pyproject.toml
│   │   └── invoice_shared/
│   │       ├── models/
│   │       │   ├── batch.py        # BatchCreate, BatchStatus (Pydantic)
│   │       │   ├── document.py     # DocumentStatus, ClassifierTag
│   │       │   └── task_payload.py # ImageTask, XmlTask, PdfTask
│   │       ├── celery_app.py       # shared Celery instance + config
│   │       └── settings.py         # pydantic-settings BaseSettings
│   ├── api/
│   │   ├── pyproject.toml          # depends on invoice-shared
│   │   └── invoice_api/
│   │       ├── main.py
│   │       ├── routers/
│   │       └── classifier.py
│   └── workers/
│       ├── pyproject.toml          # depends on invoice-shared
│       └── invoice_workers/
│           ├── image.py
│           ├── xml_branch.py
│           └── pdf_branch.py
├── docker/
│   ├── api.Dockerfile
│   ├── worker-io.Dockerfile
│   └── worker-cpu.Dockerfile
```

The `uv` workspace (`[tool.uv.workspace]` in the root `pyproject.toml`) allows a single `uv.lock` to span all three packages while keeping their `pyproject.toml` files independent. Each Docker image installs only its own package and its dependencies, not the entire workspace.

```toml
# invoice-backend/pyproject.toml (workspace root)
[tool.uv.workspace]
members = ["packages/*"]
```

**Pydantic v2 model patterns for the shared package**:

```python
# packages/shared/invoice_shared/models/task_payload.py
from pydantic import BaseModel, field_validator
from typing import Literal

class BaseTaskPayload(BaseModel):
    model_config = {"frozen": True}   # immutable after creation

    document_id: str
    batch_id: str
    object_key: str                   # S3 key only, never a URL (SSRF boundary)

class ImageTaskPayload(BaseTaskPayload):
    task_type: Literal["image"] = "image"

class XmlTaskPayload(BaseTaskPayload):
    task_type: Literal["xml"] = "xml"

class PdfTaskPayload(BaseTaskPayload):
    task_type: Literal["pdf"] = "pdf"

# Discriminated union for routing
TaskPayload = ImageTaskPayload | XmlTaskPayload | PdfTaskPayload
```

Using `model_config = {"frozen": True}` makes payloads hashable and prevents accidental mutation after deserialisation. The `Literal` discriminator allows Pydantic to select the correct model from the union without a custom validator.

---

### 14. Redis Client

**Severity**: Medium

**Current/planned**: Not yet specified.

**Recommended**: Use `redis-py` ≥5.0 for both API and workers. `aioredis` is now merged into `redis-py` and is no longer a separate package — the `redis.asyncio` namespace is the async interface.

```python
# API layer (async)
import redis.asyncio as aioredis

pool = aioredis.ConnectionPool.from_url(redis_url, max_connections=20)
client = aioredis.Redis(connection_pool=pool)

# Workers (sync — Celery is sync by default)
import redis

pool = redis.ConnectionPool.from_url(redis_url, max_connections=5)
client = redis.Redis(connection_pool=pool)
```

For the token bucket rate limiter protecting Gemini API calls (prior finding #6), use `redis-py`'s built-in Lua script support to make the check-and-decrement atomic:

```python
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)
if current == 1 then redis.call('EXPIRE', key, window) end
if current > limit then return 0 end
return 1
"""

rate_limit = client.register_script(RATE_LIMIT_SCRIPT)
allowed = rate_limit(keys=["gemini:ratelimit"], args=[15, 60])  # 15 req/min
```

---

### 15. PostgreSQL Access in Workers

**Severity**: Medium

**Current/planned**: Not yet specified (Supabase client vs direct psycopg).

**Recommended**: Use `psycopg` (psycopg3) directly in workers. Do not use the Supabase Python client for worker DB writes — it is a REST client that routes through PostgREST, adding an unnecessary HTTP hop and reducing throughput for high-volume writes.

Do not use SQLAlchemy for the workers. The ORM overhead is not justified for the simple upsert patterns workers need.

```python
# packages/workers/invoice_workers/db.py
import psycopg
from psycopg.rows import dict_row

def get_connection(database_url: str) -> psycopg.Connection:
    return psycopg.connect(database_url, row_factory=dict_row)

def upsert_document_result(conn: psycopg.Connection, document_id: str, data: dict) -> None:
    with conn.cursor() as cur:
        cur.execute("""
            INSERT INTO documents (id, status, extracted_data, updated_at)
            VALUES (%s, %s, %s, NOW())
            ON CONFLICT (id) DO UPDATE SET
                status = EXCLUDED.status,
                extracted_data = EXCLUDED.extracted_data,
                updated_at = EXCLUDED.updated_at
        """, (document_id, data["status"], psycopg.types.json.Jsonb(data["payload"])))
        conn.commit()
```

Workers are Celery tasks running in a prefork or gevent pool. Each worker process should hold one persistent psycopg3 connection (not a pool per process — the pool lives at the process level and Celery manages process count). Use `autocommit=False` and commit explicitly after each document write.

For the API layer, `asyncpg` remains the best async PostgreSQL driver (lower latency than psycopg3 async for high-concurrency scenarios).

**Do not use the Supabase Python client** (`supabase-py`) for anything other than Object Storage operations in the API. For direct DB access, bypass it entirely.

---

### 16. Gemini SDK

**Severity**: High

**Current/planned**: `google-generativeai` (implied).

**Recommended**: Use `google-genai` — the new unified Google AI Python SDK (released 2024, GA 2025). The older `google-generativeai` package is deprecated and will receive no new features. `google-cloud-aiplatform` is the Vertex AI SDK and is not appropriate unless the system is routing through Vertex AI endpoints specifically.

```python
from google import genai
from google.genai import types

client = genai.Client(api_key=settings.gemini_api_key)

def extract_fiscal_data(image_bytes: bytes) -> dict:
    response = client.models.generate_content(
        model="gemini-2.0-flash",
        contents=[
            types.Part.from_bytes(data=image_bytes, mime_type="image/jpeg"),
            "Extract all fiscal document fields as structured JSON. "
            "Return only valid JSON matching the NF-e schema.",
        ],
        config=types.GenerateContentConfig(
            response_mime_type="application/json",
            response_schema=NFeExtractedData,   # Pydantic model → JSON schema
        ),
    )
    return NFeExtractedData.model_validate_json(response.text)
```

**Async support**: `google-genai` exposes an async client at `genai.AsyncClient`. However, since image workers run under `gevent`, the sync client with gevent monkey-patching is acceptable and simpler. Use the async client only if the API layer needs to call Gemini directly (which it should not — Gemini calls belong in workers).

**Rate limit handling**: The `google-genai` SDK raises `google.genai.errors.APIError` with `status_code=429` for rate limit responses. Catch this in the Celery retry decorator:

```python
from google.genai.errors import APIError

@celery_app.task(base=IdempotentTask, name="invoice.tasks.image_process")
def process_image(document_id: str, object_key: str) -> None:
    try:
        result = extract_fiscal_data(image_bytes)
    except APIError as exc:
        if exc.status_code == 429:
            raise self.retry(exc=exc, countdown=60)
        raise  # non-retryable; will go to DLQ
```

---

### 17. Docker Base Image

**Severity**: Medium

**Current/planned**: Not yet specified.

**Recommended**: Use `python:3.12-slim-bookworm` as the base. Do not use distroless for this system.

**Why not distroless**: The `gcr.io/distroless/python3` image contains only the Python interpreter with no shell, no package manager, and no system library installation mechanism. This makes it incompatible with `poppler-utils` (which must be installed via `apt-get`) and `libmagic1`. Distroless is appropriate for pure-Python applications with no native library dependencies.

**Multi-stage build pattern** (incorporating findings #8 and #12):

```dockerfile
# docker/worker-cpu.Dockerfile
FROM python:3.12-slim-bookworm AS system-deps

RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    libmagic1 \
    && rm -rf /var/lib/apt/lists/*

FROM system-deps AS python-deps
WORKDIR /build
RUN pip install --no-cache-dir uv
COPY uv.lock pyproject.toml ./
COPY packages/shared ./packages/shared
COPY packages/workers ./packages/workers
RUN uv sync --frozen --no-dev --package invoice-workers

FROM system-deps AS runtime
WORKDIR /app
COPY --from=python-deps /build/.venv /app/.venv
COPY packages/workers/invoice_workers ./invoice_workers
COPY packages/shared/invoice_shared ./invoice_shared

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN useradd --no-create-home --shell /bin/false worker
USER worker

CMD ["celery", "-A", "invoice_shared.celery_app", "worker", \
     "--queues=pdf", "--pool=prefork", "--concurrency=4"]
```

Run the container as a non-root user (`USER worker`). ECS task definitions should also set `readonlyRootFilesystem: true` where possible; use a tmpfs mount for any temporary file writes (pdf2image's `output_folder`).

---

### 18. Python Version

**Severity**: Low

**Current/planned**: Python 3.12 (correct).

**Recommended**: Python 3.12 is the correct and current stable choice. Relevant features and gotchas for this stack:

- **`asyncio.TaskGroup`** (3.11+): Use for concurrent S3 downloads within the API layer if batches are processed in parallel.
- **Improved error messages**: `KeyError`, `AttributeError`, and `TypeError` messages are significantly more informative in 3.12, aiding debugging of Pydantic validation failures.
- **`@override` decorator** (`typing.override`): Available in 3.12 without `typing_extensions`. Use it on task `on_failure`/`on_retry` overrides in the base task class.
- **Free-threaded build (3.13t)**: Do not use the free-threaded (no-GIL) build yet. It is experimental in 3.13 and not yet stable. The prefork pool already provides true parallelism for CPU-bound work.
- **`tomllib`** (stdlib, 3.11+): If any worker reads its own `pyproject.toml` for configuration, `tomllib` is available in the stdlib — no external dependency needed.
- **Celery 5.4 and Python 3.12**: Celery 5.4 is the minimum version with full Python 3.12 support. Earlier versions have known incompatibilities with the 3.12 `datetime.utcnow()` deprecation that produce noisy warnings in logs.

---

## Implementation Checklist

### Before writing any code
- [ ] Confirm `uv` is installed and workspace structure matches finding #13
- [ ] Confirm all three Dockerfiles include `poppler-utils` and `libmagic1`
- [ ] Confirm Celery config sets `accept_content=["json"]` and both `acks_late` flags

### API package
- [ ] `lifespan` context manager creates Redis pool and asyncpg pool
- [ ] All endpoints are `async def`
- [ ] Classifier uses `magic.from_buffer(bytes)`, never a URL or file path
- [ ] Task payloads use shared Pydantic models, serialised to JSON before publishing

### Worker package
- [ ] `worker-io` Dockerfile uses `--pool=gevent`
- [ ] `worker-cpu` Dockerfile uses `--pool=prefork`
- [ ] All tasks have explicit `name=` parameter
- [ ] pdfplumber wrapped in `with` block; `None` guard on `extract_text()`
- [ ] pdf2image uses `output_folder=tmp` to avoid memory accumulation
- [ ] lxml parser has `resolve_entities=False, no_network=True, load_dtd=False, huge_tree=False`
- [ ] Pillow sets `MAX_IMAGE_PIXELS` at module import in worker process
- [ ] Gemini errors caught by status code; 429 triggers retry, others go to DLQ
- [ ] All DB writes use `INSERT ... ON CONFLICT DO UPDATE`

### Shared package
- [ ] `TaskPayload` discriminated union with `Literal` discriminator field
- [ ] `celery_app.py` is the single source of truth for Celery configuration
- [ ] `settings.py` uses `pydantic-settings` `BaseSettings` with `model_config = SettingsConfigDict(env_file=".env")`
