# CLAUDE.md for Data Engineering

Real-world CLAUDE.md templates for data engineering projects using Claude Code.

## What is CLAUDE.md?

A CLAUDE.md file is a briefing document you drop in your project folder. Claude Code reads it automatically at the start of every session. Think of it like handing a new contractor your project rules before they start working.

Every line in these templates was earned through real debugging — not theory.

## Templates

### API Integration into Microsoft Fabric

**File:** `api-integration/CLAUDE.md`

Built from a real Restaurant365 OData API integration into Microsoft Fabric. Full pipeline — extraction, incremental loads, dimensional model, and Power BI reporting.

**What it covers:**

- Verification criteria before writing any code
- Interview questions Claude must ask before planning
- API extraction rules (pagination, parent-child endpoints, rate limits)
- Incremental load strategy (change detection, delete tracking, upsert patterns)
- Bronze Lakehouse and Warehouse conventions
- Logging requirements
- Pipeline orchestration context
- Common mistakes section that grows with every project

**Tech stack:** Python | Microsoft Fabric (Lakehouse + Warehouse) | Fabric Notebooks | Fabric Pipelines | Power BI

## How to Use

1. Copy the relevant `CLAUDE.md` into your project root folder
2. Update the project-specific details (API endpoints, table names, tech stack)
3. Open Claude Code in that folder — it reads the file automatically
4. Let Claude interview you before it starts building
5. After every mistake, add it to the Common Mistakes section

## Key Principles

These templates follow three rules from Boris (creator of Claude Code):

**Verification first.** Download a closed month from the source system before building anything. Give Claude the expected totals so it knows what the output should look like. Run the code yourself and compare.

**Interview before building.** Don't jump straight to "build me a pipeline." Let Claude ask you questions first — pagination, rate limits, parent-child relationships, reconciliation rules.

**Update after every mistake.** Missed pagination? Add it. Missed delete tracking? Add it. Next project starts where this one finished. The file gets smarter with every project.


