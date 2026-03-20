# Documentation Backlog

Derived from `reviews/final-report.md`. Items resolved during the initial documentation sprint are not listed here.

---

## Before First Deploy

### `06-security/06.2-lgpd.md`
Data protection strategy for LGPD-regulated data (CNPJ/CPF).
- Data classification (which fields are PII)
- Retention policy and automated deletion jobs
- Right-to-erasure endpoint (already supported by CASCADE deletes in schema)
- What must never appear in logs or error messages
- Encryption-at-rest verification (Supabase default)
- Review finding: C-08

### `07-operations/07.1-local-development.md`
How to run the full system locally.
- `docker-compose.yml` covering API + 4 worker types + Redis (with AUTH) + PostgreSQL
- Environment variable reference per service
- Alembic migration baseline setup
- Review finding: H-21

### `07-operations/07.2-observability.md`
Structured logging, metrics, and alerting.
- `structlog` field inventory per component (what every log line must include)
- OpenTelemetry metrics: DLQ depth, Gemini 429 rate, task duration histograms per worker type
- Alerting thresholds: DLQ depth > 0, Gemini 429 rate > 10%, no completions in 5min with non-empty queue
- Log destination (Grafana Cloud recommended)
- PII scrubbing rules (CNPJ/CPF must never appear in logs)
- Review finding: H-23

### `07-operations/07.3-dlq-runbook.md`
Operational procedure for DLQ manual review.
- How to inspect a failed task in the DLQ
- Decision criteria: retryable vs permanently failed
- Command to re-enqueue a specific task
- Bulk failure procedure for unrecoverable batches
- Review finding: M-02

---

## Before Team Expansion

### `08-domain/08.1-fiscal-documents.md`
Brazilian fiscal document domain context for developers without prior knowledge.
- What an NF-e (Nota Fiscal Eletrônica) is and its legal significance
- The three document types: NF-e XML, DANFE PDF, thermal receipt image
- Why XPath extraction instead of AI for XML (SEFAZ schema is fixed by law)
- Why contrast enhancement is necessary for thermal paper images
- SEFAZ XSD schema source and supported versions
- Review finding: M-03

---

## invoice-processors Specifications

~~Each branch needs its own spec when implementation begins.~~ **Done.** Specs created in `05-invoice-processors/`.

- `05-invoice-processors/05.0-overview.md` — package contract, branch summary, dependency direction
- `05-invoice-processors/05.1-xml-processor.md` — lxml security flags, XSD validation, XPath mapping, terminal failures
- `05-invoice-processors/05.2-pdf-processor.md` — dual-library approach, confidence threshold, cell extraction, table normalization gap
- `05-invoice-processors/05.3-image-processor.md` — Pillow preprocessing, Gemini structured output, response validation, extractor interface

### Remaining gap: PDF table normalization

`05.2-pdf-processor.md` §05.2.6 documents this gap explicitly. The current implementation returns raw column names from the PDF header. Canonical field mapping and value normalization (CFOP, quantities, BRL values) must be added before table data is written to `extracted_data`.

---

## Deferred Decisions

### M-05 — Frontend status update mechanism
Supabase Realtime (WebSocket) was identified as the natural fit over polling or SSE.
Needs a formal decision and documentation in the frontend spec before the frontend is built.

### Frontend specification
Next.js component responsibilities, upload flow, status display, Supabase Realtime integration.
Not blocking backend implementation.
