# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

**PME Agentic Flow** is a specification and implementation guide for a multi-agent, TFS-connected evaluation and test coverage engine. The repository is documentation-driven — all agent logic, scoring rubrics, and command specs are defined in `.md` files. The implementation (Python + Claude Agent SDK) is built by following `IMPLEMENTATION_GUIDE.md`.

## User-Facing Commands (Slash Commands)

These are Claude Code slash commands defined under `commands/`:

```bash
/pme-evaluate PME-100957           # Quality check only (score + enrichment)
/pme-coverage PME-100957           # Full pipeline: evaluate + coverage + test gen + report
/pme-report PME-100957             # Re-generate report from prior run
/pme-coverage --file path/to.jsonl # Batch processing from Salesforce export
```

Key flags: `--threshold` (min score, default 60), `--generate-tests` (force test gen), `--tfs-write` (create in TFS), `--output-dir`, `--format`, `--offline`

## Architecture: 5-Agent Pipeline

```
User → Orchestrator → PME Evaluator (score) → Coverage Analyst (gaps)
                                                    ↓
                                             [suite exists?]
                                             NO → Test Generator
                                                    ↓
                                             Report Generator → .md + .xlsx
```

**Score-based routing** (defined in `agents/01_orchestrator.md`):
- `< 40` → HALT, return enrichment checklist only
- `40–59` → Proceed with `LOW_CONFIDENCE` flag
- `≥ 60` → Full analysis

**Agent specs:** `agents/01_orchestrator.md` through `agents/05_report_generator.md`

**Reusable skills** (prompt templates loaded by agents):
- `skills/pme_scoring.md` — 100-point rubric (6 dimensions)
- `skills/test_coverage_mapping.md` — rules for deriving test scenarios from PME fields
- `skills/report_format.md` — Markdown + Excel output template

## TFS/Azure DevOps Integration

All TFS access goes through an MCP server. Setup and tool reference in `mcp/`:
- `mcp/tfs_mcp_setup.md` — Installation (3 options), MCP config location (`~/.claude/mcp_config.json`)
- `mcp/tfs_mcp_tools.md` — 11 tool definitions: `tfs_get_work_item`, `tfs_get_test_cases`, `tfs_create_test_case`, etc.

Required PAT scopes: Work Items R/W, Test Management R/W, Project/Team R.

## Scoring Rubric Summary

Six dimensions totaling 100 pts (full criteria in `skills/pme_scoring.md`):

| Dimension | Points |
|-----------|--------|
| Problem Clarity | 20 |
| Business Impact | 20 |
| Technical Context | 20 |
| Reproducibility | 20 |
| Prioritization | 10 |
| Account Context | 10 |

Quality bands: HIGH (80–100), MEDIUM (60–79), LOW (40–59), INSUFFICIENT (0–39).

## When Implementing

The Python implementation should use:
- **Model:** `claude-sonnet-4-6`
- **Runtime:** Claude Agent SDK
- **Reports:** Markdown + `openpyxl` (Excel)
- **Retry logic:** 3 retries with exponential backoff (2s, 4s, 8s) on TFS failures; fallback to offline mode using Salesforce JSONL/CSV

Test files expected at: `tests/test_pme_scoring.py`, `tests/test_coverage_analyst.py`

CI/CD pipeline: `.github/workflows/pme_pipeline.yml` (triggered via `workflow_dispatch` with PME IDs).
