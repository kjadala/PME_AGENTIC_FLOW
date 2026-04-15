# PME Agentic Flow — TFS-Connected Evaluation & Test Coverage Engine

> **Automatically evaluate PMEs, score their quality, discover test coverage gaps, and generate test suites — all driven by Claude agents connected to TFS via MCP.**

---

## What This System Does

```
PME Export Data (JSONL/CSV)
        │
        ▼
┌───────────────────┐     TFS MCP Server
│  Orchestrator     │◄───────────────────┐
│  Agent            │                    │
└────────┬──────────┘                    │
         │                               │
         ▼                               │
┌───────────────────┐    Fetches full    │
│  PME Evaluator    │───work item ───────►
│  Agent            │    from TFS        │
└────────┬──────────┘                    │
         │ Score 0–100                   │
         ▼                               │
    ┌────┴────┐                          │
    │ ≥ 60 ?  │                          │
    └────┬────┘                          │
    YES  │  NO → Low-Quality Report      │
         ▼                               │
┌───────────────────┐    Fetches test    │
│  Coverage Analyst │───plans/suites ───►
│  Agent            │    from TFS        │
└────────┬──────────┘                    │
         │                               │
    ┌────┴────────┐                      │
    │  Suite      │                      │
    │  Found?     │                      │
    └──┬──────┬───┘                      │
  YES  │      │ NO                       │
       │      ▼                          │
       │  ┌──────────────────┐           │
       │  │  Test Generator  │           │
       │  │  Agent           │           │
       │  └────────┬─────────┘           │
       │           │                     │
       ▼           ▼                     │
┌───────────────────────────────────┐    │
│      Report Generator Agent       │    │
└────────────────┬──────────────────┘    │
                 │                       │
                 ▼                       │
     ┌───────────────────────┐           │
     │  Unified Output       │           │
     │  Report (MD + Excel)  │           │
     └───────────────────────┘           │
```

---

## Component Map

| Component | Type | Purpose |
|-----------|------|---------|
| `orchestrator-agent` | **Agent** | Entry point — routes PME through the pipeline, handles errors |
| `pme-evaluator-agent` | **Agent** | Fetches PME from TFS, scores quality on 100-pt rubric |
| `coverage-analyst-agent` | **Agent** | Reads test suite from TFS, maps gaps to PME requirements |
| `test-generator-agent` | **Agent** | Generates full test suite when none exists |
| `report-generator-agent` | **Agent** | Assembles unified markdown + Excel output |
| `/pme-evaluate` | **Command** | User entry point — triggers evaluation for a PME ID |
| `/pme-coverage` | **Command** | User entry point — runs full coverage + gap analysis |
| `/pme-report` | **Command** | User entry point — re-generates report from last run |
| `pme-scoring` | **Skill** | Reusable 100-pt scoring rubric invoked by evaluator |
| `test-coverage-mapping` | **Skill** | Maps PME fields → required test scenarios |
| `report-format` | **Skill** | Consistent report layout used by report agent |
| `tfs-mcp` | **MCP Server** | Azure DevOps / TFS API bridge for all data access |

---

## Quick Start

```bash
# 1. Configure TFS MCP server
cp mcp/tfs_mcp_config.example.json .claude/mcp_config.json
# edit with your TFS org URL and PAT

# 2. Evaluate a single PME
/pme-evaluate PME-100957

# 3. Run full pipeline (evaluate + coverage + report)
/pme-coverage PME-100957

# 4. Re-generate last report
/pme-report
```

---

## Repository Structure

```
PME_Agentic_Flow/
├── README.md                        ← You are here
├── ARCHITECTURE.md                  ← Full diagrams & decision rationale
├── IMPLEMENTATION_GUIDE.md          ← Step-by-step build guide
│
├── agents/
│   ├── 01_orchestrator.md           ← Orchestrator agent spec
│   ├── 02_pme_evaluator.md          ← PME Evaluator agent spec
│   ├── 03_coverage_analyst.md       ← Coverage Analyst agent spec
│   ├── 04_test_generator.md         ← Test Generator agent spec
│   └── 05_report_generator.md       ← Report Generator agent spec
│
├── skills/
│   ├── pme_scoring.md               ← 100-pt PME scoring rubric
│   ├── test_coverage_mapping.md     ← PME → test scenario mapper
│   └── report_format.md             ← Unified report template
│
├── commands/
│   ├── pme-evaluate.md              ← /pme-evaluate command spec
│   ├── pme-coverage.md              ← /pme-coverage command spec
│   └── pme-report.md                ← /pme-report command spec
│
└── mcp/
    ├── tfs_mcp_setup.md             ← TFS MCP server setup guide
    └── tfs_mcp_tools.md             ← Available TFS MCP tool reference
```

---

## Score Thresholds

| Score | Quality Band | Action |
|-------|-------------|--------|
| 80–100 | High Quality | Full analysis, proceed with confidence |
| 60–79 | Medium Quality | Proceed with enrichment warnings |
| 40–59 | Low Quality | Flag fields needing completion, partial analysis |
| 0–39 | Insufficient | Halt — return to reporter for more information |

---

## Prerequisites

- Claude Code CLI or Agent SDK
- TFS / Azure DevOps access with PAT token
- `tfs-mcp` MCP server installed and configured
- PME export data (JSONL or CSV from Salesforce)
