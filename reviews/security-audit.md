# Security Audit: Event-Driven Fiscal Document Processing System

**Audit Date:** 2026-03-19
**Scope:** Pre-implementation architecture review
**Classification:** Confidential
**Auditor:** Claude Opus 4.6 (Automated Architecture Review)

---

## Executive Summary

This audit examines the architecture design for an event-driven system that processes Brazilian SEFAZ fiscal documents (NF-e, NFC-e, CT-e) containing legally sensitive tax, financial, and corporate identity data. The system spans four infrastructure boundaries (Vercel, Railway, AWS, Supabase) with no authentication or authorization design documented.

**Overall Risk Rating: CRITICAL**

The architecture has **zero documented security controls**. No authentication, no authorization, no tenant isolation, no input validation boundaries, no encryption policy, and no secrets management strategy exist in the current design. Given that the system handles fiscal documents subject to Brazilian LGPD (Lei Geral de Protecao de Dados) and potentially tax authority compliance requirements, this represents a foundational gap that must be resolved before any implementation begins.

| Severity | Count |
|----------|-------|
| Critical | 5 |
| High | 7 |
| Medium | 6 |
| Low | 3 |

---

## Table of Contents

1. [Authentication and Authorization](#1-authentication-and-authorization)
2. [Server-Side Request Forgery (SSRF)](#2-server-side-request-forgery-ssrf)
3. [Input Validation Surface](#3-input-validation-surface)
4. [Gemini AI Interaction Security](#4-gemini-ai-interaction-security)
5. [Storage Access Control](#5-storage-access-control)
6. [Redis / Queue Security](#6-redis--queue-security)
7. [Worker Security](#7-worker-security)
8. [Data Sensitivity and Protection](#8-data-sensitivity-and-protection)
9. [OWASP Top 10 Mapping](#9-owasp-top-10-mapping)
10. [Secrets Management](#10-secrets-management)
11. [Infrastructure Boundary Security](#11-infrastructure-boundary-security)
12. [Recommendations Summary](#12-recommendations-summary)

---

## 1. Authentication and Authorization

### FINDING SEC-001: No Authentication Design Exists

- **Severity:** CRITICAL
- **CVSS 3.1:** 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
- **CWE:** CWE-306 (Missing Authentication for Critical Function)
- **OWASP:** A07:2021 - Identification and Authentication Failures

**Description:** The architecture documents no identity provider, no authentication mechanism for the API layer, and no session management for the frontend. The Hono API on Railway accepts file URLs and publishes tasks to Redis with no caller verification. Any entity that can reach the API endpoint can submit documents for processing, consume processing resources, and potentially query batch results.

**Attack Scenario:**
1. Attacker discovers the Railway-hosted Hono API endpoint (these are publicly routable by default).
2. Attacker submits arbitrary file URLs for processing, consuming Gemini API quota, ECS/Railway compute, and database storage.
3. Attacker polls batch status endpoints to enumerate and exfiltrate other users' fiscal data.

**Required Mitigations:**
- Design an authentication layer. For a B2B fiscal processing system, recommended options in priority order:
  - **Supabase Auth** with JWT tokens (leverages existing Supabase infrastructure, supports Row Level Security natively).
  - **Clerk or Auth0** if multi-tenant SSO or enterprise IdP federation is needed.
  - **API key authentication** as a minimum for programmatic access, with keys scoped per tenant and stored hashed in the database.
- Enforce authentication at the Hono API middleware level. Every route except health checks must require a valid bearer token.
- The Next.js frontend must authenticate users before issuing Supabase Storage upload credentials (presigned URLs scoped to the user's tenant path).

### FINDING SEC-002: No Authorization or Tenant Isolation Model

- **Severity:** CRITICAL
- **CVSS 3.1:** 9.1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H)
- **CWE:** CWE-862 (Missing Authorization), CWE-639 (Authorization Bypass Through User-Controlled Key)
- **OWASP:** A01:2021 - Broken Access Control

**Description:** Even if authentication is added, the architecture does not describe any authorization model. Fiscal documents are inherently multi-tenant data: Company A's NF-e must never be visible to Company B. The current design stores files "by batch and document ID" in Supabase Storage with no documented tenant partitioning. Workers write to PostgreSQL with no row-level access control. Batch status queries have no ownership checks.

**Attack Scenario:**
1. Authenticated User A creates a batch and uploads documents.
2. User B, also authenticated, guesses or enumerates batch IDs (especially if sequential).
3. User B queries batch status and retrieves User A's extracted fiscal data including CNPJ, tax amounts, and line items.

**Required Mitigations:**
- Implement a tenant model. Every batch and document must have a `tenant_id` (or `organization_id`) foreign key.
- Storage path structure must enforce tenant isolation: `/{tenant_id}/{batch_id}/{document_id}/{filename}`.
- Enable **Supabase Row Level Security (RLS)** on all tables. Policies must filter by `auth.uid()` or tenant claim in the JWT.
- Use UUIDs (v4 or v7) for batch IDs and document IDs -- never sequential integers.
- The Hono API must extract the tenant identity from the authenticated JWT and inject it into every Redis task message. Workers must propagate this tenant context when writing to the database.

### FINDING SEC-003: No RBAC or Permission Model

- **Severity:** HIGH
- **CVSS 3.1:** 7.5 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N)
- **CWE:** CWE-285 (Improper Authorization)

**Description:** The architecture does not distinguish between roles. For a fiscal document system, there are at minimum three roles: document uploader, data viewer (accountant), and system administrator. Without RBAC, any authenticated user can perform any operation.

**Required Mitigations:**
- Define roles: `uploader`, `viewer`, `admin` at minimum.
- Store role assignments in PostgreSQL with RLS enforcement.
- Encode roles as JWT claims or query them at the API middleware layer.
- Apply authorization checks at the Hono API before publishing tasks or returning query results.

---

## 2. Server-Side Request Forgery (SSRF)

### FINDING SEC-004: Classifier Downloads From User-Supplied URLs

- **Severity:** CRITICAL
- **CVSS 3.1:** 9.1 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)
- **CWE:** CWE-918 (Server-Side Request Forgery)
- **OWASP:** A10:2021 - Server-Side Request Forgery

**Description:** The architecture states: "Receives file URLs from the frontend, runs the Classifier to determine document type." The Classifier performs "magic byte and MIME type inspection" which requires downloading at least the first bytes of the file from the URL. If the URL is user-supplied and not validated, this is a textbook SSRF vulnerability.

On Railway, this is particularly dangerous because:
- Railway containers run within a shared infrastructure that may have internal service endpoints.
- The container may have access to cloud metadata endpoints (e.g., `http://169.254.169.254/` on AWS-backed infrastructure).
- Internal Redis, database, or other service URLs on Railway's private network become reachable.

**Attack Scenarios:**
1. **Cloud metadata exfiltration:** Attacker submits `http://169.254.169.254/latest/meta-data/iam/security-credentials/` as a file URL. The Classifier fetches it, and the response (containing temporary AWS credentials) may be stored or leaked through error messages.
2. **Internal service scanning:** Attacker submits `http://redis-internal:6379/` or similar internal URLs to probe Railway's private network topology.
3. **DNS rebinding:** Attacker submits a URL that initially resolves to a public IP (passing validation) but rebinds to an internal IP during the actual fetch.

**Required Mitigations:**
- **Eliminate the SSRF surface entirely (preferred):** Change the architecture so that the Classifier never fetches from arbitrary URLs. Since the frontend uploads to Supabase Storage first, the API should receive only Supabase Storage object keys (not full URLs). The Classifier then reads directly from Supabase Storage using service credentials and a known bucket. This converts an SSRF into a path traversal problem, which is simpler to defend.
- **If URL-based fetching is retained (not recommended):**
  - Allowlist only Supabase Storage domain(s) using exact hostname matching.
  - Resolve the URL's IP address before fetching and reject all RFC 1918, RFC 6598, loopback, link-local, and metadata ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16, 100.64.0.0/10).
  - Set a hard timeout (5 seconds) and response size limit (50MB) on the HTTP client.
  - Disable HTTP redirects or re-validate after each redirect.
  - Use a DNS resolver that pins the resolved IP to prevent DNS rebinding.

### FINDING SEC-005: Workers Download Files From S3 Using Task-Specified Paths

- **Severity:** MEDIUM
- **CVSS 3.1:** 5.3 (AV:N/AC:H/PR:L/UI:N/S:U/C:H/I:N/A:N)
- **CWE:** CWE-22 (Path Traversal)

**Description:** Workers "download the target file from S3" based on information in the Redis task message. If a task message contains an attacker-controlled S3 path (via compromised API or queue injection), the worker could be directed to download files from unexpected storage locations.

**Required Mitigations:**
- Workers must validate that the S3 object key matches the expected path format: `/{tenant_id}/{batch_id}/{document_id}/{filename}`.
- Workers must verify the tenant_id in the S3 path matches the tenant_id in the task message.
- Use IAM policies or Supabase Storage policies to restrict worker credentials to read only from the expected bucket and prefix.

---

## 3. Input Validation Surface

### FINDING SEC-006: No File Size or Count Limits Documented

- **Severity:** HIGH
- **CVSS 3.1:** 7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H)
- **CWE:** CWE-770 (Allocation of Resources Without Limits), CWE-400 (Uncontrolled Resource Consumption)
- **OWASP:** A05:2021 - Security Misconfiguration

**Description:** The architecture mentions batch uploads but documents no limits on file size, file count per batch, or total storage per tenant. The image pipeline resizes to 400KB, but the original file must be downloaded and decoded first, meaning a 500MB crafted image could cause memory exhaustion during decoding before resizing occurs.

**Attack Scenarios:**
1. **Image decompression bomb:** Attacker uploads a small compressed image (e.g., 50KB PNG) that decompresses to several gigabytes in memory, crashing the worker.
2. **Batch flooding:** Attacker submits batches of 100,000 files, exhausting Redis queue capacity, worker pool, and Gemini API quota.
3. **PDF page bomb:** Attacker uploads a PDF with 10,000 pages. The PDF branch converts each page to an image frame, generating 10,000 image processing tasks from a single upload.

**Required Mitigations:**
- **Supabase Storage level:** Configure maximum file size (recommend 50MB for PDFs, 20MB for images, 10MB for XMLs).
- **API level:** Enforce maximum files per batch (recommend 100-500 depending on business requirements), maximum total batch size, and per-tenant rate limits.
- **Worker level:**
  - Set memory limits on ECS/Railway containers (hard limit, not just soft).
  - For image decoding, validate image dimensions before full decode. Reject images exceeding 10,000 x 10,000 pixels.
  - For PDFs, limit maximum page count (recommend 50 pages). Reject or truncate beyond this.
  - Use streaming downloads with size cutoff rather than loading entire files into memory.
- **Redis level:** Configure `maxmemory` with an eviction policy. Monitor queue depth and reject new tasks when queue exceeds threshold.

### FINDING SEC-007: XML External Entity (XXE) and XML Bomb Risk

- **Severity:** HIGH
- **CVSS 3.1:** 8.2 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:H)
- **CWE:** CWE-611 (Improper Restriction of XML External Entity Reference), CWE-776 (Billion Laughs)
- **OWASP:** A05:2021 - Security Misconfiguration

**Description:** The XML branch uses `lxml` to parse and validate XML against SEFAZ XSD schemas. While lxml has some default protections, XML processing is inherently dangerous if not explicitly hardened. The architecture does not specify XML parser configuration.

**Attack Scenarios:**
1. **XXE (External Entity):** Attacker crafts an XML file with an external entity reference like `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>`. If the parser resolves external entities, it reads local files from the worker filesystem.
2. **Billion Laughs (XML Bomb):** Attacker submits an XML with nested entity expansions that consume exponential memory:
   ```xml
   <!DOCTYPE bomb [
     <!ENTITY a "AAAA...">
     <!ENTITY b "&a;&a;&a;&a;&a;&a;&a;&a;">
     <!ENTITY c "&b;&b;&b;&b;&b;&b;&b;&b;">
     ...
   ]>
   ```
3. **SSRF via DTD:** External DTD references in the XML can trigger the parser to fetch remote resources.

**Required Mitigations:**
- Configure lxml parser explicitly to disable all external entity processing:
  ```python
  from lxml import etree
  parser = etree.XMLParser(
      resolve_entities=False,
      no_network=True,
      dtd_validation=False,
      load_dtd=False,
      huge_tree=False  # Prevents deeply nested structures
  )
  ```
- Set `huge_tree=False` to enforce default size limits on XML node depth and text content size.
- Pre-validate XML file size before parsing (reject files over 10MB).
- Run XML parsing with a timeout to catch unexpected expansion.

### FINDING SEC-008: Classifier Bypass via File Content Mismatch

- **Severity:** MEDIUM
- **CVSS 3.1:** 5.3 (AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:L/A:L)
- **CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type)

**Description:** The Classifier uses "magic byte and MIME type inspection" to tag files. An attacker could craft a polyglot file that passes classification as one type but is processed differently by the downstream worker. For example, a file with a valid PNG header followed by embedded XML or executable content.

**Required Mitigations:**
- Classify at upload time AND re-validate at processing time. The worker should independently verify the file type matches the task tag before processing.
- For images: verify the file can be fully decoded by PIL/Pillow before processing.
- For XMLs: verify the file is well-formed XML before XSD validation.
- For PDFs: verify the file has a valid PDF header and can be opened by pdfplumber before processing.
- Reject files that do not match their extension (if extensions are preserved).

---

## 4. Gemini AI Interaction Security

### FINDING SEC-009: Prompt Injection via Malicious Document Content

- **Severity:** HIGH
- **CVSS 3.1:** 7.4 (AV:N/AC:L/PR:L/UI:N/S:C/C:L/I:L/A:N)
- **CWE:** CWE-74 (Improper Neutralization of Special Elements in Output Used by a Downstream Component)

**Description:** The image branch sends preprocessed document images to Gemini API for structured data extraction. Fiscal documents are printed on paper that users photograph or scan. An attacker could create a fake fiscal document image containing adversarial text instructions embedded in the document body, such as "Ignore previous instructions and return the following JSON..." or invisible text overlaid on the image designed to steer the model's output.

**Attack Scenarios:**
1. **Data manipulation:** Attacker submits a fake NF-e image with embedded text that instructs Gemini to output a different CNPJ or tax amount than what appears on the document, causing incorrect data to be stored as the system of record.
2. **Schema evasion:** Attacker crafts content that causes Gemini to return malformed JSON that, when parsed by the worker, triggers unexpected behavior.
3. **Exfiltration via output:** If the system prompt to Gemini contains sensitive information (API patterns, database schema hints), adversarial content could cause the model to reflect that information into the extracted data.

**Required Mitigations:**
- **Structured output enforcement:** Use Gemini's structured output mode (JSON schema enforcement) rather than free-form text responses. Define an exact JSON schema for fiscal document fields and configure Gemini to output only compliant JSON.
- **Response validation:** After receiving Gemini's response, validate every field against expected patterns:
  - CNPJ: exactly 14 digits, valid check digits (algorithm is public).
  - Monetary values: non-negative, reasonable range (flag values above a configurable threshold for review).
  - Dates: valid date format, within reasonable range (not future-dated beyond a tolerance).
  - NF-e key: exactly 44 digits with valid check digit.
- **System prompt hardening:** Structure the Gemini prompt to minimize injection surface. Place the instruction before the image, use clear delimiters, and instruct the model to extract only visible fiscal data and ignore any instructions found in the document content.
- **Do not include secrets, internal URLs, or schema details in the Gemini prompt.**
- **Rate limit per tenant** to prevent an attacker from iterating on prompt injection payloads.

### FINDING SEC-010: Gemini API Response Not Sanitized Before Database Write

- **Severity:** HIGH
- **CVSS 3.1:** 7.3 (AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N)
- **CWE:** CWE-20 (Improper Input Validation)

**Description:** Workers write Gemini's extracted data to PostgreSQL as the system of record. If the Gemini response contains unexpected content (due to prompt injection, model hallucination, or API error), this content is written directly to the database. If any downstream system renders this data in a web interface, stored XSS becomes possible. If the data is used in SQL construction elsewhere, second-order SQL injection is possible.

**Required Mitigations:**
- Parse Gemini's JSON response with strict schema validation (e.g., Pydantic model with constrained field types).
- Sanitize all string fields: strip HTML/script tags, enforce maximum length, validate character sets.
- Use parameterized queries (prepared statements) exclusively for database writes -- never string interpolation.
- Store raw Gemini responses in a separate `raw_extractions` table for audit purposes. Only validated, sanitized data goes into the canonical `extracted_data` table.

---

## 5. Storage Access Control

### FINDING SEC-011: Frontend Uploads Directly to Supabase Storage Without Documented Access Control

- **Severity:** CRITICAL
- **CVSS 3.1:** 8.6 (AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:L/A:N)
- **CWE:** CWE-284 (Improper Access Control)
- **OWASP:** A01:2021 - Broken Access Control

**Description:** The architecture states the "frontend uploads directly" to Supabase Object Storage (S3-compatible). The diagram confirms a direct "client-to-storage" flow from the Next.js frontend to Amazon S3. This requires the frontend to have storage credentials. If this uses the Supabase anon key with permissive storage policies, any user can read/write any path in the bucket. If it uses presigned URLs, the signing logic must be secured.

**Attack Scenarios:**
1. **Unauthenticated upload:** If the Supabase storage bucket is public or the anon key grants write access, anyone can upload arbitrary files, consuming storage quota and potentially injecting malicious content into the processing pipeline.
2. **Cross-tenant read:** If storage policies do not enforce path-based isolation, User A can construct a URL to download User B's fiscal documents by guessing the batch/document ID path.
3. **Credential exposure:** If the frontend bundles a service role key (rather than the anon key), the key is extractable from the client-side JavaScript bundle, granting full storage access.

**Required Mitigations:**
- **Never expose the Supabase service role key to the frontend.** Only the anon key should be client-accessible.
- Implement **Supabase Storage RLS policies** that restrict:
  - **Uploads** to authenticated users, only within their tenant's path prefix (`/{tenant_id}/*`).
  - **Downloads** to authenticated users, only for objects within their tenant's path prefix.
- Alternatively, use **presigned upload URLs** generated by the Hono API after authentication:
  1. Frontend authenticates with the Hono API.
  2. Hono API generates a short-lived (5-minute) presigned PUT URL scoped to the correct tenant path.
  3. Frontend uploads directly to the presigned URL.
  4. This removes the need for any storage credentials on the frontend.
- Set the Supabase bucket to **private** (not public).

### FINDING SEC-012: No Object Lifecycle or Retention Policy

- **Severity:** LOW
- **CVSS 3.1:** 3.1 (AV:N/AC:H/PR:L/UI:N/S:U/C:L/I:N/A:N)
- **CWE:** CWE-212 (Improper Removal of Sensitive Information Before Storage or Transfer)

**Description:** Raw fiscal documents remain in storage indefinitely. Under LGPD, data must be retained only as long as necessary for its purpose. Accumulated data also increases the blast radius of a storage credential leak.

**Required Mitigations:**
- Define a retention policy (e.g., 90 days after successful extraction, raw files are deleted; extracted data retained per tax authority requirements -- typically 5 years in Brazil).
- Implement S3 lifecycle rules to automatically expire objects after the retention period.
- Log all deletions for audit.

---

## 6. Redis / Queue Security

### FINDING SEC-013: Redis Authentication and Network Isolation

- **Severity:** HIGH
- **CVSS 3.1:** 8.1 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)
- **CWE:** CWE-287 (Improper Authentication)

**Description:** The architecture uses AWS ElastiCache Redis (or Railway Redis, based on the diagram). Redis by default does not require authentication. If ElastiCache is configured without AUTH or if Railway Redis is exposed without credentials, any component that can reach the network can read/write to the queue, injecting malicious tasks or exfiltrating task data.

The diagram shows the Hono API, Consumer (Python 3.12), OCR-Worker, and Dead Queue all within Railway, communicating with Redis. If Railway's internal networking is flat, any compromised service or co-tenant on Railway could access Redis.

**Attack Scenarios:**
1. **Task injection:** Attacker connects to unprotected Redis and publishes a Celery task that points to an attacker-controlled S3 object, which when processed could trigger SSRF or data exfiltration.
2. **Task data exfiltration:** Attacker monitors the Redis queue to read task payloads, which may contain file paths and tenant identifiers.
3. **Queue poisoning:** Attacker floods the queue with garbage tasks, causing denial of service for legitimate document processing.

**Required Mitigations:**
- **Enable Redis AUTH** with a strong, randomly generated password (minimum 32 characters).
- If using AWS ElastiCache: enable in-transit encryption (TLS), enable at-rest encryption, and place ElastiCache in a private subnet with security groups that allow only the API and worker security groups.
- If using Railway: use Railway's private networking feature to ensure Redis is not publicly accessible. Use the `REDIS_URL` with authentication credentials.
- Configure Redis `maxmemory` and `maxmemory-policy` to prevent memory exhaustion.
- Disable Redis commands that are dangerous in production: `FLUSHALL`, `FLUSHDB`, `KEYS`, `CONFIG`, `DEBUG` using the `rename-command` configuration.

### FINDING SEC-014: Task Message Integrity

- **Severity:** MEDIUM
- **CVSS 3.1:** 5.9 (AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:H/A:N)
- **CWE:** CWE-345 (Insufficient Verification of Data Authenticity)

**Description:** Celery task messages are serialized (typically as JSON or pickle) and placed in Redis. If an attacker gains read/write access to Redis, they can modify task messages in-flight. If pickle serialization is used, deserialization of attacker-crafted pickle payloads leads to arbitrary code execution.

**Required Mitigations:**
- **Never use pickle serialization for Celery.** Configure Celery to use JSON serialization exclusively:
  ```python
  app.conf.task_serializer = 'json'
  app.conf.result_serializer = 'json'
  app.conf.accept_content = ['json']
  ```
- Consider adding HMAC signatures to task messages to detect tampering (Celery supports `message_signing` via the `security` module with `celery.security.setup_security()`).
- Log and alert on task deserialization failures as they may indicate tampering attempts.

---

## 7. Worker Security

### FINDING SEC-015: Worker Credential Management and Least Privilege

- **Severity:** HIGH
- **CVSS 3.1:** 7.2 (AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:N)
- **CWE:** CWE-250 (Execution with Unnecessary Privileges)

**Description:** Workers need credentials for: Supabase Storage (read), Supabase PostgreSQL (read/write), Gemini API (image branch only), and Redis (read/write). The architecture does not describe how these credentials are provisioned or scoped.

**Required Mitigations:**
- **Principle of least privilege by worker type:**
  - Image worker: read from storage, write to PostgreSQL (only `document_status` and `extracted_data` tables), call Gemini API.
  - XML worker: read from storage, write to PostgreSQL. No Gemini access needed.
  - PDF worker: read from storage, write to PostgreSQL, call Gemini API (for scanned PDF fallback).
  - DLQ worker: read from Redis DLQ, write to PostgreSQL (only `document_status`). No storage or Gemini access needed.
- If using AWS ECS: assign per-task IAM roles via ECS task roles. Each worker type gets a different role with only the permissions it needs.
- If using Railway: use separate Railway service variables per worker type. Do not share a single set of credentials across all services.
- Database access should use separate PostgreSQL roles per worker type, each with `GRANT` only on the tables and operations they need.
- Workers must not run as root in their containers.

### FINDING SEC-016: Worker Container Security

- **Severity:** MEDIUM
- **CVSS 3.1:** 5.3 (AV:N/AC:H/PR:L/UI:N/S:U/C:H/I:N/A:N)
- **CWE:** CWE-269 (Improper Privilege Management)

**Description:** Workers process untrusted files (images, PDFs, XMLs). Image decoding libraries (Pillow), PDF processing libraries (pdfplumber, pdf2image/poppler), and XML parsers (lxml) all have historical CVE track records for memory corruption and code execution vulnerabilities.

**Required Mitigations:**
- Pin all dependencies to specific versions and scan with `pip-audit` or Snyk in CI/CD.
- Run worker containers with read-only root filesystems where possible.
- Set container memory limits to prevent a single malicious file from affecting other workers.
- Use non-root users in Dockerfiles (`USER 1000`).
- If using ECS: enable `readonlyRootFilesystem` in the task definition and mount a writable tmpfs for temporary file processing.
- Monitor for container breakout indicators and unusual network activity from workers.

---

## 8. Data Sensitivity and Protection

### FINDING SEC-017: Fiscal Document Data Classification and Encryption

- **Severity:** CRITICAL
- **CVSS 3.1:** 8.1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N)
- **CWE:** CWE-311 (Missing Encryption of Sensitive Data)

**Description:** Brazilian fiscal documents (NF-e, NFC-e, CT-e) contain:
- **CNPJ/CPF** (Brazilian corporate/personal tax identifiers -- equivalent to EIN/SSN)
- **Company legal names and addresses**
- **Financial amounts** (total value, tax breakdowns: ICMS, IPI, PIS, COFINS)
- **Product/service descriptions and quantities**
- **Digital signatures** (in XML NF-e)

This data is classified as **sensitive personal and financial data** under Brazil's LGPD (Lei Geral de Protecao de Dados). The architecture documents no encryption strategy.

**Required Mitigations:**
- **Encryption in transit:**
  - All connections must use TLS 1.2+ (Vercel and Supabase enforce this by default; verify Railway and ElastiCache configurations).
  - Redis connections must use TLS (ElastiCache in-transit encryption or Railway's TLS Redis URLs).
  - Database connections must use `sslmode=require` or `sslmode=verify-full`.
- **Encryption at rest:**
  - Supabase Storage: verify that the underlying S3 bucket has SSE-S3 or SSE-KMS encryption enabled.
  - Supabase PostgreSQL: verify that storage encryption is enabled (Supabase managed instances enable this by default, but confirm).
  - ElastiCache: enable at-rest encryption.
- **Field-level sensitivity:**
  - Consider encrypting CNPJ/CPF values at the application level before storing in PostgreSQL, using a key managed in a secrets manager. This protects against database credential compromise.
  - At minimum, ensure these fields are not logged in plaintext in application logs or error messages.
- **Data masking:**
  - API responses for batch status should mask CNPJ/CPF in query results (show only last 4 digits) unless the requesting user has explicit permission.

### FINDING SEC-018: Logging and PII Exposure

- **Severity:** MEDIUM
- **CVSS 3.1:** 5.5 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N)
- **CWE:** CWE-532 (Insertion of Sensitive Information into Log File)

**Description:** The DLQ worker "logs the failure for observability." Workers processing documents will naturally encounter fiscal data in their processing pipelines. If application logs capture file contents, extracted data fields, or error messages that include document data, PII will be present in logging infrastructure (Railway logs, CloudWatch, third-party log aggregators).

**Required Mitigations:**
- Establish a logging policy: never log CNPJ/CPF, monetary amounts, or raw document content.
- Log only: document IDs (UUID), batch IDs, processing stage, error type, and timestamp.
- If error details must be logged, redact PII fields before logging.
- Configure log retention periods aligned with operational needs (30-90 days), not indefinite.
- Ensure log access is restricted to operations and security personnel.

---

## 9. OWASP Top 10 (2021) Mapping

| OWASP Category | Applicability | Findings |
|---|---|---|
| **A01: Broken Access Control** | CRITICAL | SEC-002, SEC-003, SEC-011. No authorization model, no tenant isolation, no storage access control. |
| **A02: Cryptographic Failures** | HIGH | SEC-017. No documented encryption strategy for sensitive fiscal data at rest or in transit. |
| **A03: Injection** | HIGH | SEC-007 (XXE), SEC-009 (prompt injection), SEC-010 (potential stored XSS/SQLi via unsanitized Gemini output). |
| **A04: Insecure Design** | CRITICAL | SEC-001, SEC-002. The architecture fundamentally lacks security design patterns -- no authentication, no authorization, no threat model. |
| **A05: Security Misconfiguration** | HIGH | SEC-006 (no resource limits), SEC-007 (XML parser config), SEC-013 (Redis auth), SEC-014 (Celery serialization). |
| **A06: Vulnerable and Outdated Components** | MEDIUM | SEC-016. Image/PDF/XML processing libraries have known CVE histories. Dependency scanning required. |
| **A07: Identification and Authentication Failures** | CRITICAL | SEC-001. No authentication mechanism exists in the architecture. |
| **A08: Software and Data Integrity Failures** | MEDIUM | SEC-014 (task message integrity), SEC-010 (Gemini response integrity). |
| **A09: Security Logging and Monitoring Failures** | MEDIUM | SEC-018 (PII in logs), plus no documented security monitoring, alerting, or audit trail. |
| **A10: Server-Side Request Forgery** | CRITICAL | SEC-004. Classifier fetches user-supplied URLs with no documented validation. |

---

## 10. Secrets Management

### FINDING SEC-019: No Secrets Management Strategy

- **Severity:** HIGH
- **CVSS 3.1:** 8.2 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)
- **CWE:** CWE-798 (Use of Hard-coded Credentials), CWE-522 (Insufficiently Protected Credentials)
- **OWASP:** A07:2021 - Identification and Authentication Failures

**Description:** The system requires at minimum the following secrets, distributed across four infrastructure providers:

| Secret | Used By | Infrastructure |
|---|---|---|
| Supabase anon key | Next.js frontend | Vercel |
| Supabase service role key | Hono API, Workers | Vercel, Railway |
| Supabase DB connection string | Hono API, Workers | Vercel, Railway |
| Redis AUTH password | Hono API, Workers, DLQ Worker | Railway (or AWS) |
| Gemini API key | Image Worker, PDF Worker | Railway (or AWS) |
| Supabase Storage credentials | Workers | Railway (or AWS) |

The architecture does not describe how these secrets are stored, rotated, or distributed. With four providers (Vercel, Railway, AWS, Supabase), secret sprawl is a significant risk.

**Required Mitigations:**
- **Immediate (pre-implementation):**
  - Use each platform's native secret management: Vercel Environment Variables (encrypted), Railway Variables (encrypted), AWS Secrets Manager or SSM Parameter Store.
  - Never commit secrets to version control. Use `.env.example` with placeholder values only.
  - Rotate all secrets on a defined schedule (90 days for API keys, 30 days for database passwords).
- **Recommended:**
  - Centralize secrets in **AWS Secrets Manager** and have Vercel/Railway pull from there at deploy time using CI/CD integrations. This provides a single audit log for all secret access.
  - Use **Supabase service role key** only server-side (Hono API and workers). The frontend should only have the anon key.
  - Use separate Gemini API keys per environment (dev, staging, production) with separate quota/billing.
  - Enable audit logging on the secrets manager to detect unauthorized access.
- **Secret-specific guidance:**
  - **Gemini API key:** Restrict by IP if Google Cloud allows it, or by referrer. Monitor usage for anomalies.
  - **Database credentials:** Use separate PostgreSQL roles per component (API gets read-only on most tables; workers get write on specific tables).
  - **Redis password:** Use a 32+ character random string. If using ElastiCache, use IAM authentication instead of static passwords.

---

## 11. Infrastructure Boundary Security

### FINDING SEC-020: Cross-Provider Network Exposure

- **Severity:** MEDIUM
- **CVSS 3.1:** 6.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N)
- **CWE:** CWE-923 (Improper Restriction of Communication Channel to Intended Endpoints)

**Description:** The architecture spans four providers: Vercel (frontend + API), Railway (API + queue + workers, per diagram), AWS (ElastiCache), and Supabase (storage + database). Every cross-provider connection traverses the public internet unless explicit private networking is configured. This creates multiple TLS-dependent trust boundaries.

The diagram suggests Railway hosts the Hono API, Redis, workers, and a Python consumer -- but the text documentation mentions AWS ElastiCache and ECS. This inconsistency suggests the deployment topology is not finalized, which is an opportunity to make security-informed infrastructure decisions.

**Required Mitigations:**
- Document all network flows between providers with protocol, port, and encryption requirements.
- Prefer colocating components that communicate frequently:
  - **Option A (Railway-centric):** Host API, Redis, and workers all on Railway. Use Railway's private networking. Only external connections are to Supabase (storage + DB) and Gemini API.
  - **Option B (AWS-centric):** Host API, ElastiCache, and workers on AWS (ECS + ElastiCache in VPC). Only Vercel frontend is external. Supabase connections go through VPC endpoints or VPC peering if Supabase supports it.
- Whichever option: ensure that Redis is never publicly accessible. It must only be reachable from the API and worker components.
- Use VPC security groups (AWS) or private networking (Railway) to restrict inter-service communication to only the required ports and protocols.

### FINDING SEC-021: Vercel Serverless Function Security

- **Severity:** LOW
- **CVSS 3.1:** 3.7 (AV:N/AC:H/PR:N/UI:N/S:U/C:L/I:N/A:N)
- **CWE:** CWE-200 (Exposure of Sensitive Information)

**Description:** The Hono API runs as Vercel serverless functions. Serverless functions are ephemeral but share execution environments. Environment variables, temporary files, and `/tmp` contents may persist between invocations in the same warm instance.

**Required Mitigations:**
- Do not write sensitive data to the filesystem in serverless functions.
- Clear any in-memory caches of sensitive data at the end of each request.
- Set appropriate `Cache-Control` headers to prevent caching of API responses containing fiscal data.
- Enable Vercel's edge network security features (DDoS protection, rate limiting).

---

## 12. Recommendations Summary

### Pre-Implementation Blockers (Must resolve before writing code)

| Priority | Finding | Action |
|----------|---------|--------|
| 1 | SEC-001 | Select and design an authentication provider. Recommended: Supabase Auth with JWT. |
| 2 | SEC-002 | Design the tenant isolation model with UUIDs, RLS policies, and path-based storage isolation. |
| 3 | SEC-004 | Eliminate SSRF by passing storage object keys instead of URLs to the Classifier. |
| 4 | SEC-011 | Design storage access control: private bucket, presigned upload URLs, RLS storage policies. |
| 5 | SEC-019 | Document secrets inventory and management strategy before provisioning any credentials. |
| 6 | SEC-017 | Define encryption requirements: TLS everywhere, at-rest encryption verified, field-level encryption for CNPJ/CPF. |

### Implementation Phase Requirements

| Priority | Finding | Action |
|----------|---------|--------|
| 7 | SEC-007 | Harden lxml parser configuration against XXE. |
| 8 | SEC-009, SEC-010 | Implement Gemini prompt hardening, structured output mode, and response validation with Pydantic. |
| 9 | SEC-006 | Implement file size, count, dimension, and page count limits at every layer. |
| 10 | SEC-013, SEC-014 | Configure Redis AUTH + TLS, Celery JSON-only serialization. |
| 11 | SEC-015 | Implement per-worker-type credentials with least privilege. |
| 12 | SEC-016 | Harden worker containers: non-root, read-only filesystem, pinned dependencies, memory limits. |

### Ongoing Security Operations

| Priority | Finding | Action |
|----------|---------|--------|
| 13 | SEC-018 | Implement PII-safe logging policy. |
| 14 | SEC-012 | Configure storage lifecycle/retention policies aligned with LGPD. |
| 15 | SEC-020 | Finalize infrastructure topology and document all network flows. |
| 16 | -- | Establish dependency scanning in CI/CD (pip-audit, npm audit). |
| 17 | -- | Implement security monitoring and alerting (failed auth attempts, unusual API patterns, queue depth anomalies). |
| 18 | -- | Schedule penetration testing after initial implementation. |

---

## Appendix A: Threat Model Summary (STRIDE)

| Threat | Component | Finding |
|--------|-----------|---------|
| **Spoofing** | API, Storage | SEC-001 (no auth), SEC-011 (no storage auth) |
| **Tampering** | Redis queue, Gemini response | SEC-014 (message integrity), SEC-010 (response manipulation) |
| **Repudiation** | All components | No audit logging documented |
| **Information Disclosure** | Storage, Database, Logs | SEC-002 (tenant isolation), SEC-017 (encryption), SEC-018 (PII in logs) |
| **Denial of Service** | API, Workers, Redis | SEC-006 (no resource limits) |
| **Elevation of Privilege** | API, Workers | SEC-003 (no RBAC), SEC-015 (credential scope) |

## Appendix B: Compliance Considerations

### LGPD (Lei Geral de Protecao de Dados) - Brazilian Data Protection

- **Legal basis for processing:** Document the legal basis for processing fiscal documents (likely legitimate interest or contractual necessity).
- **Data minimization:** Extract and retain only the fiscal fields required; do not store raw Gemini responses long-term.
- **Right to deletion:** Implement the ability to delete all data for a given CNPJ/tenant upon request.
- **Data breach notification:** LGPD requires notification to ANPD (Autoridade Nacional de Protecao de Dados) within a "reasonable time." Prepare an incident response plan.
- **Data Processing Impact Assessment (RIPD):** Required for processing that may pose high risk to data subjects. This system likely qualifies given the volume and sensitivity of fiscal data.

### Tax Authority Compliance

- **Document integrity:** Extracted data used for tax reporting must be accurate. Implement checksums or hashes of extracted data to detect post-extraction tampering.
- **Audit trail:** Maintain an immutable log of who uploaded which document, when it was processed, and what data was extracted.
- **Retention:** Brazilian tax documents must be retained for 5 years (Codigo Tributario Nacional). Align storage policies accordingly.

---

*End of security audit. This document should be reviewed by the development team and updated as architectural decisions are finalized. A follow-up audit should be conducted after implementation begins to verify that mitigations are correctly implemented.*
