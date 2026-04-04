# Upload Pipeline Diagnosis — Frontend vs Backend

**Date:** 2026-04-04
**Source:** `D:\Contabiliz\invoice-infra\log2.md`
**Severity:** Medium — documents get stuck in `processing` state

## Summary

The frontend upload flow works correctly (presign → upload → confirm → polling), but backend workers fail silently, leaving documents stuck in `processing` state indefinitely. The frontend has no mechanism to detect stale processing state.

## Problems Found

### 🔴 Problem 1: Event loop crash in worker-promotion-3
```
RuntimeError: Event loop is closed
  File anyio/_backends/_asyncio.py, line 1344, in aclose
  File httpcore/_async/http11.py, line 258, in aclose
```
httpx client is used after event loop is closed in `invoice-processors` promotion module. This corrupts the promotion step for documents processed after the crash.

**Impact:** Documents skip promotion entirely after crash — status never reaches final state.

### 🟡 Problem 2: Missed heartbeats — workers silently die
```
missed heartbeat from celery@04a28cee43d8 (14x between 18:14-18:26)
missed heartbeat from celery@b7e11a991c26
missed heartbeat from celery@e94bb90fe6ba
missed heartbeat from celery@d3462dc3a260
```
Workers (xml, pdf, img, promotion, dlq) all lose heartbeats. Tasks remain queued but have no active worker.

**Impact:** User sees `processing` status forever — no timeout, no retry, no error notification.

### 🟡 Problem 3: Duplicate promotion batch after XML completion
```
celery-beat: promote-documents task running every ~65s
18:15:05 - promotion.batch_done promoted=16 batch_id=6c2aff23
18:16:10 - promotion.batch_start count=1
18:21:48 - promote-documents sent again
18:22:53 - promote-documents sent again
18:23:55 - promote-documents sent again
18:24:02 - promotion.event_loop_closed (crash)
18:24:56 - promotion.document_skipped (missing_fields=['nome_emitente'])
18:26:00 - promotion.batch_start count=1
```
`promote_documents` task triggered repeatedly by celery-beat. Same documents get promoted multiple times, and after the crash, promotion skips documents with missing_fields.

**Impact:** Duplicate processing, wasted resources, incomplete promotion after crash.

### 🟠 Problem 4: Warm shutdowns on active workers
```
worker-promotion-1: warm shutdown
worker-dlq-1: warm shutdown
worker-img-1: warm shutdown
worker-pdf-1: warm shutdown
```
All workers shutting down gracefully during active processing. Could be Docker restart, aggressive timeout config, or manual intervention.

**Impact:** Tasks interrupted mid-execution, potentially leaving documents in inconsistent state.

## Frontend Gap

The frontend polls `/batches/{id}` correctly but has no logic to detect:
1. **Stale processing** — batch stuck in `processing` for > 30min with no status change
2. **Worker death** — no way to know backend workers are dead
3. **Promotion failure** — processing completes but promotion fails, leaving partial results

## Recommended Fixes

### Backend (invoice-pipeline)
1. Fix event loop lifecycle — don't close event loop while httpx client is still active
2. Add worker health monitoring — detect dead workers and restart
3. Fix celery-beat schedule — promotion should trigger per-batch, not on fixed interval
4. Fix warm shutdown — investigate Docker/service restart config

### Frontend (invoice-web)
1. Add stale state detection — if batch.status === 'processing' and no update for N minutes, show warning
2. Add retry button for stuck batches
3. Show more granular error messages when promotion fails

## Related
- Analytics API tech debt: `tech-debt/analytics-api.md`
- Batch list API tech debt: `tech-debt/batch-list-api.md`
