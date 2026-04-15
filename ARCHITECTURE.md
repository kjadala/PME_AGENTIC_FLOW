# Architecture — PME Agentic Evaluation & Test Coverage Engine

---

## 1. System Overview

The system is a **multi-agent pipeline** built on the Claude Agent SDK, connected to TFS/Azure DevOps through an MCP server. It operates as a directed workflow with conditional branching based on PME quality scores and test suite availability.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PME AGENTIC FLOW SYSTEM                          │
│                                                                         │
│  INPUT LAYER          AGENT LAYER              OUTPUT LAYER             │
│  ┌──────────┐        ┌─────────────┐          ┌──────────────────────┐ │
│  │ PME CSV  │        │             │          │  Unified Report      │ │
│  │ JSONL    ├──────► │ Orchestrator├────────► │  • Score Card        │ │
│  │ PME ID   │        │ Agent       │          │  • Gap Analysis      │ │
│  └──────────┘        │             │          │  • Test Suite        │ │
│                      └──────┬──────┘          │  • Recommendations   │ │
│                             │                 └──────────────────────┘ │
│         ┌───────────────────┼───────────────────┐                      │
│         ▼                   ▼                   ▼                      │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐            │
│  │ PME         │   │ Coverage     │   │ Test Generator   │            │
│  │ Evaluator   │   │ Analyst      │   │ Agent            │            │
│  │ Agent       │   │ Agent        │   │                  │            │
│  └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘            │
│         │                 │                    │                       │
│         └─────────────────┴────────────────────┘                      │
│                           │                                            │
│                    TFS MCP SERVER                                       │
│         ┌─────────────────┴──────────────────┐                        │
│         │  • get_work_item    • get_test_plan │                        │
│         │  • search_items     • get_test_suite│                        │
│         │  • create_test_case • update_item   │                        │
│         └────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Full Pipeline Flow Diagram

```mermaid
flowchart TD
    %% ── Input ──────────────────────────────────────────
    A([🗂️ PME Export\nCSV / JSONL / PME ID]):::input

    %% ── Orchestrator ────────────────────────────────────
    B[🎛️ Orchestrator Agent\nEntry point & router]:::agent

    %% ── TFS MCP ─────────────────────────────────────────
    MCP[(🔗 TFS MCP Server\nAzure DevOps API)]:::mcp

    %% ── Evaluator ───────────────────────────────────────
    C[🔍 PME Evaluator Agent\nFetches & scores PME]:::agent
    D{📊 PME Score\n0 – 100}:::decision

    %% ── Score branches ──────────────────────────────────
    E([❌ Low-Quality Report\nReturn for enrichment]):::output_bad
    F[⚠️ Proceed with\nWarnings]:::warn

    %% ── Coverage Analyst ─────────────────────────────────
    G[🧪 Coverage Analyst Agent\nFetches & maps test suite]:::agent
    H{📋 Test Suite\nExists in TFS?}:::decision

    %% ── Test Generator ───────────────────────────────────
    I[⚙️ Test Generator Agent\nBuilds new test suite]:::agent

    %% ── Report Generator ─────────────────────────────────
    J[📄 Report Generator Agent\nAssembles unified output]:::agent

    %% ── Output ───────────────────────────────────────────
    K([✅ Unified Output Report\nMarkdown + Excel]):::output_good

    %% ── Skills invoked inline ────────────────────────────
    SK1[[📐 Skill: pme-scoring\n100-pt rubric]]:::skill
    SK2[[🗺️ Skill: test-coverage-mapping\nPME → test scenarios]]:::skill
    SK3[[📑 Skill: report-format\nUnified template]]:::skill

    %% ── Flow ─────────────────────────────────────────────
    A --> B
    B -->|Resolve PME ID| MCP
    MCP -->|Work item data| C
    C --> SK1
    SK1 --> D

    D -->|Score < 40| E
    D -->|40 ≤ Score < 60| F
    D -->|Score ≥ 60| G
    F --> G

    G -->|Fetch test plans\nfor project| MCP
    MCP -->|Test suite data| H

    H -->|Suite found| G2[🔎 Gap Analysis\nvs PME requirements]:::agent
    G2 --> SK2
    SK2 --> J

    H -->|No suite| I
    I --> SK2
    SK2 --> J

    J --> SK3
    SK3 --> K

    %% ── Styles ───────────────────────────────────────────
    classDef input      fill:#1F3864,color:#fff,stroke:#fff,rx:8
    classDef agent      fill:#2E75B6,color:#fff,stroke:#1F3864,rx:6
    classDef mcp        fill:#375623,color:#fff,stroke:#375623,rx:6
    classDef decision   fill:#7030A0,color:#fff,stroke:#7030A0
    classDef skill      fill:#833C00,color:#fff,stroke:#C55A11,rx:4
    classDef output_good fill:#375623,color:#fff,stroke:#375623,rx:8
    classDef output_bad  fill:#C00000,color:#fff,stroke:#C00000,rx:8
    classDef warn        fill:#FF9900,color:#000,stroke:#FF9900,rx:4
```

---

## 3. Agent Interaction Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Orch as 🎛️ Orchestrator
    participant TFS as 🔗 TFS MCP
    participant Eval as 🔍 PME Evaluator
    participant Cov  as 🧪 Coverage Analyst
    participant Gen  as ⚙️ Test Generator
    participant Rep  as 📄 Report Generator

    User->>Orch: /pme-coverage PME-100957

    Orch->>TFS: get_work_item(PME-100957)
    TFS-->>Orch: Full work item JSON

    Orch->>Eval: Evaluate PME (work item data)
    Eval->>Eval: Apply pme-scoring skill
    Eval-->>Orch: Score=78, quality=MEDIUM, gaps=[domain, business_value]

    alt Score < 40
        Orch-->>User: ❌ Insufficient PME — missing fields list
    else Score 40–59
        Orch->>Cov: Proceed (low confidence flag)
    else Score ≥ 60
        Orch->>Cov: Proceed (score + evaluated PME)
    end

    Cov->>TFS: get_test_plans(project=azure_devops_project)
    TFS-->>Cov: Test plan list

    Cov->>TFS: get_test_suites(plan_id)
    TFS-->>Cov: Test suite + test cases

    alt Test suite found
        Cov->>Cov: Apply test-coverage-mapping skill
        Cov-->>Orch: Gap report (covered, missing, partial)
        Orch->>Rep: Build gap analysis report
    else No test suite
        Orch->>Gen: Generate test suite for PME
        Gen->>Gen: Apply test-coverage-mapping skill
        Gen->>TFS: create_test_case (each generated case)
        TFS-->>Gen: Created item IDs
        Gen-->>Orch: New test suite + TFS links
        Orch->>Rep: Build new suite report
    end

    Rep->>Rep: Apply report-format skill
    Rep-->>User: ✅ Unified Report (MD + Excel)
```

---

## 4. PME Scoring Decision Tree

```mermaid
flowchart LR
    START([PME Work Item]) --> A

    subgraph SCORING ["📊 100-Point Scoring Rubric"]
        A[Problem Clarity\n0–20 pts]
        B[Business Impact\n0–20 pts]
        C[Technical Context\n0–20 pts]
        D[Reproducibility\n0–20 pts]
        E[Prioritization\n0–10 pts]
        F[Account Context\n0–10 pts]
    end

    A --> TOTAL
    B --> TOTAL
    C --> TOTAL
    D --> TOTAL
    E --> TOTAL
    F --> TOTAL

    TOTAL{Total Score}

    TOTAL -->|80–100| HIGH[🟢 HIGH QUALITY\nFull analysis\nHigh confidence]
    TOTAL -->|60–79|  MED[🟡 MEDIUM QUALITY\nProceed with\nenrichment flags]
    TOTAL -->|40–59|  LOW[🟠 LOW QUALITY\nPartial analysis\nWarning in report]
    TOTAL -->|0–39|   INSUF[🔴 INSUFFICIENT\nHalt — return\nfor more info]

    HIGH --> NEXT([Proceed to\nTest Coverage])
    MED  --> NEXT
    LOW  --> NEXT2([Flag + Partial\nCoverage])
    INSUF --> STOP([Return Report\nOnly])
```

---

## 5. Test Coverage Analysis Flow

```mermaid
flowchart TD
    PME[PME Details\n+ Score] --> MAP

    subgraph MAP ["🗺️ Coverage Mapping (test-coverage-mapping skill)"]
        M1[Extract: product area\nand sub-area]
        M2[Extract: problem domain\nand scenario type]
        M3[Extract: affected\nuser journeys]
        M4[Extract: edge cases\nfrom case description]
        M1 & M2 & M3 & M4 --> REQ[Required Test\nScenarios List]
    end

    REQ --> COMPARE

    TFS2[(TFS Test Suite)] --> COMPARE

    subgraph COMPARE ["🔎 Gap Analysis"]
        C1{For each required\nscenario...}
        C1 -->|Found in suite| COVERED[✅ Covered]
        C1 -->|Partial match| PARTIAL[⚠️ Partially Covered]
        C1 -->|Not found| MISSING[❌ Missing]
    end

    COVERED & PARTIAL & MISSING --> RESULT

    subgraph RESULT ["📋 Gap Report"]
        R1[Coverage %]
        R2[Missing test cases\nwith descriptions]
        R3[Partial coverage\nrecommendations]
        R4[Risk rating\nper gap]
    end

    RESULT --> OUT([Report Generator])

    subgraph GENERATE ["⚙️ If No Suite Exists"]
        G1[Generate Happy Path tests]
        G2[Generate Negative/Edge tests]
        G3[Generate Regression tests]
        G4[Generate Integration tests]
        G1 & G2 & G3 & G4 --> SUITE[New Test Suite\nwith TFS IDs]
    end

    MISSING -.->|When no suite| GENERATE
    GENERATE --> Out2([Report Generator])
```

---

## 6. Component Decision Rationale

### Why Agents (not scripts)?

| Need | Why Agent Fits |
|------|---------------|
| Fetch PME from TFS + reason about quality | Requires LLM judgment — a script can't score narrative fields like `summary` or `business_impact` |
| Find test gaps | Semantic matching between PME requirements and test case titles — not string search |
| Generate test cases | Creative, context-aware generation based on PME domain, product, and case description |
| Route on score threshold | Dynamic decision-making with fallback strategies |
| Assemble report | Synthesizes multiple heterogeneous data sources into coherent narrative |

### Why Skills (not agents) for Scoring & Mapping?

| Reason | Detail |
|--------|--------|
| **Reusability** | Both `pme-evaluator` and `report-generator` need the scoring rubric — a skill keeps it DRY |
| **Consistency** | Every PME scored with the same 100-pt rubric — no drift between agent invocations |
| **Testability** | Skills can be validated independently with fixed inputs |
| **Lightweight** | No tool calls needed — pure prompt template with structured output |

### Why Commands for User Entry Points?

| Reason | Detail |
|--------|--------|
| **Discoverability** | `/pme-evaluate`, `/pme-coverage`, `/pme-report` are self-documenting |
| **Parameterized** | Accept PME IDs, project names, score thresholds as args |
| **Composable** | `pme-coverage` internally calls `pme-evaluate` first |
| **IDE-friendly** | Slash commands appear in Claude Code autocomplete |

### Why MCP (not direct API calls)?

| Reason | Detail |
|--------|--------|
| **Tool abstraction** | All 5 agents share the same TFS tools — one MCP server, not 5 separate HTTP clients |
| **Auth centralized** | PAT token managed once in MCP config |
| **Retries & errors** | MCP server handles rate limiting and TFS API errors |
| **Extensible** | Add `jira-mcp` or `ado-mcp` later without changing agent code |

---

## 7. Data Flow Map

```
PME JSONL Export                TFS / Azure DevOps
       │                               │
       │  pme.id                       │  Work Item fields
       │  pme.dev_tracking.            │  Test Plans
       │    azure_devops_project  ────►│  Test Suites
       │  pme.support_product          │  Test Cases
       │  case.description             │
       │                               │
       └──────────────┬────────────────┘
                      │
                      ▼
           ┌──────────────────────┐
           │   Enriched PME       │
           │   Object             │
           │   + TFS context      │
           └──────────┬───────────┘
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
      Score Card   Gap Report   Test Suite
          │           │            │
          └───────────┴────────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  Unified Output │
             │  ─────────────  │
             │  pme_report.md  │
             │  pme_report.xlsx│
             └─────────────────┘
```

---

## 8. Error Handling & Fallback Strategy

```
TFS Unreachable
      │
      ▼
Retry × 3 (exponential backoff)
      │
      ▼ (still failing)
Use PME CSV/JSONL data only
Score PME from export fields
Mark report: "TFS data unavailable — offline mode"

PME Score < 40
      │
      ▼
Generate enrichment checklist
List missing fields with descriptions
Return early — no test analysis
Recommend: "Complete these fields then re-run"

No Test Suite in TFS
      │
      ▼
Auto-generate test suite
Create test cases in TFS (if write access)
OR export as CSV for manual import
Mark report: "New suite generated — review before merge"
```

---

## 9. Technology Stack

| Layer | Technology |
|-------|-----------|
| Agent Runtime | Claude Agent SDK (claude-sonnet-4-6) |
| Orchestration | Claude Code CLI / Agent tool |
| TFS Integration | `tfs-mcp` MCP Server |
| PME Data | Salesforce export (JSONL → in-memory) |
| Report Output | Markdown + openpyxl (Excel) |
| Auth | TFS PAT token (MCP config) |
| Hosting | Local CLI or CI/CD pipeline |
