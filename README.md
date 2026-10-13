# Agentic AI for Data Validation

Note: This is a high-level representation of a system I worked on professionally. The architecture, workflows, tools, and examples shown here have been generalized for confidentiality and do not include proprietary code, credentials, internal configurations, or sensitive business information.

---

## The Problem

Data-certification workflows detect mismatches between source systems and a target data warehouse, but identifying **why** the data differs still requires manual investigation.

Engineers typically need to:

- Identify mismatched records or column values
- Trace source-to-target lineage across data layers
- Query multiple systems and inspect schemas or ingestion logic
- Find where the mismatch originated and document the RCA

This process is time-consuming and difficult to scale across many tables and pipelines.

## The Solution

A **multi-agent data-validation application** that automates the first-pass investigation and generates an evidence-backed Root Cause Analysis (RCA).

- **Master Agent:** Validates the request, collects evidence, and coordinates the investigation
- **Count Mismatch Agent:** Traces missing or extra records using natural keys
- **Column Mismatch Agent:** Investigates incorrect column values using lineage, schemas, and ingestion logic
- **Merger Agent:** Combines findings into an RCA with supporting evidence and next steps

```text
Data Mismatch Detected
        ↓
Master Agent → Evidence Collection
        ↓
Count / Column Mismatch Investigation
        ↓
Merger Agent → RCA + Next Steps
        ↓
Report Stored and Shared with Engineers
```

## Architecture

```text
Data Certification / Airflow
           │
           ▼
 Mismatch Detected → FastAPI
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MASTER AGENT                               │
│                    (Google ADK + Gemini)                        │
│  Validate input • Fetch mismatch context • Manage session state│
│  Select and coordinate investigation agents                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│   COUNT MISMATCH AGENT   │   │  COLUMN MISMATCH AGENT   │
├──────────────────────────┤   ├──────────────────────────┤
│ Investigation tools:     │   │ Investigation tools:     │
│ • Read natural keys      │   │ • Read column lineage    │
│ • Query source records   │   │ • Read schema / DDL      │
│ • Query Bronze records   │   │ • Inspect column mapping │
│ • Query target records   │   │ • Inspect ingestion logic│
│ • Trace missing/extra IDs│   │ • Compare column values  │
└─────────────┬────────────┘   └─────────────┬────────────┘
              │                              │
              └──────────────┬───────────────┘
                             ▼
                  ┌─────────────────────┐
                  │     MERGER AGENT    │
                  │ Combine findings    │
                  │ Evidence + RCA      │
                  │ Recommended actions │
                  └──────────┬──────────┘
                             ▼
                       RCA REPORT
                             │
                   ┌─────────┴─────────┐
                   ▼                   ▼
            Cloud Storage         Engineering Team
            Report / logs         Review / next steps

Shared tools and enterprise data:
  BigQuery          → Target warehouse queries
  Snowflake/Teradata→ Source system queries
  PostgreSQL        → Metadata, natural keys, and lineage
  Certification data→ Mismatch details and results
  GCS               → Large artifacts and RCA reports
```

*Tool labels describe capabilities, not exact internal function names.* Google ADK **session state** shares structured findings between agents; **artifacts** hold large JSON results without repeatedly inserting them into the LLM context.

## Tech Stack

**Python · Google ADK · Gemini · FastAPI · Airflow · BigQuery · Snowflake · Teradata · PostgreSQL · GCS · Docker · GKE · GitHub Actions · Helm · ArgoCD**

## End-to-End Example: Count Mismatch

Suppose data certification detects records that exist in a source table but are missing from the target.

1. **Trigger:** Airflow/data certification detects the mismatch and submits the process details through FastAPI.
2. **Collect evidence:** The Master Agent retrieves the mismatch results, natural keys, and relevant metadata.
3. **Investigate:** The Count Mismatch Agent compares records across the source, Bronze, and target layers to locate where the keys diverged.
4. **Generate RCA:** The Merger Agent combines the findings into a likely root cause, supporting evidence, and recommended next steps.
5. **Deliver:** The report is stored in Cloud Storage and shared with the engineering team.

For a **column mismatch**, the Column Mismatch Agent instead checks column lineage, source-to-target mappings, schema definitions, and ingestion logic.

## Production Safety and Guardrails

The LLM guides the investigation, but data and metadata checks happen through **controlled tools**. The RCA should be supported by retrieved evidence, not by the model's assumptions.

```text
LLM Investigation
        ↓
Controlled Source / Target / Metadata Queries
        ↓
Evidence-Backed Findings
        ↓
RCA + Recommended Next Steps → Engineer Review
```

- **Input validation:** The Master Agent validates the investigation request and gathers the relevant certification context.
- **Evidence grounding:** The agents use actual mismatch records, natural keys, lineage, schema/DDL, and ingestion metadata.
- **Controlled access:** Database and metadata access is performed through tools instead of unrestricted LLM access.
- **Context management:** Structured findings are stored in session state; large query results are passed as artifacts.
- **Human-owned remediation:** The documented workflow generates an RCA and recommendations; it does not describe automatic changes to source or target data.

**For inconclusive investigations:** The safe handling policy is to flag missing evidence or tool failures and request engineer review rather than state an unverified root cause. This is a guardrail to enforce; the source design does not confirm it is already automated.

## Production Impact

| Metric | Impact |
|---|---|
| **Manual Investigation** | Automated first-pass RCA for count and column mismatches, reducing the need for engineers to manually query multiple systems, trace lineage, and inspect ingestion logic. |
| **Root Cause Identification** | Traces missing records and incorrect column values across data layers to identify where discrepancies originated, using source data, metadata, and lineage as evidence. |
| **RCA Generation** | Automatically generates a consolidated RCA report with the identified root cause, supporting evidence, and recommended remediation steps. |
| **Engineering Efficiency** | Shifts engineering effort from repetitive investigations toward validating findings and resolving underlying data-quality issues. |
