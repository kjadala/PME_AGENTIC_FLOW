# Command: /pme-coverage

> **Type:** User-invocable slash command  
> **Purpose:** Run the full PME pipeline — evaluate, analyze test coverage, generate test suite if needed, produce report  
> **Use when:** You want the complete end-to-end analysis for a PME

---

## Usage

```bash
/pme-coverage PME-100957
/pme-coverage PME-100957 --generate-tests
/pme-coverage PME-100957 --threshold 70 --format full
/pme-coverage --file "C:\Users\kjadala\Downloads\pme-export.jsonl"  # batch from export
```

---

## Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `pme_id` | Yes (or `--file`) | — | PME ID to analyze |
| `--file` | No | — | Path to JSONL/CSV for batch processing |
| `--threshold` | No | 60 | Min score to proceed to coverage |
| `--generate-tests` | No | Auto (if no suite) | Force test generation even if suite exists |
| `--format` | No | `full` | `summary` \| `full` \| `json` |
| `--output-dir` | No | `./pme_reports/` | Directory for output files |
| `--tfs-write` | No | false | Create generated test cases in TFS |

---

## Prompt Template

```
Run the full PME evaluation and coverage analysis pipeline for: {pme_ids}

Pipeline Steps:
1. EVALUATE
   - Fetch PME from TFS via tfs_get_work_item
   - Apply pme-scoring skill
   - If score < 40: return enrichment report and stop
   - If score 40-59: continue with LOW_CONFIDENCE flag

2. COVERAGE ANALYSIS
   - Identify TFS project from pme.dev_tracking.azure_devops_project
   - Fetch test plans: tfs_get_test_plans(project)
   - Fetch test suites: tfs_get_test_suites(plan_id)
   - Apply test-coverage-mapping skill to derive required scenarios
   - Compare scenarios against existing test cases (semantic match)
   - Produce gap report: covered / partial / missing

3. TEST GENERATION (if needed)
   - If no suite found OR --generate-tests flag OR gaps exist:
     - Apply test-coverage-mapping skill in generation mode
     - Generate test cases for all 4 categories
     - If --tfs-write: create in TFS via tfs_create_test_case
     - Return test suite with IDs

4. REPORT
   - Apply report-format skill
   - Generate Markdown report to {output_dir}/{pme_id}_report_{date}.md
   - Generate Excel workbook to {output_dir}/{pme_id}_report_{date}.xlsx

Settings:
  Score threshold: {threshold}
  TFS write access: {tfs_write}
  Output directory: {output_dir}
  Format: {format}

Run all steps, show progress after each, return final report.
```

---

## Pipeline Progress Display

```
/pme-coverage PME-100957

═══════════════════════════════════════════════════════
  PME Coverage Analysis Pipeline — PME-100957
═══════════════════════════════════════════════════════

  [1/4] Fetching from TFS...                    ✓ 2.1s
        Work item: PME-100957 (Active, Q2 2026)

  [2/4] Evaluating PME quality...               ✓ 3.4s
        Score: 78/100 [MEDIUM] — proceeding

  [3/4] Analyzing test coverage...              ✓ 4.8s
        Test Plan: LMP Regression Suite (42 cases)
        Coverage: 50% (6/12 scenarios covered)
        Gaps: 4 (2 HIGH, 1 MEDIUM, 1 LOW)

  [4/4] Generating missing test cases...        ✓ 5.2s
        Generated: 4 test cases
        TFS: Not created (use --tfs-write to push)

═══════════════════════════════════════════════════════
  ✅ Complete — 15.5s
═══════════════════════════════════════════════════════

Reports saved:
  📄 pme_reports/PME-100957_report_2026-04-09.md
  📊 pme_reports/PME-100957_report_2026-04-09.xlsx

Executive Summary:
  PME-100957 scores 78/100 [MEDIUM QUALITY].
  Coverage is 50% — 4 gaps found (2 HIGH risk).
  4 test cases generated. Review and add to LMP suite before release.
```

---

## Batch Mode Output

```
/pme-coverage --file pme-export.jsonl

Processing 3,660 PMEs...
  Progress: ████████████████████ 100%

Completed: 3,660 PMEs
  ✅ High quality (80+):       892  (24%)
  🟡 Medium quality (60-79): 1,847  (50%)
  🟠 Low quality (40-59):      712  (19%)
  🔴 Insufficient (<40):       209  ( 6%)

Coverage Analysis (2,739 PMEs that proceeded):
  With test suite found:     1,612  (59%)
  No test suite:             1,127  (41%)
  
Test Cases Generated:        8,940

Reports saved to: ./pme_reports/
  Batch summary: pme_reports/batch_summary_2026-04-09.xlsx
```
