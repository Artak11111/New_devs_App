# Assignment fixes only (local test scope)

This file lists **only fixes that directly address the three issues described in `ASSIGNMENT.md`**:

- **Client A**: March totals mismatch
- **Client B**: cross-tenant revenue shown on refresh (privacy)
- **Finance**: totals off by a few cents

Local environment / build / deploy / external-connection concerns are intentionally ignored.

---

## 1) Privacy: cross-tenant revenue shown on refresh

- **Fix cache key tenant isolation**
  - **Where**: `backend/app/services/cache.py`
  - **Current**: `cache_key = f"revenue:{property_id}"`
  - **Change**: include tenant in the key: `revenue:{tenant_id}:{property_id}`
  - **Why**: seed data has the same `property_id` across tenants (`prop-001` exists for both `tenant-a` and `tenant-b`), so current key is guaranteed to collide.

---

## 2) March totals mismatch (time zones)

- **Make monthly windows timezone-aware using property timezone**
  - **Where**: revenue-by-month logic (currently `backend/app/services/reservations.py::calculate_monthly_revenue`)
  - **Why**: `properties.timezone` exists in schema (`database/schema.sql`) and is populated (`database/seed.sql`), and seed includes a boundary case near month start (`2024-02-29 23:30:00+00`).
  - **Change**: compute \[month_start, month_end) in the **property timezone**, convert to UTC, then filter `reservations.check_in_date` against that range.

---

## 3) Totals off by a few cents (precision)

- **It shouldn't convert money to float in the API**
  - **Where**: `backend/app/api/v1/dashboard.py`
  - **Current**: `float(revenue_data['total'])`
  - **Change**: return money as a **string decimal** (or integer minor units). Prefer string here because DB uses `NUMERIC(10, 3)` (sub-cent precision) in `database/schema.sql`.

