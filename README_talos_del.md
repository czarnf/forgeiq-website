# Talos Intelligent Systems — Data Extraction Layer (DEL)

> **Margin intelligence for UK SME food & beverage manufacturers — built from data they already have.**

[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)]()
[![Tests](https://github.com/czarnf/talos-del/actions/workflows/ci.yml/badge.svg)](https://github.com/czarnf/talos-del/actions/workflows/ci.yml)
[![Sprint](https://img.shields.io/badge/sprint-2%20%E2%80%94%20Schema%20Depth-brightgreen)]()

---

## What This Is

The **Talos DEL (Data Extraction Layer)** is the data pipeline at the core of the Talos Intelligent Systems platform — a commercially-deployed margin intelligence system for UK SME manufacturers, with Food & Beverage as the Phase 1 target sector.

UK food manufacturers carry enormous amounts of commercially valuable data inside their ERP systems, spreadsheets, and production logs — but almost none of them can query it. They cannot answer questions like:

- Which batches are actually profitable vs. which ones are quietly losing money?
- Where does our ingredient cost deviate from what we planned — and by how much?
- What is the real yield on our production runs, and what is the variance costing us each month?

Talos exists to answer these questions. The DEL is the pipeline that makes it possible: a 7-layer normalisation engine that takes raw ERP exports, cleans and validates them, and loads them into a structured schema ready for variance analysis and margin reporting.

---

## Commercial Context

Talos operates a **7-Day Batch Margin Audit** delivery model:

| Stage | What happens |
|-------|-------------|
| Client data handoff | Client exports last 30–60 days of batch/production/purchase data from ERP (Sage 50/200, Access, or Excel) as CSV. Sent via ProtonDrive. |
| DEL processing (off-site) | Pipeline runs on consultant's machine — auto-detects ERP profile, maps columns, normalises, validates. Zero client IT dependency. |
| Variance analysis | Price variance, usage variance, and yield variance calculated per batch. Top loss drivers quantified in £. |
| Report delivery (Day 7) | Talos Health Report delivered via video call: board-ready 15-page PDF. Each finding quantified in £. Minimum 5× ROI guarantee. |

**No software runs on client infrastructure.** The client's data never leaves their site unencrypted. This design eliminates IT access risk, data residency concerns, and integration overhead — critical for SME environments where IT governance is minimal.

The intake template (Excel, 4 tabs) exists as a fallback for clients without ERP export capability. The standard delivery path is DEL ingestion of raw ERP exports.

---

## Architecture

```
talos-del/
├── forgeiq_extract/          # Python package (migrating to talos_extract in Sprint 3)
│   ├── ingestion/            # Layer 1: File discovery, SHA-256 fingerprinting, run log
│   ├── parsing/              # Layer 2: CSV, Excel (multi-sheet), PDF extraction
│   ├── schema_mapping/       # Layer 3: Fuzzy column matching via YAML ERP profiles
│   ├── normalisation/        # Layer 4: Currency cleaning, date parsing, type coercion
│   ├── validation/           # Layer 5: Quality scoring (0–100/record), dataset gate ≥60
│   ├── storage/              # Layer 6: SQLite → PostgreSQL (Phase 2)
│   └── output/               # Layer 7: Health report output, diagnostic reporter
├── config/
│   ├── sage200.yaml          # Sage 200 CNC/Engineering ERP profile (Sprint 2)
│   ├── access_mrp.yaml       # Access ManufacturePlus profile (Sprint 2)
│   ├── generic.yaml          # Fallback for custom Excel schemas (Sprint 2)
│   └── [F&B profiles]        # sage50_food.yaml, generic_food.yaml (Sprint 3 — next)
├── tests/
└── pyproject.toml
```

### The 7-Layer Pipeline

```
Raw ERP Export / Spreadsheet / CSV Batch Data
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 1 — Ingestion                                         │
│  Discover files · SHA-256 fingerprint · Skip already-seen   │
│  Log every run to run_log (idempotent re-run guard)          │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 2 — Parsing                                           │
│  CSV: encoding detection (UTF-8 BOM, latin-1, cp1252)        │
│       delimiter sniffing (CSV/TSV/pipe)                      │
│  Excel: multi-sheet scoring, preferred sheet detection       │
│  PDF: pdfplumber table extraction + text fallback            │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 3 — Schema Mapping                                    │
│  YAML ERP profile auto-detection by column overlap scoring   │
│  rapidfuzz token_sort_ratio against 100+ field aliases       │
│  Unmatched columns retained as _raw_{col} (no data lost)     │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 4 — Normalisation                                     │
│  Floats: £/$€ symbols, commas, bracket negatives             │
│  Dates: 8 format patterns (UK/EU/US/ISO)                     │
│  Status: WON | LOST | PENDING | UNKNOWN (Sage 200 Closed→WON)│
│  Strings: title-case, upper-case identifiers                 │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 5 — Validation / Quality Gating                       │
│  Per-record score 0–100                                      │
│  Structured error classifier: 18 error codes, WARN/ERROR     │
│  Dataset gate: mean score ≥ 60 required to proceed           │
│  Diagnostic reporter: diagnostic.txt per run                 │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 6 — Storage                                           │
│  Phase 1: SQLite (local, zero infra)                         │
│  Deduplication: SHA-256 exact + composite key (job+customer) │
│  Phase 2: PostgreSQL via SQLAlchemy engine swap + Alembic    │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Layer 7 — Output                                            │
│  Talos Health Report: Margin Leakage · Ghost Stock · Yield   │
│  All findings quantified in £ (board-ready deliverable)      │
│  diagnostic.txt: error log, unmapped columns, recommendations│
└──────────────────────────────────────────────────────────────┘
```

---

## Canonical Schema (Sprints 1 & 2 — Engineering/CNC)

14 canonical fields covering standard ERP job/quote data. F&B canonical fields (batch_id, recipe_name, yield_actual, ingredient, unit_cost) are added in Sprint 3.

| Field | Type | ERP sources (examples) |
|-------|------|------------------------|
| `job_number` | str | "Job Number", "WO Number", "Order Ref", "Works Order No" |
| `customer_name` | str | "Customer", "Account Name", "Client", "Sold To" |
| `quoted_value` | float | "Quoted Value", "Net Value", "Contract Value", "Sale Price" |
| `actual_cost` | float | "Actual Cost", "Job Cost", "Total Cost", "Works Cost" |
| `won_lost` | str | "Status", "Outcome", "Won/Lost", "Quote Status" |
| `date_quoted` | date | "Date Quoted", "Quote Date", "Raised Date" |
| `date_ordered` | date | "Date Ordered", "Order Date", "WO Date" |
| `date_completed` | date | "Date Completed", "Completion Date", "Closed Date" |
| `material_cost` | float | "Material Cost", "Materials", "Bought In Cost" |
| `labour_cost` | float | "Labour Cost", "Direct Labour", "Labor Hours Cost" |
| `job_description` | str | "Description", "Part Description", "Works Description" |
| `customer_code` | str | "Customer Code", "Account Code", "Cust Code" |
| `part_number` | str | "Part Number", "Drawing Number", "Stock Code", "Item Code" |
| `quantity` | float | "Quantity", "Qty", "Batch Size", "Units" |

---

## CLI Usage

```bash
# Install
pip install -e .

# Run the full pipeline on a folder of ERP exports
forgeiq run ./client_data/ --client "Bristol Bakehouse Ltd" --db client.db

# Dry run (validate only, no storage)
forgeiq run ./client_data/ --dry-run

# Check last run summary
forgeiq status --db client.db

# Generate health report
forgeiq report --db client.db --out ./output/
```

> **Note:** The CLI command (`forgeiq`) and Python package (`forgeiq_extract`) migrate to `talos` / `talos_extract` in Sprint 3. Existing integrations continue to work until that migration.

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| CLI | Click 8.1 | Standard Python CLI framework; clean command groups |
| Data wrangling | pandas 2.x | Industry standard; performant on SME-scale data (< 1M rows) |
| Excel | openpyxl | Native .xlsx read/write with sheet intelligence |
| PDF | pdfplumber | Table extraction from digital PDFs without OCR dependency |
| Fuzzy matching | rapidfuzz | C++ accelerated; handles ERP column name variations robustly |
| Validation | pydantic v2 | Schema enforcement, serialisation, field validators |
| Storage | SQLAlchemy 2 + SQLite | Zero-infra Phase 1; single engine swap to PostgreSQL for Phase 2 |
| Migrations | Alembic | Controlled schema evolution |
| Console | rich | Professional CLI output; progress bars, tables |
| Testing | pytest | 51 tests across all 7 layers (Sprints 1 & 2) |

---

## Development Roadmap

| Sprint | Focus | Status |
|--------|-------|--------|
| 1 — Foundation | 7-layer skeleton, CLI, ingestion, parsing, storage, 22 tests | ✅ Complete |
| 2 — Schema depth | YAML ERP profiles (Sage 200, Access MRP, Generic), Pydantic v2 models, error classifier, deduplicator, diagnostic reporter, 51 tests | ✅ Complete |
| 3 — F&B schema | sage50_food.yaml, generic_food.yaml, F&B canonical fields, abstract CNC-specific variables, F&B test fixtures | 🔲 Next |
| 4 — Variance engine | price_variance(), usage_variance(), yield_variance() functions, F&B E2E integration test | 🔲 |
| 5 — Health Report | Talos Health Report PDF (ReportLab): executive summary, batch variance table, loss drivers in £ | 🔲 |
| 6 — Monitoring | Monthly delta comparison, alert rules for price spike/yield drift, 3-page update report | 🔲 |
| 7 — Client portal | Secure CSV file drop, status tracker, report download (Streamlit or static HTML) | 🔲 |
| 8 — Multi-sector | Abstract all F&B-specific logic to universal DEL parameters, Precision Engineering profile re-entry | 🔲 |

---

## Project Background

Talos Intelligent Systems was founded in March 2026. This repository is the core technical component of the Talos platform — built as part of an Innovator Founder Visa application and commercial launch targeting UK SME food & beverage manufacturers.

**Founder:** Emmanuel — IT professional bridging technical systems and operational delivery across NHS, logistics, and manufacturing sectors.

**Phase 1 target market:** UK SME food & beverage manufacturers (£2M–£20M turnover) running Sage 50/200, Access, or Excel-based production tracking, with batch-level data available for export.

**Platform vision:** Sector-agnostic margin intelligence platform supporting Food & Beverage, Precision Engineering, Electronics Assembly, and Chemicals. Phase 1 proves the model in F&B; Phase 2 generalises to universal manufacturing DEL.

---

## Licence

Proprietary. All rights reserved. © 2026 Talos Intelligent Systems Ltd.
