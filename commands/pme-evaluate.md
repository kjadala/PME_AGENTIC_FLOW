# Command: /pme-evaluate

> **Type:** User-invocable slash command  
> **Purpose:** Fetch a PME from TFS and score its quality — without running full coverage analysis  
> **Use when:** You want a quick quality check before deciding whether to run the full pipeline

---

## Usage

```bash
/pme-evaluate PME-100957
/pme-evaluate PME-100957 --project "Consumer Solutions\Lead Management Program"
/pme-evaluate PME-100957 --threshold 70
/pme-evaluate PME-100957,PME-101427,PME-102873   # batch mode
```

---

## Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `pme_id` | Yes | — | PME ID(s) to evaluate. Comma-separated for batch. |
| `--project` | No | Auto-detect from ADO field | TFS project path override |
| `--threshold` | No | 60 | Minimum score to recommend proceeding to coverage |
| `--format` | No | `summary` | `summary` \| `full` \| `json` |
| `--offline` | No | false | Use CSV export only (skip TFS fetch) |

---

## Prompt Template

```
Evaluate the quality of PME work item(s): {pme_ids}

Steps:
1. For each PME ID:
   a. Call tfs_get_work_item({pme_id}) to fetch the complete work item
   b. If TFS unavailable and --offline flag set, use the provided CSV/JSONL data
   c. Apply the pme-scoring skill to score the PME on the 100-pt rubric
   d. Return structured results

2. Output format based on --format flag:
   - summary: One-line result per PME + table of scores
   - full: Complete score breakdown with enrichment checklist
   - json: Raw JSON score object(s)

3. If batch mode (multiple IDs):
   - Score all PMEs
   - Sort by score ascending (worst first)
   - Show aggregate stats at end

Score threshold for recommendation: {threshold}
Project override: {project}

Return results and indicate which PMEs should proceed to /pme-coverage.
```

---

## Example Output

```
/pme-evaluate PME-100957

Fetching PME-100957 from TFS... ✓

┌─────────────────────────────────────────────────────────────────┐
│  PME QUALITY SCORE — PME-100957                                  │
│  Lead2Lease - Notifications filter                               │
├──────────────────────┬──────────┬───────┬────────────────────────┤
│  Dimension           │  Score   │  Max  │  Status                │
├──────────────────────┼──────────┼───────┼────────────────────────┤
│  Problem Clarity     │   16     │  20   │  ✅ Good               │
│  Business Impact     │    8     │  20   │  ⚠️  Missing fields    │
│  Technical Context   │   18     │  20   │  ✅ ADO linked         │
│  Reproducibility     │   20     │  20   │  ✅ Full description   │
│  Prioritization      │   10     │  10   │  ✅ P3, appropriate    │
│  Account Context     │    6     │  10   │  ⚠️  No CSM assigned   │
├──────────────────────┼──────────┼───────┼────────────────────────┤
│  TOTAL               │   78     │ 100   │  🟡 MEDIUM QUALITY     │
└──────────────────────┴──────────┴───────┴────────────────────────┘

Missing: domain, business_value, customer_impact

Recommendation: ✅ Score 78 ≥ threshold 60 — PROCEED TO COVERAGE
  → Run: /pme-coverage PME-100957
```

---

## Batch Output

```
/pme-evaluate PME-100957,PME-101427,PME-102873

Evaluating 3 PMEs...

  PME-102873  Score: 45/100  🔴 LOW          ❌ Do not proceed
  PME-101427  Score: 62/100  🟡 MEDIUM       ✅ Proceed with warnings
  PME-100957  Score: 78/100  🟡 MEDIUM       ✅ Proceed

Aggregate: Avg score 61.7 | 2/3 ready for coverage analysis

To run coverage on ready PMEs:
  /pme-coverage PME-101427,PME-100957
```
