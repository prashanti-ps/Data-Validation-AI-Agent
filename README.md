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

This process was time-consuming and difficult to scale across multiple tables and pipelines. A typical investigation took 80–90 minutes manually.

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
                   Mismatch Detected → FastAPI
                              │
                              ▼
                        MASTER AGENT
                  (Understand + Orchestrate)
                              │
          ├─────────────────────────────────────────────┐
          ▼                                             ▼
    Count Mismatch Agent                        Column Mismatch Agent
          │                                             │
          ▼                                             ▼
┌────────────────────────────────┐    ┌────────────────────────────────┐
│ read_input_comparison_data     │    │ read_input_comparison_data     │
└────────────────────────────────┘    └────────────────────────────────┘
┌────────────────────────────────┐    ┌────────────────────────────────┐
│ get_full_table_lineage         │    │ get_full_column_lineage        │
└────────────────────────────────┘    └────────────────────────────────┘
┌────────────────────────────────┐    ┌────────────────────────────────┐
│ query_SC1_source_NKS           │    │ get_ingestion_script_and_      │
│ (Snowflake / Teradata)         │    │ schema_metadata                │
└────────────────────────────────┘    └────────────────────────────────┘
┌────────────────────────────────┐    
│ query_SC1_target_bronze_nks    │
│ (BigQuery - Bronze Layer)      │
└────────────────────────────────┘
          │                                             │
          └──────────────────────┬──────────────────────┘
                                 ▼
                             MERGER AGENT
                     (Combine Findings + RCA)
                                 │
                  ┌──────────────┴──────────────┐
                  ▼                             ▼
        ┌──────────────────────┐     ┌──────────────────────┐
        │   upload_to_gcs()    │     │     send_email()     │
        └──────────┬───────────┘     └──────────┬───────────┘
                   ▼                            ▼
        ┌──────────────────────┐     ┌──────────────────────┐
        │      GCS Bucket      │     │    Stakeholders      │
        │     (RCA Report)     │     │    (Email / RCA)     │
        └──────────────────────┘     └──────────────────────┘



┌────────────────────────────────┐
│      SHARED TOOLS / UTILITIES  │
├────────────────────────────────┤
│ Database Connection Management │
│ ADK Session State & Artifacts  │
│ Logging & Error Handling       │
│ Retry & Rate Limit Handling    │
└────────────────────────────────┘    

```
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

| Metric | Result |
| --- | --- |
| **Time to Initial RCA / Manual Investigation** | Reduced from **80–90 minutes to 3–5 minutes**, achieving ~95% faster investigations. |
| **Automated Investigation Coverage** | Percentage of eligible mismatches investigated automatically without manual intervention. |
| **Human Escalation Rate** | Percentage of investigations requiring human review due to insufficient evidence or uncertain root cause. |

