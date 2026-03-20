# CI/CD and DevOps Strategy: Event-Driven Fiscal Document Processing System

**Document Date:** 2026-03-19
**Scope:** Pre-implementation DevOps design — CI/CD, containers, secrets, observability, scaling, rollback
**Classification:** Internal
**Reviewer:** Claude Sonnet 4.6 (Automated DevOps Architecture Review)

---

## Executive Summary

This document defines the CI/CD and operational posture for a Python-based event-driven system that processes Brazilian SEFAZ fiscal documents across five services: a FastAPI API, three Celery worker branches (image, XML, PDF), and a Dead Letter Queue worker. The system spans Vercel (Next.js frontend), Railway or AWS ECS (API + workers), AWS ElastiCache (Redis), and Supabase (PostgreSQL + Object Storage).

The prior phase security audit identified no secrets management strategy, no container hardening, and no observability pipeline — all of which are prerequisites for safe deployment. The performance analysis identified that workers must be split into separate services per branch for independent scaling. This document addresses both in a single unified strategy.

**Findings at a glance:**

| Severity | Count |
|----------|-------|
| Critical | 3 |
| High | 7 |
| Medium | 7 |
| Low | 3 |

---

## Table of Contents

1. [CI Pipeline](#1-ci-pipeline)
2. [CD Pipeline](#2-cd-pipeline)
3. [Container Strategy](#3-container-strategy)
4. [Secrets Management](#4-secrets-management)
5. [Database Migrations](#5-database-migrations)
6. [Environment Strategy](#6-environment-strategy)
7. [Scaling Configuration](#7-scaling-configuration)
8. [Observability Setup](#8-observability-setup)
9. [Dependency Vulnerability Scanning](#9-dependency-vulnerability-scanning)
10. [Rollback Strategy](#10-rollback-strategy)

---

## 1. CI Pipeline

### FINDING CICD-001: No Automated Quality Gates on Pull Requests

- **Severity:** Critical
- **Operational risk if omitted:** Code with type errors, security vulnerabilities, or broken tests reaches staging undetected. For a system processing legally sensitive fiscal data, a production bug in CNPJ extraction or database write logic has direct compliance consequences.
- **Recommendation:**

Run the following checks on every pull request using GitHub Actions (or equivalent). All checks must pass before merge is permitted. The total CI runtime target is under 5 minutes for the fast gates; integration tests run in parallel and may extend to 10 minutes.

#### 1.1 Linting and Formatting

**Tool:** `ruff` (replaces flake8 + isort + pyupgrade in a single binary)

```yaml
# .github/workflows/ci.yml
- name: Lint
  run: ruff check . --output-format=github
- name: Format check
  run: ruff format --check .
```

`ruff` configuration in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"
select = ["E", "F", "I", "UP", "B", "S", "ANN"]
# S = bandit-equivalent security rules built into ruff
# ANN = annotation enforcement (works alongside mypy)
ignore = ["ANN101", "ANN102"]  # self/cls annotations

[tool.ruff.per-file-ignores]
"tests/**" = ["S101"]  # allow assert in tests
```

Note: `ruff`'s `S` ruleset covers most of what standalone `bandit` provides for common Python security anti-patterns (hardcoded passwords, use of `eval`, `subprocess` shell injection, etc.). Use `ruff S` as the first line of defence; supplement with `bandit` for a deeper scan on a separate schedule (see Finding CICD-009).

#### 1.2 Type Checking

**Tool:** `mypy` in strict mode

```toml
[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
# Required for Celery and SQLAlchemy which lack complete stubs
ignore_missing_imports = true
```

```yaml
- name: Type check
  run: mypy src/ --ignore-missing-imports
```

Run mypy against all application source directories (`src/api/`, `src/workers/`, `src/shared/`). Do not skip worker code from type checking — the Pydantic models that validate Gemini responses are the primary defense against prompt injection reaching the database, and mypy catches model definition errors that tests may miss.

#### 1.3 Unit and Security Configuration Tests

```yaml
- name: Unit tests
  run: pytest tests/unit/ tests/security/ -x --tb=short -q
  env:
    CELERY_TASK_ALWAYS_EAGER: "true"
```

The `tests/security/` suite (as defined in the Testing Strategy document) includes configuration assertions — `celery_app.conf.task_serializer == 'json'`, lxml parser hardening checks — which are fast and must run on every commit. Do not defer security configuration tests to integration or staging pipelines.

Coverage gate: `pytest --cov=src --cov-fail-under=85` enforced on unit tests. Per-component minimums are defined in the Testing Strategy (Classifier and URL validator require 100%).

#### 1.4 Integration Tests

```yaml
- name: Integration tests
  run: pytest tests/integration/ --tb=short -q
  services:
    redis:
      image: redis:7-alpine
      ports: ["6379:6379"]
    postgres:
      image: postgres:16
      env:
        POSTGRES_PASSWORD: test
      ports: ["5432:5432"]
```

Use `testcontainers-python` or GitHub Actions service containers for Redis and PostgreSQL. Gemini and Supabase Storage are mocked via `pytest-mock` and `moto`. Integration tests must run on every PR. Target runtime: under 4 minutes.

#### 1.5 Container Image Scanning

**Tool:** `trivy` (Aqua Security), run in CI against the built image

```yaml
- name: Build image
  run: docker build -t invoice-worker:pr-${{ github.sha }} -f docker/Dockerfile.workers .

- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: invoice-worker:pr-${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1  # fail CI on CRITICAL or HIGH CVEs
```

Upload the SARIF report to GitHub Security tab for visibility. Block merges on CRITICAL severity findings. HIGH severity findings generate a warning but do not block unless they are in production-facing libraries (`lxml`, `Pillow`, `pdfplumber`, `httpx`).

#### 1.6 Dependency Scanning

**Tool:** `pip-audit` for Python; `npm audit` for the Next.js frontend

```yaml
- name: Python dependency audit
  run: pip-audit --requirement requirements.txt --format markdown >> $GITHUB_STEP_SUMMARY
  continue-on-error: false  # fail on any known CVE with a fix available
```

See Section 9 for the full dependency scanning strategy including scheduled scans.

#### Complete CI Pipeline Summary

| Check | Tool | Trigger | Blocks Merge? |
|-------|------|---------|---------------|
| Linting + format | ruff | Every PR commit | Yes |
| Type checking | mypy strict | Every PR commit | Yes |
| Unit + security config tests | pytest | Every PR commit | Yes |
| Integration tests | pytest + testcontainers | Every PR | Yes |
| Image CVE scan | trivy | Every PR (after build) | Yes (CRITICAL) |
| Dependency audit | pip-audit | Every PR | Yes (fixable CVEs) |
| Performance regression | pytest-benchmark | Merge to main | Yes (>20% regression) |

---

## 2. CD Pipeline

### FINDING CICD-002: No Deployment Strategy for Multiple Services

- **Severity:** Critical
- **Operational risk if omitted:** Without a defined deployment order, a worker version that depends on a new database schema may be deployed before the migration runs. A bad image worker deployment can corrupt fiscal data records before it is caught. Services deployed together hide independent failure modes.
- **Recommendation:**

#### 2.1 Railway vs ECS Decision

The architecture documentation is ambiguous about the final deployment target (Railway vs AWS ECS). This document provides a concrete strategy for each, and recommends ECS for production given the system's operational requirements. Railway is appropriate for staging.

**Recommendation:** Use Railway for staging (lower operational overhead, faster iteration). Use AWS ECS Fargate for production (more granular autoscaling, IAM integration for secrets, VPC isolation for Redis).

The CI/CD pipeline must support both targets without requiring different application code.

#### 2.2 Service Decomposition

Based on the performance analysis requirement for independent scaling, the following services are deployed independently:

| Service | Railway Service | ECS Service | Queue |
|---------|----------------|-------------|-------|
| FastAPI API | `invoice-api` | `invoice-api` | — |
| Image worker | `worker-image` | `worker-image` | `queue:image` |
| XML worker | `worker-xml` | `worker-xml` | `queue:xml` |
| PDF worker | `worker-pdf` | `worker-pdf` | `queue:pdf` |
| DLQ worker | `worker-dlq` | `worker-dlq` | `queue:dlq` |

Each service has its own:
- Deployment configuration (CPU/memory allocation tuned per workload type)
- Autoscaling policy (different queue depth thresholds per branch)
- Log group
- Health check

#### 2.3 Deployment Order and Coordination

Deployments are not fully independent: migrations must precede worker deployments that depend on new schema columns.

**Enforced deployment order:**

```
1. Run Alembic migrations (one-off ECS task / Railway job)
   └─ Gate: migrations must complete successfully
2. Deploy FastAPI API
   └─ Gate: health check must pass (/health returns 200)
3. Deploy workers (image, XML, PDF, DLQ) in parallel
   └─ Gate: at least one task per service reaches RUNNING state
4. Run smoke tests against staging
   └─ Gate: E2E-001 (XML happy path) must pass within 60 seconds
```

Implement this in the CD pipeline as a GitHub Actions workflow with explicit job dependencies:

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/run-migration.sh

  deploy-api:
    needs: migrate
    steps:
      - run: ./scripts/deploy-service.sh invoice-api

  deploy-workers:
    needs: deploy-api
    strategy:
      matrix:
        service: [worker-image, worker-xml, worker-pdf, worker-dlq]
    steps:
      - run: ./scripts/deploy-service.sh ${{ matrix.service }}

  smoke-test:
    needs: deploy-workers
    steps:
      - run: pytest tests/e2e/test_happy_paths.py::test_xml_e2e -x
```

#### 2.4 Blue/Green Deployment for Workers

Workers consume tasks from Redis queues. A naive rolling update (killing old containers and starting new ones while tasks are in-flight) risks task loss if a worker is mid-processing when it is terminated.

Use blue/green deployment for workers:

1. Deploy the new worker version alongside the old version (both listening on the same queue).
2. Old workers complete their current tasks and exit gracefully (Celery `--soft-time-limit` ensures tasks have a bounded execution window).
3. New workers pick up all subsequent tasks from the queue.
4. After the old version's containers drain (configurable timeout, recommend 120 seconds for image workers; 30 seconds for XML), terminate the old containers.

On ECS: use the `ECS_CONTAINER_STOP_TIMEOUT` and `SIGTERM` → `SIGTERM` to Celery's `warm_shutdown` signal. Configure Celery with `worker_cancel_long_running_tasks_on_connection_loss = True` to handle unexpected disconnections.

On Railway: Railway's deployment uses a rolling strategy by default. For workers, set `healthcheckTimeout = 120` to allow in-flight tasks to complete before the old container is terminated.

#### 2.5 Deploy Together vs Independently

- **Migrations:** Always deploy first, independently, before any service.
- **API:** Deploy when API code changes, independent of worker changes (as long as it is backward-compatible with the current schema).
- **Workers:** Deploy when worker code changes. If the change requires a schema migration, the migration gate enforces correct ordering. Workers of different branch types (image/xml/pdf/dlq) deploy in parallel to each other.
- **Do not deploy all services on every PR merge.** Use path-based deployment triggers:

```yaml
on:
  push:
    paths:
      - 'src/workers/image/**'  # only triggers worker-image deployment
      - 'src/workers/xml/**'
      - 'src/api/**'            # only triggers API deployment
      - 'migrations/**'         # always runs migration job first
```

---

## 3. Container Strategy

### FINDING CICD-003: No Container Hardening or Multi-Stage Build Strategy

- **Severity:** High
- **Operational risk if omitted:** Workers that run as root can write to any filesystem path inside the container. A Pillow CVE exploiting a crafted image would have root access to the container and potentially the host (if using privileged mode). System dependencies installed without pinning introduce uncontrolled CVE surface.
- **Recommendation:**

#### 3.1 Base Image Selection

Use `python:3.12-slim-bookworm` as the base for all Python services. Do not use `python:3.12` (includes build tools and many packages not needed at runtime) or `alpine` (musl libc causes known compatibility issues with `lxml` and `Pillow`'s compiled extensions).

For the API (no system dependencies beyond libmagic): `python:3.12-slim-bookworm` is sufficient.

For workers: `python:3.12-slim-bookworm` with apt-installed system dependencies.

#### 3.2 Multi-Stage Dockerfile

Use a single `Dockerfile.workers` for all worker types, with build arguments to control which worker is started at runtime. This avoids maintaining four separate Dockerfiles while still allowing independent container builds.

```dockerfile
# ─── Stage 1: Build dependencies ───────────────────────────────────────────
FROM python:3.12-slim-bookworm AS builder

WORKDIR /build

# System dependencies required for building Python packages with C extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    libxml2-dev \
    libxslt1-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies into a virtual environment
# Using a venv allows clean copy to the runtime stage
COPY requirements.txt .
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --no-cache-dir --upgrade pip && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# ─── Stage 2: Runtime image ────────────────────────────────────────────────
FROM python:3.12-slim-bookworm AS runtime

# System dependencies required at runtime only
# - poppler-utils: required by pdf2image (PDF → image conversion)
# - libmagic1: required by python-magic (file type detection)
# - libxml2, libxslt1.1: required by lxml at runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    libmagic1 \
    libxml2 \
    libxslt1.1 \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Create a non-root user
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/bash --create-home appuser

# Copy virtual environment from builder (no compiler toolchain in runtime image)
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copy application source
COPY --chown=appuser:appgroup src/ ./src/

# Switch to non-root user before the final CMD
USER 1001

# WORKER_TYPE is set per-service in Railway/ECS environment variables
# Valid values: image | xml | pdf | dlq
ARG WORKER_TYPE=image
ENV WORKER_TYPE=${WORKER_TYPE}

CMD celery -A src.workers.celery_app worker \
    --queues=queue:${WORKER_TYPE} \
    --loglevel=info \
    --without-gossip \
    --without-mingle \
    --pool=${CELERY_POOL:-prefork} \
    --concurrency=${CELERY_CONCURRENCY:-4}
```

**API Dockerfile** (separate, simpler — no poppler, no libmagic needed at API layer if classification is moved to workers per the performance analysis recommendation):

```dockerfile
FROM python:3.12-slim-bookworm AS builder
# ... same builder pattern ...

FROM python:3.12-slim-bookworm AS runtime
# No system dependencies beyond the base — libmagic only if Classifier stays in API
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/bash --create-home appuser

COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
WORKDIR /app
COPY --chown=appuser:appgroup src/api/ ./src/api/
USER 1001

CMD uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --workers 2
```

#### 3.3 System Dependency Pinning

`poppler-utils` and `libmagic1` installed via `apt-get` receive the version shipped by Debian Bookworm's security mirrors. This is not pinned to a specific version — it tracks Debian's security updates, which is correct behavior for OS-level packages. Do not pin apt packages to specific versions unless a known regression exists; pin Python packages via `requirements.txt` with exact versions (`==`).

#### 3.4 One Image vs Separate Images Per Branch

**Decision: One image, runtime WORKER_TYPE environment variable.**

Rationale:
- All workers share the same Python dependencies and system libraries. Separate images would duplicate the 400–600 MB runtime layer.
- The `poppler-utils` layer (for the PDF branch) adds ~30 MB. Keeping it in the unified image is acceptable; the alternatives (a PDF-only image and a non-PDF image) add pipeline complexity without meaningful security benefit.
- Worker-specific behavior is controlled at process startup time by `--queues=queue:${WORKER_TYPE}` and the Celery routing configuration. No conditional compile-time behavior is needed.

The security concern with a unified image is that the image worker has `poppler-utils` installed even though it does not use it. This is an acceptable trade-off. If strict least-privilege containers are required, split into two images: `Dockerfile.workers.light` (image + XML, no poppler) and `Dockerfile.workers.pdf` (PDF + DLQ, with poppler).

#### 3.5 Read-Only Root Filesystem (ECS)

On ECS, configure `readonlyRootFilesystem: true` in the task definition and mount a writable tmpfs for temporary file processing:

```json
{
  "readonlyRootFilesystem": true,
  "mountPoints": [
    {
      "sourceVolume": "tmp-volume",
      "containerPath": "/tmp",
      "readOnly": false
    }
  ]
}
```

Workers should write all temporary files (downloaded documents, rasterized frames) to `/tmp`. This volume is ephemeral and worker-scoped. On Railway, the equivalent is setting the container's `tmpfs` mount in the Railway configuration.

#### 3.6 Container Memory Limits

Per the security audit (SEC-016) and performance analysis, set hard memory limits per worker type to prevent a decompression bomb or page bomb from consuming all available host memory:

| Worker | Memory reservation | Memory limit (hard) |
|--------|-------------------|--------------------|
| API | 256 MB | 512 MB |
| Image worker | 512 MB | 1024 MB |
| XML worker | 256 MB | 512 MB |
| PDF worker | 1024 MB | 2048 MB |
| DLQ worker | 128 MB | 256 MB |

These limits are starting points. Tune based on observed p99 memory usage in staging before setting them in production.

---

## 4. Secrets Management

### FINDING CICD-004: No Secrets Management Strategy (Carried from Security Audit SEC-019)

- **Severity:** Critical
- **Operational risk if omitted:** Secrets stored in version control or deployed as unencrypted environment variables are exposed in CI logs, container images, and on compromise of any one provider. With four infrastructure providers (Vercel, Railway, AWS, Supabase), secret sprawl without a central strategy means rotation is manual, inconsistent, and likely to be skipped.
- **Recommendation:**

#### 4.1 Secrets Inventory

| Secret | Used By | Classification |
|--------|---------|----------------|
| Supabase anon key | Next.js frontend | Low sensitivity (public-facing by design, scoped) |
| Supabase service role key | FastAPI API, Workers | High — full database access |
| Supabase DB connection string (pooler) | FastAPI API, Workers | High — database credentials |
| Redis AUTH password | FastAPI API, all Workers | High — queue access |
| Gemini API key | Image + PDF workers | High — external API billing |
| Supabase JWT secret | FastAPI API (token verification) | High — authentication bypass if leaked |
| Internal API signing key (if used) | API ↔ Workers | Medium |

#### 4.2 Per-Platform Secret Storage

**Vercel (Next.js frontend):**
- Use Vercel Environment Variables (encrypted at rest, not exposed in build logs).
- Store only `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
- Never store the service role key in Vercel. The frontend has no legitimate use for it.
- Use Vercel's environment promotion (Development → Preview → Production) to ensure staging and production use separate Supabase projects.

**Railway (API + workers, staging):**
- Use Railway's Service Variables (encrypted, injected as environment variables at container startup).
- Create separate Railway environments: `staging` and (optionally) `production`.
- Use Railway's `Shared Variables` for secrets shared across services within an environment (e.g., `REDIS_URL`, `DATABASE_URL`).
- Set per-service overrides for worker-type-specific credentials (e.g., only `worker-image` and `worker-pdf` receive `GEMINI_API_KEY`).

```
Railway staging environment:
  Shared:
    REDIS_URL=rediss://:password@redis-private:6379/0
    DATABASE_URL=postgresql://user:pass@db-pooler:5432/db
  invoice-api service:
    SUPABASE_SERVICE_ROLE_KEY=...
    SUPABASE_JWT_SECRET=...
  worker-image service:
    GEMINI_API_KEY=...
  worker-pdf service:
    GEMINI_API_KEY=...
  worker-xml service:
    (no Gemini key — least privilege)
  worker-dlq service:
    (no Gemini key, no storage write key — least privilege)
```

**AWS ECS (API + workers, production):**
- Store all production secrets in **AWS Secrets Manager** as JSON objects.
- Reference them in ECS task definitions via the `secrets` block (not `environment`). ECS injects them as environment variables at task startup without exposing them in the task definition JSON.

```json
{
  "secrets": [
    {
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/invoice/db-url"
    },
    {
      "name": "REDIS_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/invoice/redis-url"
    },
    {
      "name": "GEMINI_API_KEY",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/invoice/gemini-key"
    }
  ]
}
```

- Assign the ECS task execution role IAM permission to read only the specific secrets it needs (`secretsmanager:GetSecretValue` on the specific ARN, not `*`).
- Different task definitions for `worker-image` and `worker-xml` reference different secrets. XML workers do not have access to the Gemini API key secret ARN.

**Supabase:**
- Supabase project-level secrets (JWT secret, service role key) are managed in the Supabase dashboard.
- Export them to Railway/AWS Secrets Manager as the authoritative store for use by other services.
- Do not rely on Supabase dashboard as the only copy — maintain them in the central secret store.

#### 4.3 Secret Rotation Strategy

| Secret | Rotation Frequency | Method |
|--------|-------------------|--------|
| Database password | 30 days | AWS Secrets Manager rotation lambda; update Railway secrets manually for staging |
| Redis AUTH password | 60 days | Manual rotation with blue/green — update all consumers before removing old password |
| Gemini API key | 90 days | Google Cloud console rotation; immediately on any suspected exposure |
| Supabase service role key | 90 days | Supabase dashboard rotation |
| JWT secret | 180 days | Coordinate with frontend (invalidates all active sessions) |

Automate rotation reminders via a scheduled GitHub Actions workflow that checks secret age in AWS Secrets Manager and opens a GitHub issue when any secret approaches its rotation deadline.

#### 4.4 Secret Hygiene Rules

- `.env` files in the repository: committed `.env.example` only, with placeholder values. `.env` is in `.gitignore`.
- CI/CD: secrets are passed as GitHub Actions secrets (`${{ secrets.GEMINI_API_KEY }}`), not hardcoded in workflow YAML.
- Logs: secrets must never appear in application logs. Use `structlog`'s `redact_keys` configuration to strip any accidentally logged secrets.
- Container images: `docker history` must not reveal secrets. The multi-stage build ensures no secrets are present in the build stage layers that ship to production. Verify with `docker history --no-trunc`.

---

## 5. Database Migrations

### FINDING CICD-005: No Migration Strategy Before First Deployment

- **Severity:** High
- **Operational risk if omitted:** Schema changes deployed with a new worker version but without a migration leave the database in an inconsistent state. Workers that try to write to a non-existent column fail silently or crash. Without a migration history, recovering the schema state after a botched deployment requires manual intervention against a production database containing legally sensitive fiscal data.
- **Recommendation:**

#### 5.1 Tool: Alembic

Use `alembic` for all PostgreSQL schema migrations. Configure it to connect to Supabase PostgreSQL via the direct connection URL (not PgBouncer — Alembic requires a direct connection that supports DDL transactions).

```
alembic/
  env.py          # reads DATABASE_URL from environment
  versions/       # migration scripts, committed to version control
  alembic.ini     # base configuration
```

```python
# alembic/env.py
import os
from alembic import context
from sqlalchemy import engine_from_config

def run_migrations_online():
    config = context.config
    config.set_main_option("sqlalchemy.url", os.environ["DATABASE_DIRECT_URL"])
    # DATABASE_DIRECT_URL: Supabase direct connection (not pooler)
    # Separate from DATABASE_URL (pooler) used by the application
```

Maintain two database connection strings in secrets:
- `DATABASE_URL`: PgBouncer pooler endpoint — used by FastAPI and all workers at runtime.
- `DATABASE_DIRECT_URL`: Supabase direct connection — used only by Alembic migrations.

#### 5.2 Migration Execution in CD Pipeline

Migrations run as a one-off job before any service deployment. On ECS:

```bash
# Run as a one-off ECS task (not a service)
aws ecs run-task \
  --cluster invoice-prod \
  --task-definition invoice-migration \
  --overrides '{"containerOverrides":[{"name":"migration","command":["alembic","upgrade","head"]}]}' \
  --launch-type FARGATE \
  --network-configuration awsvpcConfiguration={subnets=[...],securityGroups=[...]}
```

On Railway, use a Railway one-off job:

```bash
railway run --service invoice-api alembic upgrade head
```

The CD pipeline waits for the migration job to exit 0 before proceeding to deploy any service. Exit code 1 blocks the entire deployment.

#### 5.3 Safe Migration Patterns

Workers may be processing tasks from the old schema version while migrations are running (blue/green worker deployment). Migrations must be backward-compatible with the previous worker version. Follow the expand-contract pattern:

**Expand (migration):** Add new columns as nullable. Old workers ignore unknown columns; new workers write to the new column.

```sql
-- Safe: add nullable column, old workers unaffected
ALTER TABLE documents ADD COLUMN processing_node_id UUID NULL;
```

**Contract (subsequent migration, after old workers are drained):** Add `NOT NULL` constraint or drop old columns only after all old worker containers have been replaced.

Never:
- Drop a column in the same migration that adds a replacement column.
- Rename a column without an intermediate migration that keeps both names.
- Add a `NOT NULL` column without a default value while any old worker version is running.

#### 5.4 Migration Safety Checks in CI

Add a migration check to CI that verifies:
1. Every migration is reversible (`alembic downgrade -1` succeeds on a test database).
2. The migration does not contain patterns that lock production tables (e.g., `ALTER TABLE ... ADD COLUMN NOT NULL` without a default in PostgreSQL < 11; Supabase uses PostgreSQL 15+, which handles this safely for most cases, but `LOCK TABLE` statements are still dangerous).

```yaml
- name: Test migrations
  run: |
    # Spin up test PostgreSQL
    # Apply all migrations
    alembic upgrade head
    # Test reversibility
    alembic downgrade -1
    alembic upgrade head
```

---

## 6. Environment Strategy

### FINDING CICD-006: No Environment Parity or Local Development Stack

- **Severity:** High
- **Operational risk if omitted:** Bugs that only manifest in specific environments (e.g., a Celery configuration that works in Eager mode but fails against a real Redis broker) are discovered in staging or production rather than during development. Without a local full-stack environment, developers cannot reproduce production issues.
- **Recommendation:**

#### 6.1 Environment Tiers

| Environment | Purpose | Infrastructure |
|------------|---------|----------------|
| `local` | Developer iteration | docker-compose, local Redis + PostgreSQL |
| `staging` | Integration and E2E testing | Railway (all Python services) + Supabase staging project |
| `production` | Live system | AWS ECS + ElastiCache + Supabase production project |

Each environment uses **separate Supabase projects** (separate database, separate storage bucket, separate auth configuration). Never share a Supabase project between staging and production. This is not just operational hygiene — Supabase Row Level Security policies and storage bucket policies are configured at the project level; a misconfiguration in staging could affect production if they share a project.

#### 6.2 Local Development with docker-compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/invoice_dev
      - REDIS_URL=redis://:localpassword@redis:6379/0
      - SUPABASE_URL=${SUPABASE_STAGING_URL}       # use staging Supabase
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_STAGING_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src  # hot reload in development

  worker-image:
    build:
      context: .
      dockerfile: docker/Dockerfile.workers
    environment:
      - WORKER_TYPE=image
      - CELERY_POOL=prefork
      - CELERY_CONCURRENCY=2
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/invoice_dev
      - REDIS_URL=redis://:localpassword@redis:6379/0
      - GEMINI_API_KEY=${GEMINI_API_KEY_DEV}
    depends_on: [redis, postgres]

  worker-xml:
    build:
      context: .
      dockerfile: docker/Dockerfile.workers
    environment:
      - WORKER_TYPE=xml
      - CELERY_CONCURRENCY=2
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/invoice_dev
      - REDIS_URL=redis://:localpassword@redis:6379/0
    depends_on: [redis, postgres]

  worker-pdf:
    build:
      context: .
      dockerfile: docker/Dockerfile.workers
    environment:
      - WORKER_TYPE=pdf
      - CELERY_CONCURRENCY=1
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/invoice_dev
      - REDIS_URL=redis://:localpassword@redis:6379/0
      - GEMINI_API_KEY=${GEMINI_API_KEY_DEV}
    depends_on: [redis, postgres]

  worker-dlq:
    build:
      context: .
      dockerfile: docker/Dockerfile.workers
    environment:
      - WORKER_TYPE=dlq
      - CELERY_CONCURRENCY=1
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/invoice_dev
      - REDIS_URL=redis://:localpassword@redis:6379/0
    depends_on: [redis, postgres]

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass localpassword
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "localpassword", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: invoice_dev
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

Developers run `docker-compose up` to bring up the full stack. The Gemini API is mocked in local development using a mock server (`docker run -p 9090:9090 gemini-mock`) or by setting `GEMINI_MOCK=true` which triggers the application's test double.

Alternatively, developers can run workers outside Docker (for faster iteration with hot reload) by installing dependencies locally and pointing at the containerized Redis + PostgreSQL.

#### 6.3 Staging Environment Parity

Staging must reflect production as closely as possible:

- Same Docker images (built from the same Dockerfile, same tag).
- Same Celery configuration (pool type, concurrency) with reduced replica counts.
- Same Alembic migration history (staging database applies all migrations, including pending ones being tested).
- Real Redis (Railway Redis addon with TLS) — not an in-memory substitute.
- Real Supabase Storage (staging project) — not a mock S3.
- Separate Gemini API project with its own quota (so staging tests do not consume production Gemini quota).

The primary difference between staging and production: replica counts, ECS vs Railway, and ElastiCache vs Railway Redis.

---

## 7. Scaling Configuration

### FINDING CICD-007: No Autoscaling Policy for Independent Worker Branches

- **Severity:** High
- **Operational risk if omitted:** Without autoscaling, either workers are over-provisioned (wasteful cost during off-peak hours) or under-provisioned (queue depth grows unboundedly during burst load, increasing document processing latency from seconds to hours). The performance analysis confirmed that co-located workers cannot scale independently — image surges must not prevent XML from scaling separately.
- **Recommendation:**

#### 7.1 ECS Autoscaling (Production)

Celery queue depth is not a native CloudWatch metric. Publish it as a custom metric from a lightweight CloudWatch metric publisher (a scheduled Lambda or a Celery beat task running on the DLQ worker):

```python
# tasks/metrics.py — runs every 30 seconds via celery beat
@celery_app.task
def publish_queue_metrics():
    import boto3
    r = redis.from_url(settings.REDIS_URL)
    cw = boto3.client("cloudwatch")

    for queue in ["image", "xml", "pdf", "dlq"]:
        depth = r.llen(f"queue:{queue}")
        cw.put_metric_data(
            Namespace="Invoice/Celery",
            MetricData=[{
                "MetricName": "QueueDepth",
                "Dimensions": [{"Name": "Queue", "Value": queue}],
                "Value": depth,
                "Unit": "Count"
            }]
        )
```

Define ECS Application Auto Scaling policies per service, keyed on `Invoice/Celery/QueueDepth`:

| Service | Scale-Out Threshold | Scale-In Threshold | Min Tasks | Max Tasks | Cooldown |
|---------|--------------------|--------------------|-----------|-----------|---------|
| worker-image | > 50 tasks | < 10 tasks | 1 | 20 | 120s out / 300s in |
| worker-xml | > 200 tasks | < 50 tasks | 1 | 10 | 60s out / 180s in |
| worker-pdf | > 20 tasks | < 5 tasks | 1 | 8 | 120s out / 300s in |
| worker-dlq | > 0 tasks | queue empty | 1 | 3 | 60s out / 600s in |

Scale-out cooldowns are intentionally shorter than scale-in. Scale in conservatively to avoid oscillation.

Keep a minimum of 1 task per service running at all times to eliminate the ECS cold-start latency (45–135 seconds for first container) for moderate load. Scale to zero only for `worker-dlq` during sustained off-peak periods if DLQ depth is 0.

The image worker is bounded by the Gemini API rate limit regardless of replica count. Set `max_tasks=20` but enforce a distributed Redis token bucket (as described in the performance analysis) so that scaling workers beyond the Gemini quota does not cause a 429 cascade.

#### 7.2 Railway Autoscaling (Staging)

Railway supports autoscaling based on CPU and memory utilization (as of 2026). For queue-depth-based scaling, Railway does not natively support custom metrics as scaling triggers.

For staging, use Railway's horizontal scaling with a fixed replica count per service:

| Service | Staging Replicas |
|---------|-----------------|
| invoice-api | 1 |
| worker-image | 2 |
| worker-xml | 1 |
| worker-pdf | 1 |
| worker-dlq | 1 |

This is sufficient for staging load. Queue-depth-based autoscaling is a production-only concern.

#### 7.3 Redis Scaling

**AWS ElastiCache (production):**
- Use ElastiCache for Redis 7.x in cluster mode disabled (single primary + one replica) for simplicity at v1.
- Enable Multi-AZ with automatic failover.
- Instance type: `cache.t4g.medium` (3.84 GB) is adequate for the expected queue depths described in the performance analysis (<100,000 tasks at ~500 bytes = ~50 MB).
- Enable AOF persistence (`appendfsync everysec`) to prevent task loss on node replacement.
- Set `maxmemory-policy noeviction` — never silently evict tasks from the queue. Raise an alarm instead.

**Railway Redis addon (staging):**
- Railway's Redis addon is a managed Redis instance with TLS enabled by default.
- Use the `rediss://` (TLS) URL scheme for all connections.
- Set `maxmemory 512mb` in Railway's Redis configuration.

#### 7.4 Database Scaling

Supabase PostgreSQL scales vertically (upgrading instance size) rather than horizontally. Use the Supabase PgBouncer pooler endpoint for all application connections (not the direct connection string). This is already documented in Section 5.

Monitor connection count via Supabase's built-in metrics dashboard. If connection count approaches the plan limit, upgrade the Supabase plan (higher limits) before adding worker replicas.

---

## 8. Observability Setup

### FINDING CICD-008: No Observability Pipeline Defined

- **Severity:** High
- **Operational risk if omitted:** When a worker silently processes a scanned PDF and stores an incorrect extraction (wrong CNPJ, wrong amount), there is no signal that anything went wrong. Without structured logs and metrics, debugging a production incident in a Brazilian fiscal compliance system requires manually querying the database — too slow for incident response. Without alerting, a full queue (all documents stuck in DLQ) is invisible until a client complains.
- **Recommendation:**

#### 8.1 Structured Logging with structlog

Install `structlog` and configure it as the logging backend for all Python services. Configure structlog to output JSON in production and human-readable colored output in local development.

```python
# src/shared/logging.py
import structlog
import logging
import os

def configure_logging():
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]

    if os.environ.get("ENV") == "production":
        processors = shared_processors + [
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),
        ]
    else:
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(),
        ]

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )
```

Bind context variables at the start of each Celery task so all log lines from the task carry identifying metadata without explicitly passing them:

```python
# src/workers/base_task.py
import structlog
from celery import Task

log = structlog.get_logger()

class InvoiceBaseTask(Task):
    def __call__(self, *args, **kwargs):
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            task_id=self.request.id,
            task_name=self.name,
            batch_id=kwargs.get("batch_id"),
            document_id=kwargs.get("document_id"),
            worker_type=os.environ.get("WORKER_TYPE"),
        )
        return super().__call__(*args, **kwargs)
```

**Log format for Celery task events** (each line is a JSON object):

```json
{"timestamp": "2026-03-19T14:32:01.123Z", "level": "info", "event": "task.started", "task_id": "abc123", "task_name": "process_image", "batch_id": "uuid-batch", "document_id": "uuid-doc", "worker_type": "image"}
{"timestamp": "2026-03-19T14:32:05.456Z", "level": "info", "event": "gemini.request", "task_id": "abc123", "document_id": "uuid-doc", "image_size_kb": 312}
{"timestamp": "2026-03-19T14:32:07.789Z", "level": "info", "event": "task.completed", "task_id": "abc123", "document_id": "uuid-doc", "duration_ms": 6666, "extraction_fields_count": 12}
```

**PII logging policy:** Never log CNPJ, CPF, monetary amounts, issuer names, or recipient names. Log only document IDs (UUIDs), processing stages, durations, and error types. This is enforced by logging policy documentation and validated by a `grep`-based CI check on log output from integration test runs:

```yaml
- name: Check for PII in logs
  run: |
    pytest tests/integration/ 2>&1 | \
    python scripts/check_log_pii.py  # rejects lines containing CNPJ patterns
```

#### 8.2 Metrics with OpenTelemetry

Use `opentelemetry-sdk` with the `opentelemetry-instrumentation-fastapi` and `opentelemetry-instrumentation-celery` auto-instrumentation packages. These emit spans and metrics without requiring manual instrumentation in most cases.

```python
# src/shared/telemetry.py
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

def configure_telemetry(service_name: str):
    provider = TracerProvider(
        resource=Resource.create({"service.name": service_name})
    )
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint=os.environ["OTEL_ENDPOINT"]))
    )
    trace.set_tracer_provider(provider)
```

Custom metrics to define beyond auto-instrumentation:

| Metric | Type | Labels | Purpose |
|--------|------|--------|---------|
| `invoice.documents.processed` | Counter | `worker_type`, `status` (completed/failed) | Track extraction success rate |
| `invoice.gemini.latency_ms` | Histogram | — | Gemini API response time distribution |
| `invoice.gemini.rate_limit_hits` | Counter | — | 429 rate — primary alert trigger |
| `invoice.queue.depth` | Gauge | `queue_name` | Queue depth per branch |
| `invoice.task.duration_ms` | Histogram | `worker_type` | End-to-end task processing time |
| `invoice.dlq.depth` | Gauge | — | DLQ depth — alert when non-zero |

#### 8.3 Where to Ship (Backend Recommendation)

**Recommended: Grafana Cloud (Loki + Tempo + Prometheus)**

Rationale:
- Grafana Cloud's free tier covers the expected log volume and metric cardinality for v1.
- A single Grafana instance can ingest logs (Loki), traces (Tempo), and metrics (Prometheus-compatible) from all four infrastructure providers (Vercel, Railway, AWS ECS, Supabase) via a unified OpenTelemetry collector.
- Avoids vendor lock-in: the OTel collector can be switched to Datadog or CloudWatch without changing application code.

Alternative: **AWS CloudWatch** if the system is primarily ECS-based and the team prefers staying within AWS. CloudWatch has higher per-GB log ingestion costs but native ECS integration (Container Insights, metric filters on log groups).

Do not use Datadog for v1 unless it is already part of the organization's tooling stack. Datadog's cost at scale can exceed the compute cost of the system itself.

**Deployment for Grafana Cloud:**

```yaml
# docker-compose.yml — add to local stack
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  volumes:
    - ./otel-config.yaml:/etc/otelcol/config.yaml
  ports:
    - "4317:4317"   # OTLP gRPC
    - "4318:4318"   # OTLP HTTP
```

```yaml
# otel-config.yaml
exporters:
  loki:
    endpoint: https://logs-prod-us-central1.grafana.net/loki/api/v1/push
    auth:
      authenticator: basicauth/grafana
  otlphttp/tempo:
    endpoint: https://tempo-us-central1.grafana.net/tempo
  prometheusremotewrite:
    endpoint: https://prometheus-us-central1.grafana.net/api/prom/push
```

#### 8.4 Alerting Rules

Configure the following alerts in Grafana (or CloudWatch Alarms):

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| DLQ non-empty | `invoice.dlq.depth > 0` for 5 minutes | Warning | Slack #ops channel |
| Gemini rate limit spike | `invoice.gemini.rate_limit_hits > 10` in 5 minutes | Warning | Slack #ops channel |
| Task failure rate | `failed / total > 10%` over 10 minutes | Critical | PagerDuty |
| Worker offline | No `task.started` events from a worker type for 5 minutes while queue depth > 0 | Critical | PagerDuty |
| Queue depth unbounded | `invoice.queue.depth > 5000` for any queue | Warning | Slack #ops channel |
| Image worker memory > 80% of limit | Container memory approaching hard limit | Warning | Slack #ops channel |

---

## 9. Dependency Vulnerability Scanning

### FINDING CICD-009: No CVE Scanning Cadence for Python Dependencies

- **Severity:** Medium
- **Operational risk if omitted:** `lxml`, `Pillow`, `pdfplumber`, and `pdf2image` (via Poppler) have documented histories of CVEs involving memory corruption and code execution during file parsing. Workers process untrusted user-supplied files using these libraries. A known CVE in a production dependency processing fiscal documents creates regulatory exposure (LGPD breach notification obligations) and operational risk.
- **Recommendation:**

#### 9.1 In-PR Scanning

Run `pip-audit` on every pull request (configured in CICD-001 above). Fail the build if a vulnerability with an available fix is found in any direct or transitive dependency.

```bash
pip-audit \
  --requirement requirements.txt \
  --format markdown \
  --output audit-report.md \
  --ignore-vuln GHSA-xxxx-xxxx  # only for known false positives with documented rationale
```

Upload the audit report as a PR artifact.

#### 9.2 Scheduled Weekly Scan

New CVEs are published continuously against existing dependency versions. Weekly scans catch vulnerabilities that were not present when the PR was merged.

```yaml
# .github/workflows/weekly-audit.yml
on:
  schedule:
    - cron: '0 8 * * 1'  # every Monday 8:00 UTC

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Python audit
        run: pip-audit --requirement requirements.txt --format json > audit.json
        continue-on-error: true

      - name: Open issue on new CVEs
        run: python scripts/audit-to-github-issue.py audit.json
        # Creates a GitHub issue if new CVEs are found that were not in the previous scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `scripts/audit-to-github-issue.py` script compares the current scan against the cached scan from the previous week (stored as a GitHub Actions artifact) and creates a GitHub issue only for net-new findings. This prevents alert fatigue from repeatedly reopening the same issue for known CVEs under evaluation.

#### 9.3 Container Image Scanning Cadence

`trivy` in CI (CICD-001) scans the image at build time. For images that have not been rebuilt in a while (e.g., a stable worker with no code changes for 30 days), OS-level CVEs accumulate without being caught.

Add a scheduled weekly `trivy` scan of the production image tags currently running in ECS:

```yaml
- name: Scan deployed image
  run: |
    IMAGE=$(aws ecs describe-task-definition \
      --task-definition worker-image \
      --query 'taskDefinition.containerDefinitions[0].image' \
      --output text)
    trivy image $IMAGE --severity CRITICAL,HIGH --exit-code 1
```

This catches CVEs in the OS layer (Debian Bookworm packages) that are fixed upstream but not yet rebuilt into a new image.

#### 9.4 Tool Coverage Summary

| Layer | Tool | Trigger | Blocks Deployment? |
|-------|------|---------|-------------------|
| Python packages | pip-audit | Every PR | Yes (fixable CVEs) |
| Python packages | pip-audit | Weekly scheduled | No (creates issue) |
| Container OS + packages | trivy | Every PR (image build) | Yes (CRITICAL) |
| Container OS + packages | trivy | Weekly scheduled (deployed image) | No (creates issue) |
| Next.js packages | npm audit | Every PR (Vercel build) | Warning only |

---

## 10. Rollback Strategy

### FINDING CICD-010: No Rollback Procedure for Bad Worker Deployments

- **Severity:** Medium
- **Operational risk if omitted:** If a deployed worker version has a bug in Gemini response parsing that stores incorrect extraction data (wrong CNPJ, wrong monetary value), and there is no rollback procedure, the team faces an open-ended incident: how many documents were affected? Which records need correction? How do we stop the bleeding while fixing the code? Without a defined procedure, the answer is manual and slow.
- **Recommendation:**

#### 10.1 Worker Rollback Procedure

Worker rollbacks are complicated by three factors:
1. Tasks already consumed from the queue by the bad version are processed and cannot be un-consumed.
2. Incorrect data may already be written to PostgreSQL.
3. The bad version may be actively processing more tasks while the rollback is being executed.

**Immediate response (0–5 minutes after detection):**

1. **Stop the bad worker:** Scale the affected worker service to 0 replicas. On ECS: `aws ecs update-service --service worker-image --desired-count 0`. On Railway: scale to 0 from the dashboard or `railway scale --replicas 0`.
2. **Verify the queue is not being consumed:** Confirm queue depth is stable (not decreasing) in Grafana.
3. **Do not flush the queue.** Tasks not yet consumed by the bad version are still valid and will be re-processed by the correct version.

**Identification (5–15 minutes):**

4. Identify the bad deployment by timestamp: determine which `document_id`s were processed after the bad version was deployed. Query: `SELECT id FROM documents WHERE updated_at > '<deploy_timestamp>' AND status = 'completed' AND worker_type = 'image'`.
5. Mark those documents for re-review: `UPDATE documents SET status = 'needs_review', review_reason = 'bad_worker_version_rollback' WHERE id IN (...)`.

**Rollback (15–30 minutes):**

6. **Redeploy the previous image version.** On ECS, update the service to use the previous task definition revision: `aws ecs update-service --service worker-image --task-definition worker-image:PREVIOUS_REVISION`. On Railway, redeploy from the previous deployment in the Railway dashboard.
7. Wait for the previous version's health check to pass.
8. **Re-queue affected documents for reprocessing:** For documents marked `needs_review`, issue new Celery tasks to reprocess them. The re-queued task will overwrite the bad extraction with the correct one.

**Recovery (30 minutes – several hours):**

9. Monitor the reprocessing tasks via Grafana. Confirm `status = 'completed'` for each affected document.
10. Remove the `needs_review` status after validation.
11. Post a post-incident report documenting: how many documents were affected, what data was incorrect, and what code change caused the bug.

#### 10.2 API Rollback Procedure

API rollbacks are simpler: the API does not write fiscal extraction data, only queues tasks and queries status. A bad API deployment causes task queuing failures (documents never processed) rather than data corruption.

1. Scale the bad API version to 0.
2. Redeploy the previous image version.
3. Any tasks that failed to queue (returned an error to the client during the bad window) must be manually re-submitted by the client or re-queued from error logs.

#### 10.3 Migration Rollback Procedure

Database migrations that add columns or tables can be reversed with `alembic downgrade -1` if the migration is reversible (expand-contract pattern enforced in Section 5). Migrations that delete data or drop columns are irreversible without a backup.

**Pre-production rule:** Before every migration that drops a column or table, create a manual Supabase database backup. Supabase provides point-in-time recovery (PITR) on paid plans — verify it is enabled.

Migration rollback procedure:
1. Scale all workers and the API to 0.
2. Run `alembic downgrade -1` via a one-off ECS/Railway job.
3. Redeploy the previous version of all services.
4. Scale services back to their normal replica counts.

Never roll back a migration while any worker version that depends on the migrated schema is running.

#### 10.4 Rollback Decision Criteria

Define explicit criteria for when a rollback is required (as opposed to a fast-forward fix):

| Scenario | Action |
|----------|--------|
| Worker crashes immediately after deploy (< 5 minutes, 0 documents processed) | Rollback — no data affected |
| Worker processes documents but stores wrong data (detected < 1 hour) | Rollback + re-queue affected documents |
| Worker processes documents but stores wrong data (detected > 4 hours) | Evaluate: rollback may not be faster than a fix forward. Assess re-queue scope first. |
| API returns 500 on task submission (documents not queued) | Rollback API; no data corruption, no re-queue needed |
| Migration fails midway | Alembic transactional rollback (PostgreSQL DDL is transactional); verify state and retry |

---

## Summary Table

| # | Finding | Severity | Area |
|---|---------|----------|------|
| CICD-001 | No automated quality gates on PRs | **Critical** | CI Pipeline |
| CICD-002 | No deployment strategy for multiple services | **Critical** | CD Pipeline |
| CICD-003 | No container hardening or multi-stage builds | **High** | Container Strategy |
| CICD-004 | No secrets management strategy | **Critical** | Secrets |
| CICD-005 | No migration strategy before first deployment | **High** | Database Migrations |
| CICD-006 | No environment parity or local development stack | **High** | Environment Strategy |
| CICD-007 | No autoscaling policy for independent worker branches | **High** | Scaling |
| CICD-008 | No observability pipeline defined | **High** | Observability |
| CICD-009 | No CVE scanning cadence for Python dependencies | **Medium** | Security Scanning |
| CICD-010 | No rollback procedure for bad worker deployments | **Medium** | Rollback |

---

## Implementation Checklist (Pre-First Deployment)

The following items are blockers before any deployment to a network-accessible environment. Items not in this list are important but can be completed during the initial development sprint.

- [ ] GitHub Actions CI workflow with all gates from CICD-001
- [ ] `pyproject.toml` with ruff, mypy strict, and coverage configuration
- [ ] Multi-stage `Dockerfile.workers` and `Dockerfile.api` with non-root user
- [ ] `docker-compose.yml` for full local stack
- [ ] Secrets inventory documented; all secrets provisioned in Railway (staging) and AWS Secrets Manager (production) — nothing in environment variables committed to git
- [ ] Alembic initialized with initial schema migration; migration tested for reversibility
- [ ] Separate Supabase projects for staging and production
- [ ] Grafana Cloud workspace created; OpenTelemetry collector configured
- [ ] DLQ non-empty alert configured
- [ ] Task failure rate alert configured
- [ ] Rollback runbook documented in the team's incident response wiki

---

*This document should be reviewed by the development team before implementation begins. Findings rated Critical in this document must be resolved before the first deployment to any environment accessible from the internet. This document cross-references the Security Audit (02. Security/architecture-security-audit.md), Testing Strategy (03. Testing/testing-strategy.md), and Performance Analysis (01. Architecture/performance-scalability-analysis.md).*
