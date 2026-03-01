# CLAUDE.md — API Integration into Microsoft Fabric

## Project Context

**Goal:** Extract data from source API → Load into Microsoft Fabric → Transform into dimensional model → Serve to Power BI.

**Tech Stack:** Python | Microsoft Fabric (Lakehouse + Warehouse) | Fabric Notebooks | Fabric Pipelines | Power BI

**Architecture:**
```
Source API → Fabric Notebook (Python) → Bronze Lakehouse (raw) → Stored Procedures → Fabric Warehouse (Fact & Dim) → Power BI
```

Supporting: Logging Lakehouse | GitHub (CI/CD) | Fabric Pipelines (daily orchestration)

---

## IMPORTANT: Verification First

**This is non-negotiable. Every pipeline must have verification criteria before writing any code.**

1. Download a single closed month from the source system UI (e.g., January P&L export)
2. This export is your **truth** — all reconciliations, journal entries, and adjustments are finalized
3. Your pipeline output MUST match this benchmark exactly before building further
4. Verify: row counts, totals by category, totals by location, grand total
5. If numbers don't match, stop and investigate — do NOT proceed to the next step

**YOU MUST** include these checks in every pipeline:
- Row count comparison: API extracted vs Bronze loaded
- Grain validation: no duplicate rows at expected grain level in Fact tables
- Join validation: row count after joins must match expected count (fan-out = broken join)
- Business total validation: pipeline output matches source system UI export

---

## IMPORTANT: Interview Me Before Planning

Before writing any extraction code, ask me these questions:

1. What is the API's pagination method? (cursor, $top/$skip, nextLink, continuation token?)
2. Is there a rate limit or throttling?
3. Are there parent-child relationships between endpoints?
4. Does the source system modify or delete historical records? (month-end reconciliation, journal entries, reallocations)
5. Is there a deleted records endpoint?
6. What does "correct" look like? How does the client validate these numbers today?
7. What is the reporting grain? (daily by location? monthly by GL account?)
8. What is the incremental load strategy? (modifiedOn? rowVersion? full reload?)

**Do NOT skip this step. Do NOT assume answers. Ask me.**

---

## API Extraction Rules

### Pagination — IMPORTANT

- **YOU MUST** use explicit `$orderby` on a unique field (e.g., primary key asc). Without ordering, records shift between pages — you WILL miss data or get duplicates.
- **YOU MUST** set explicit page size (e.g., `$top=1000`). Never rely on API defaults.
- Test pagination against a known record count. If counts don't match, pagination is broken.

### Parent-Child Endpoints

- **Check the API docs for recommended extraction patterns.** Many APIs say "get header IDs first, then query details by header ID." If the docs say this, follow it — do NOT extract detail tables directly.
- Batch parent IDs (e.g., 30 per request) to stay within URL length limits.

### Rate Limits

- Partition large extractions by date range (month by month) or by dimension (location).
- Implement exponential backoff with retry logic if throttled.
- For initial historical loads, pull month by month — never try to pull everything at once.

---

## Incremental Load Strategy

### Change Detection

- Use `modifiedOn` field with a **7-day lookback window**. This catches backdated entries, late journal postings, and month-end reconciliation changes.
- The lookback window is **NOT optional** — without it, numbers will silently drift.
- For small reference tables (< 10,000 rows), full reload each run is fine.

### Handling Deletes — IMPORTANT

- **YOU MUST** check if the API has a deleted records endpoint.
- If the source system processes month-end reconciliations by deleting and recreating records, tracking deletes is the ONLY way to maintain accuracy.
- Apply soft deletes in the Lakehouse (flag as deleted, don't physically remove).

### Upsert, Not Overwrite — IMPORTANT

- **NEVER** use `.mode("overwrite")` on daily loads. If the API returns fewer records than expected (timeout, rate limit), you wipe your entire table.
- Always use MERGE/upsert: insert new, update existing, based on primary key.

---

## Bronze Lakehouse — Raw Data

- Land data exactly as the API returns it. No transformations. No renaming. No aggregations.
- Store ALL fields — you will need them later.
- Add metadata columns to every table:
  - `_load_timestamp` — when the record was loaded
  - `_source_endpoint` — which API endpoint
  - `_batch_id` — unique identifier for each load run
- **Table naming:** snake_case, prefixed by source if multiple sources (e.g., `r365_transaction`)

---

## Fabric Warehouse — Dimensional Model

- All transformations happen in **stored procedures**, not Notebooks.
- Notebooks handle extraction and Bronze loading ONLY.
- **Naming:** `fact_` prefix for facts, `dim_` prefix for dimensions.
- Always include `dim_date` with complete date spine.
- **YOU MUST** define the grain of every fact table before building it.
- After building joins, validate: if row count exceeds expected, you have a fan-out.

---

## Logging — Non-Negotiable

Every API load logs to `Logging Lakehouse → dbo.api_load_log`:

- Endpoint name
- Start/end timestamp
- Records extracted vs records loaded (must match)
- Success/failure status + error message
- Load duration in seconds

**When something breaks at 3am, this table tells you exactly where and why.**

---

## Pipeline Orchestration — For Context Only

Claude Code does not configure Fabric Pipelines. You set that up yourself. But Claude Code needs to know the execution order so it writes code that respects dependencies.

Execution order:
1. Notebook runs first (extract → Bronze)
2. Stored Procedures run after (Bronze → Warehouse Fact/Dim)

When writing stored procedures, assume Bronze tables are already populated. When writing extraction scripts, assume they run before any transformation.

**Note:** Pipeline scheduling, failure alerts, Power BI report building, and dataset refresh are all handled by you outside Claude Code.

---

## Common Mistakes — Update This Section After Every Correction

**These are real mistakes made during builds. Each one cost debugging time.**

1. **No $orderby on pagination** — records shifted between pages, missed data. FIX: always order by primary key asc.
2. **Pulling detail tables directly** — API docs said to use header IDs. Missed this on first read. FIX: always check API docs for parent-child extraction pattern.
3. **No delete tracking** — month-end reconciliation deleted and recreated records. P&L numbers were wrong with no obvious cause. FIX: always check for deleted records endpoint.
4. **Overwrite instead of upsert** — one failed API call wiped the table. FIX: always use MERGE.
5. **No explicit page size** — API default page size varied. FIX: always set $top explicitly.
6. **TransactionDetail has no date field** — assumed it did, filtered on a field that didn't exist. FIX: always check the actual endpoint schema before writing filters.
7. **No lookback window** — incremental load missed backdated journal entries. FIX: always use 7-day lookback on modifiedOn.
8. **Validated too late** — built full pipeline before comparing against source system. Found mismatches after weeks of work. FIX: validate a single closed month FIRST, before building anything else.

---

## After Every Session

**IMPORTANT: If Claude makes a mistake during this project, add it to the "Common Mistakes" section above before closing the session. Every mistake becomes a permanent improvement.**
