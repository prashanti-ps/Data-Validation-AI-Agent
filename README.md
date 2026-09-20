# An agentic data-quality investigation system

An AI-powered multi-agent system that automatically investigates data-quality mismatches between source systems and a target data warehouse, identifies the likely root cause using enterprise data and metadata, and generates an actionable Root Cause Analysis (RCA).

The system transforms a traditionally manual, multi-step data-engineering investigation into an automated agentic workflow.

---

## Problem

Data certification and reconciliation processes can identify that a mismatch exists, but determining **why** it happened often requires significant manual investigation.

A typical investigation requires an engineer to:

1. Identify the mismatched records or columns
2. Trace data lineage across multiple layers
3. Query source and target systems
4. Inspect ingestion logic and schemas
5. Determine where the data diverged
6. Document the root cause and next steps

This process is time-consuming, inconsistent, and difficult to scale as the number of data sources and tables increases.

---

## Solution

This project uses a **multi-agent architecture** to automate the first-pass investigation.

Instead of asking a single LLM to analyze everything, the system decomposes the investigation into specialized responsibilities:

* **Master Agent** — understands the investigation context, gathers evidence, and orchestrates the workflow
* **Count Mismatch Agent** — investigates missing or extra records and traces natural keys across source and target layers
* **Column Mismatch Agent** — investigates value-level discrepancies using column lineage, schemas, and ingestion logic
* **Merger Agent** — synthesizes findings from the investigation agents into a concise RCA report

The agents use enterprise data sources and metadata as evidence rather than relying solely on the language model's generated response.

---

## High-Level Architecture

```text
                         ┌──────────────────────┐
                         │   Data Certification │
                         │       / Airflow      │
                         └──────────┬───────────┘
                                    │
                              Mismatch Found
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    FastAPI / API     │
                         │      Interface       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │          MASTER AGENT          │
                    │       Google ADK + Gemini      │
                    │                                │
                    │  • Validate input              │
                    │  • Gather investigation data  │
                    │  • Manage state/artifacts      │
                    │  • Orchestrate agents          │
                    └───────────────┬────────────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                │
                    ▼                                ▼
          ┌──────────────────┐             ┌──────────────────┐
          │ Count Mismatch   │             │ Column Mismatch  │
          │     Agent        │             │      Agent       │
          │                  │             │                  │
          │ • Missing keys   │             │ • Column lineage │
          │ • Source checks  │             │ • Schema / DDL   │
          │ • Bronze checks  │             │ • Ingestion code │
          │ • Root cause     │             │ • Root cause     │
          └────────┬─────────┘             └────────┬─────────┘
                   │                                │
                   └───────────────┬────────────────┘
                                   │
                                   ▼
                         ┌──────────────────────┐
                         │     MERGER AGENT     │
                         │                      │
                         │ • Combine findings   │
                         │ • Generate RCA       │
                         │ • Summarize evidence │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      RCA Report      │
                         │                      │
                         │ • Root cause         │
                         │ • Evidence            │
                         │ • Investigation      │
                         │ • Next steps         │
                         └──────────┬───────────┘
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                  Cloud Storage          Stakeholder
                  Report / Logs             Delivery
```

---

## Agentic Workflow

### 1. Trigger

An automated data-certification workflow can trigger an investigation when a mismatch is detected.

A human can also initiate an investigation using the API/interface by providing the relevant process identifier.

### 2. Evidence Collection

The Master Agent gathers:

* Certification results
* Mismatch records
* Table and column metadata
* Natural-key information
* Lineage information
* Relevant schema and ingestion metadata

Large datasets are stored as artifacts rather than repeatedly passing them through the LLM context.

### 3. Specialized Investigation

The investigation is divided by mismatch type.

**Count Mismatch Agent**

Investigates whether records:

* Were never ingested
* Were dropped during transformation
* Exist in the source but not the target
* Differ across ingestion layers

**Column Mismatch Agent**

Investigates:

* Column-level lineage
* Source-to-target mappings
* Schema definitions
* Ingestion logic
* Transformation-related discrepancies

### 4. Root Cause Synthesis

The Merger Agent combines the findings into a single human-readable RCA.

The report focuses on:

* What failed
* Where the discrepancy originated
* Evidence supporting the conclusion
* Recommended next investigation/remediation steps

### 5. Delivery

The RCA is stored for audit/troubleshooting purposes and delivered to relevant engineering stakeholders.

---

## Agent Tools & Enterprise Data

The agents interact with enterprise systems through controlled tools rather than directly relying on LLM knowledge.

```text
                    Agent
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
    BigQuery      Snowflake      Teradata
        │             │             │
        └─────────────┼─────────────┘
                      ▼
               PostgreSQL
          Metadata / Lineage
                      │
                      ▼
              Schemas / DDLs
             Ingestion Metadata
```

This allows the agent to ground its investigation in **actual data and metadata**.

---

## Agent Communication

The system uses Google ADK session state and artifacts for inter-agent communication.

* **Session state** stores structured investigation results
* **Artifacts** store larger JSON datasets by reference
* Agents do not repeatedly send large payloads through the LLM context
* This reduces unnecessary token usage and keeps the workflow manageable

---

## Reliability & Scalability

A key engineering challenge was handling concurrent agent execution and LLM rate limits.

The system incorporates:

* Dynamic batching
* Concurrency controls
* Retry and exponential backoff
* Controlled execution sequencing
* Rate-limit handling
* Health checks
* Persistent logs
* Failure handling

The initial design used parallel investigation agents. Based on observed rate-limit behavior, the workflow was changed to controlled sequential execution to improve reliability under load.

This illustrates an important principle of production agentic systems:

> **Agent orchestration is not only about reasoning—it also requires managing concurrency, cost, reliability, and system limits.**

---

## Technology Stack

| Layer              | Technology                                            |
| ------------------ | ----------------------------------------------------- |
| Language           | Python                                                |
| Agent Framework    | Google Agent Development Kit (ADK)                    |
| LLM                | Gemini                                                |
| API                | FastAPI                                               |
| Data Warehouse     | BigQuery                                              |
| Source Systems     | Snowflake, Teradata                                   |
| Metadata / Lineage | PostgreSQL                                            |
| Orchestration      | Apache Airflow                                        |
| Storage            | Google Cloud Storage                                  |
| Containerization   | Docker                                                |
| Compute            | Google Kubernetes Engine                              |
| CI/CD              | GitHub Actions                                        |
| Deployment         | Helm + ArgoCD                                         |
| Networking         | Istio                                                 |
| Security           | Secret Manager + Workload Identity                    |
| Observability      | OpenTelemetry, Cloud Trace, Cloud Logging, Prometheus |

---

## Operational KPIs

The system is designed to measure more than just whether an agent produces an answer.

### Investigation Efficiency

**Mean Time to RCA (MTTR)**
Time from mismatch detection to delivery of the initial RCA.

**Manual Investigation Reduction**
Percentage reduction in engineer investigation time compared with the traditional workflow.

**First-Pass RCA Rate**
Percentage of mismatches for which the automated investigation provides a usable first-pass RCA.

### Agent Quality

**RCA Accuracy**
Percentage of automated root-cause conclusions confirmed by engineers.

**Evidence Coverage**
Percentage of RCA conclusions supported by traceable source/metadata evidence.

**Human Escalation Rate**
Percentage of investigations requiring human review.

### Operational Reliability

**Agent Completion Rate**
Percentage of investigations completed without timeout or system failure.

**Tool Failure Rate**
Frequency of failed database/API/tool calls.

**Average Investigation Cost**
LLM + query + infrastructure cost per investigation.

### Adoption

**RCA Adoption Rate**
Percentage of generated reports that engineers use without restarting the investigation manually.

**Recurring Issue Detection**
Number of repeated/systemic data-quality issues identified across investigations.

---

## Business Impact

The system changes the data-quality workflow from:

```text
Mismatch
   ↓
Engineer investigates manually
   ↓
Trace lineage
   ↓
Query multiple systems
   ↓
Inspect ingestion logic
   ↓
Determine RCA
   ↓
Write report
```

to:

```text
Mismatch
   ↓
Agentic investigation
   ↓
Evidence gathering
   ↓
Specialized RCA agents
   ↓
Evidence-backed root cause
   ↓
Automated RCA
   ↓
Engineer focuses on remediation
```

The resulting benefits include:

* Faster data-quality investigations
* Reduced repetitive engineering effort
* More consistent RCA methodology
* Better visibility into recurring data-quality issues
* Faster escalation of systemic problems
* More engineering capacity for prevention and remediation

---

## Dashboard & Operational Visibility

A dashboard can be layered on top of the agent workflow to provide operational visibility into:

* Investigation volume
* Investigation status
* RCA turnaround time
* RCA accuracy
* Automated vs. human investigations
* Agent success/failure rates
* Recurring root-cause categories
* Data-quality trends
* System reliability
* Cost per investigation

This turns the project from an individual AI workflow into an **operational data-quality platform**.

---

## Key Takeaway

This project demonstrates how **Agentic AI + enterprise data + tool calling + multi-agent orchestration** can transform a manual data-engineering workflow into an automated investigation system.

The core principle is:

> **Don't use an LLM simply to generate an answer. Give the agent the tools, data, context, and controls required to investigate a real operational problem.**
