# Tech Debt: Analytics API Endpoints

**Severity:** Medium
**Status:** Deferred
**Created:** 2026-04-04

## Description

The analytics dashboard frontend (`invoice-web`) is fully implemented but uses hardcoded mock data instead of real backend endpoints. The following endpoints need to be created in `invoice-api`:

| Endpoint | Purpose | Status |
|----------|---------|--------|
| `GET /api/v1/analytics/kpis` | Aggregated KPIs for authenticated user | Mock |
| `GET /api/v1/analytics/timeseries` | Daily aggregation with preset date ranges | Mock |
| `GET /api/v1/analytics/documents` | Paginated document breakdown list | Mock |

## What was reverted

- `invoice-api/app/api/routes/analytics.py` — deleted (was created then reverted)
- `invoice-api/app/adapters/database.py` — analytics query methods reverted
- `invoice-api/app/api/main.py` — analytics router registration reverted
- `invoice-models/invoice_models/api/analytics.py` — Pydantic models deleted (reverted)

## What the frontend expects

File: `src/lib/api/analytics-client.ts`

Response types (see `src/lib/api/types.ts`):

```ts
interface AnalyticsKPIResponse {
  total_documents: number;
  success_count: number;
  failure_count: number;
  success_rate: number;
}

interface AnalyticsTimeSeriesPoint {
  date: string;    // YYYY-MM-DD
  total: number;
  success: number;
  failure: number;
}

interface AnalyticsTimeSeriesResponse {
  points: AnalyticsTimeSeriesPoint[];
  date_range: string;
  presets: Record<string, string>;
}

interface AnalyticsDocumentBreakdown {
  document_id: string;
  type: string;
  status: string;
  processing_time_ms: number;
  extraction_result: string;
  created_at: string;
}

interface AnalyticsDocumentBreakdownList {
  items: AnalyticsDocumentBreakdown[];
  total: number;
  page: number;
  page_size: number;
}
```

## SQL aggregations needed (reference)

### KPIs
```sql
SELECT
  COUNT(*) as total_documents,
  COUNT(*) FILTER (WHERE status = 'completed') as success_count,
  COUNT(*) FILTER (WHERE status IN ('failed', 'partial_failure')) as failure_count,
  ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'completed') / NULLIF(COUNT(*), 0), 1) as success_rate
FROM documents d
JOIN batches b ON d.batch_id = b.id
WHERE b.user_id = $1 AND d.created_at >= NOW() - INTERVAL '30 days'
```

### Time Series (daily)
```sql
SELECT
  d.created_at::date as date,
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE d.status = 'completed') as success,
  COUNT(*) FILTER (WHERE d.status IN ('failed', 'partial_failure')) as failure
FROM documents d
JOIN batches b ON d.batch_id = b.id
WHERE b.user_id = $1 AND d.created_at >= $2
GROUP BY d.created_at::date
ORDER BY d.created_at::date
```

### Document Breakdown (paginated)
```sql
SELECT d.*, b.created_at as batch_created_at
FROM documents d
JOIN batches b ON d.batch_id = b.id
WHERE b.user_id = $1
ORDER BY d.created_at DESC
LIMIT $2 OFFSET $3
```

## Security requirements

- All endpoints MUST require `get_current_user` dependency (JWT via Supabase JWKS)
- All SQL queries MUST filter by `user_id` — tenant isolation is critical
- Pagination must cap `page_size` (max 200) to prevent abuse

## Steps to resolve

1. Re-create Pydantic response models in `invoice-models/invoice_models/api/analytics.py`
2. Create `invoice-api/app/api/routes/analytics.py` with 3 endpoints
3. Add SQL aggregation methods to `invoice-api/app/adapters/database.py`
4. Register analytics router in `invoice-api/app/api/main.py`
5. Replace mock data in `src/lib/api/analytics-client.ts` with real `apiClient` calls
6. Run `pnpm generate:types` to sync OpenAPI types
7. Verify dashboard against real data

## Impact

- Dashboard shows realistic mock data — visually complete but not accurate
- No data loss or security risk from deferring
- Must resolve before any user relies on analytics for decision-making
