# Retail WW — Upsell Performance Dashboard

Full-stack Excel-to-PostgreSQL dashboard with Executive Overview, interactive filters, drill-down, and refresh uploads. Uses **FastAPI + SQLAlchemy + PostgreSQL + Chart.js**, Docker, GitHub Actions, and Render Blueprint.

## Quick start (Docker)

```bash
cp .env.example .env
# Edit .env: set ADMIN_TOKEN and POSTGRES_PASSWORD to strong secrets
docker compose up --build -d
```
Open **http://localhost:8000**. On first startup the application imports `data/seed.xlsx` into PostgreSQL. Upload a new full `.xlsx` workbook via **Upload** and enter the `ADMIN_TOKEN` from `.env`.

## Deploy to Render from GitHub

1. Upload the folder contents to a **private GitHub repository**. The provided seed file includes sales and employee information.
2. In Render choose **New → Blueprint**, connect your GitHub repository, and apply `render.yaml`.
3. The Blueprint creates a PostgreSQL database and Docker Web Service. `ADMIN_TOKEN` is automatically generated as a Render environment variable; open service settings to view/copy it for upload.
4. Visit the Render service URL. Initial database import runs once on a new database; later deployments retain existing data in PostgreSQL.
5. Use the Upload page with the original workbook format to refresh the entire snapshot.

Render paid resources are used in the example (`starter` web and `basic-256mb` database). Confirm current plan availability and pricing in Render before provisioning. Do not put passwords in source control. Consider SSO or a proxy with authentication before making staff performance accessible publicly: **the example protects the upload endpoint but read-only dashboards are otherwise public**.

## Required Excel structure

| Sheet | Required columns (representative) | Used for |
|---|---|---|
| `Raw` | `UPSELL_TYPE`, `BSNS_INTRACN_KEY`, `INTRACN_DT_CMPLET`, `INTRACN_STAT`, `NET_RC`, `DLR_OUTLET_CODE`, `SL_PARTNER_CODE`, `SL_PARTNER_NAME` | Completed Upsell transactions (quantity=1/row) |
| `TARGET` | `CODE`, `TOL_UPSELL` | Monthly shop target |
| `TARGET STAFF` | header on Excel row 2: `EMP_ID`, `NAME`, `shop_code`, `TOL_UPSELL` | Staff roster and monthly targets |
| `Shop list` | `SHOP CODE`, `SHOP NAME`, `REGION`, `AREA`, `TYPE SHOP` | Shop attributes |

Excel sheet names and columns must match. Import rejects invalid workbooks without clearing the previous data. Re-upload **replaces** the entire sales, shop, and staff snapshots in one DB transaction; it does **not append**, avoiding duplicate runs. Import history is retained. Blank/invalid rows and non-completed transactions are ignored. Duplicates with the same `(transaction ID, product, shop)` are ignored.

## KPI definitions & important assumptions

- **Total Target**: sum of monthly `TARGET.TOL_UPSELL` across shops in scope, including shops with zero sales.
- **Total Actual**: sum of `Raw.NET_RC` for completed Upsell transactions in selected month, filtered by region/area/shop/product.
- **Total Runrate**: `Actual × days in selected month ÷ elapsed days`. For historical months, the elapsed cutoff is the last sales date in the uploaded snapshot; if no sales exist, the calendar month end is used. **This is a simple projection, not a forecast model**.
- **%Achievement**: `Actual / Target × 100`; **RR %Achievement**: `Runrate / Target × 100`; **GAP**: `Actual − Target` (negative means shortfall).
- **MOM Growth**: `(selected month Actual − previous month Actual) ÷ previous month Actual × 100`. No prior-month data results in `—`, not zero.
- **Active Shops / Staff**: shops and selling staff with at least one completed transaction in the selected reporting month.
- **No Sale Staff**: staff listed in `TARGET STAFF` with no matching completed transaction, for selected shops. Staff attribution uses staff ID where possible, otherwise normalized staff names; unmatched seller rows remain visible separately.
- **Quantity**: 1 transaction line = 1 quantity (no independent qty column in supplied `Raw`).
- **Product selection**: targets/achievement/gap are suppressed at KPI level because the workbook contains an overall monthly Upsell target, not per-product targets. Product breakdown still shows actuals.
- **WOW**: ISO week (Monday–Sunday), with ISO week-year labeling; **MTD**: month-to-date snapshot; **MOM**: month-over-month. Data not present in the workbook cannot be inferred. `Raw` did not contain a previous-year reference series.
- **Top / Bottom**: ranked by actual revenue, not achievement. Bottom 10 includes shops or staff with zero sales.

Note: The seed file has mixed sale periods and some unlinked staff transaction identifiers. For strictly accurate staff rankings, ensure `SL_PARTNER_CODE` aligns with `TARGET STAFF.EMP_ID` or that seller names match. For production, add a maintained mapping table for staff identifiers.

## API

- `GET /api/options` — filter options
- `GET /api/dashboard?month=2026-10&region=UPC%20II&area=...&shop=...&product=CCTV` — all KPIs, series, and ranked detail
- `POST /api/upload` — multipart `file=.xlsx`, header `X-Admin-Token: <ADMIN_TOKEN>`
- `GET /health` — database liveness for Render
- `GET /docs` — OpenAPI interactive API documentation

## Local development without Docker

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
export ADMIN_TOKEN='change-this-long-secret'
export DATABASE_URL='sqlite:///./retail_ww.db' # local tests only
uvicorn app.main:app --reload
```

Database tables are automatically created at startup. For long-lived enterprise deployments, migrate the simple startup schema creation to Alembic migrations. This starter intentionally has no organization-wide authentication or RBAC.
