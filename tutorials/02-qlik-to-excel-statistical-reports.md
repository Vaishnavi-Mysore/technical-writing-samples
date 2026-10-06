# How to Build a Qlik-to-Excel Statistical Report Pipeline (Flask + React)

> **Type:** Step-by-step tutorial  
> **Stack:** Python, Flask, React, Qlik Sense, openpyxl, Oracle  
> **Based on:** Statistical-Reports-main

---

## What you will build

A reporting flow where:

1. The React SPA lets an operator pick a Qlik app, dimensions, and measures  
2. Flask fetches a hypercube from Qlik Sense  
3. A **report engine** fills a branded Excel template (bilingual / RTL-aware)  
4. The download is recorded in export history  

**Time:** Conceptual walkthrough of a production-shaped stack (not a toy hello-world)  
**Level:** Intermediate–advanced

---

## Prerequisites

- Python 3.11+ and Node.js 18+  
- Access to a Qlik Sense engine (certs/session as required)  
- Oracle DB for staging/production (`oracle+oracledb://...`)  
- A branded `.xlsx` template (with anchors your generators understand)

---

## Step 1 — Understand the architecture

```text
React SPA
   │  session cookie
   ▼
Flask API ──► Oracle (templates, configs, exports, metadata)
   │
   ├── qlik_service/    WebSocket client (metadata + hypercubes)
   └── report_engine/   generate_report(template, payload) → XLSX
```

**Rule:** React never talks to Qlik directly. Flask owns auth, visibility, and file generation.

---

## Step 2 — Set up the backend

```bash
cd Statistical-Reports-main/backend
python -m venv venv
venv\Scripts\activate          # Windows
pip install -r requirements.txt
copy .env.example .env
```

Minimum `.env` values:

| Variable | Purpose |
|----------|---------|
| `SECRET_KEY` | Session / signing |
| `DATABASE_URL` | `oracle+oracledb://...` |
| `CORS_ORIGINS` | Your Vite origin, e.g. `http://localhost:5173` |
| `ADMIN_PASSWORD` | Bootstrap admin (12+ chars) |
| `APP_ENV` | `development` |

```bash
python app.py
```

API: http://localhost:5000

Empty Oracle schema (staging):

```bash
python scripts/create_oracle_schema.py
```

---

## Step 3 — Set up the frontend

```bash
cd Statistical-Reports-main/frontend
npm ci
npm run dev
```

UI: http://localhost:5173  

Vite proxies `/api` to Flask in development.

---

## Step 4 — Connect to Qlik and enforce access

Use a composed client:

```text
QlikSenseService
  ├── connection / session
  ├── hypercube queries
  └── field / measure metadata
```

Before every export:

1. Authenticate the portal user (HttpOnly cookie)  
2. `require_app_access(qlik, dataset_id)` — visibility gate  
3. Ensure a live Qlik session from the connection pool  
4. Build the hypercube for the selected dims/measures/filters  

Datasets come from the **live Qlik app list**, filtered by portal visibility—not hard-coded app IDs in env vars.

---

## Step 5 — Shape a report payload

After Qlik returns rows, build a payload the Excel layer understands:

```python
report_payload = {
    "tables": [
        {
            "sheet_name": "Table 1",
            "table_number": "1",
            # rows, headers, bilingual titles, notes...
        }
    ],
    "index": {
        "title_ar": "...",
        "title_en": "...",
    },
    "metadata": {
        # sources, definitions, curated fields
    },
}
```

Keep Qlik-specific field names out of the template layer when possible—map them once in the export pipeline.

---

## Step 6 — Generate Excel with the report engine

```python
from app.report_engine.orchestrator import generate_report

generate_report(
    template_path="templates/fcsc_report.xlsx",
    output_path="/tmp/out.xlsx",
    report_payload=report_payload,
    logo_path="uploads/logo.png",
)
```

What `generate_report` does:

1. Validates **anchors** in the template  
2. Loads the workbook (openpyxl)  
3. Applies RTL sheet settings  
4. Writes table sheets, index, and metadata  
5. Ensures logos on sheets that need them  
6. Saves the output file  

### Why anchors matter

Designers edit the `.xlsx`. Developers fill named anchors. If anchors are missing, fail loudly—do not ship a half-branded file.

Theme/fonts live in `fcsc_style.py` + YAML export rules under `backend/config/`.

---

## Step 7 — Record export history

After a successful write:

- Insert an `Export` row (dataset, status, run id, file path)  
- Scope history queries to the current user (admins see all)  
- Expose `GET /api/exports?page=1&limit=50` for the SPA  

---

## Step 8 — Operator checklist (happy path)

1. Sign in  
2. Open **Datasets** → pick a visible Qlik app  
3. Build pivot (dims / measures / time filters)  
4. **Preview**  
5. **Export** → download XLSX  
6. Confirm row appears under **Exports**  

Optional flows:

| Goal | Where |
|------|--------|
| Reuse layout | Saved **Configurations** |
| Multi-sheet workbook | **Multi Export** |
| Bilingual field labels | SPA `/metadata` |
| Branding | Upload template / logo (extension + magic-byte checks) |

---

## Step 9 — Production harden (minimum)

From `SECURITY.md`:

- [ ] `APP_ENV=production`  
- [ ] Strong `SECRET_KEY` and `ADMIN_PASSWORD`  
- [ ] Exact `CORS_ORIGINS` (no `*`)  
- [ ] `QLIK_VERIFY_SSL=true`  
- [ ] No `.env`, PEMs, or Qlik session cookies in git  
- [ ] Uploads validated by magic bytes, not filename alone  
- [ ] Health: `/api/health/live`, `/api/health/ready`  

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| CORS errors | Match `CORS_ORIGINS` to the SPA origin exactly |
| 403 on export | App visibility / access check |
| Qlik connect fails | Certs, session dir, SSL verify, network |
| Template rejected | Real `.xlsx` magic bytes |
| Broken logo in Excel | Engine rewrites zip-backed image refs before save |

---

## What you learned

- Separate **Qlik fetch**, **payload shaping**, and **Excel generation**  
- Use templates + anchors instead of painting cells in ad hoc scripts  
- Gate analytics apps by portal visibility and session auth  
- Treat exports as auditable jobs (history rows), not anonymous downloads  

---

## Source

Statistical-Reports-main: `backend/app/report_engine/`, `backend/app/qlik_service/`, `backend/app/api/`, `SECURITY.md`.
