# Skill: report-format

> **Type:** Skill (reusable prompt template)  
> **Invoked by:** Report Generator Agent  
> **Purpose:** Enforce a consistent, professional report layout across all pipeline runs

---

## Skill Prompt

```
You are formatting a PME Evaluation Report. Apply the following layout exactly.

## Inputs
{pme_data}
{eval_result}
{coverage_result}
{generated_suite}
{pipeline_meta}

## Layout Specification

─────────────────────────────────────────────────────────────────────
SECTION 1: HEADER
─────────────────────────────────────────────────────────────────────
# PME Evaluation Report
**PME:** {pme_id} | **Product:** {support_product} | **Generated:** {date}
**Pipeline Mode:** {mode} | **Run Duration:** {duration}

---

─────────────────────────────────────────────────────────────────────
SECTION 2: EXECUTIVE SUMMARY (always ≤ 5 lines)
─────────────────────────────────────────────────────────────────────
> **[QUALITY BAND] PME** | Score: {score}/100
> **Coverage:** {coverage_pct}% | **Gaps:** {gaps_count} ({high_risk_gaps} HIGH risk)
> **Status:** {one_line_verdict}
> **Immediate Action:** {top_action}

| KPI | Value |
|-----|-------|
| PME Name | {pme_name} |
| Account | {upa_name} |
| Total ACV | ${upa_total_acv:,.0f} |
| Priority | {priority} |
| Days Open | {business_days_open} |
| Escalation Status | {escalation_status} |
| Assigned To | {assignee} |

─────────────────────────────────────────────────────────────────────
SECTION 3: PME SCORE CARD
─────────────────────────────────────────────────────────────────────
## Score Card — {score}/100 [{quality_band}]

| Dimension | Score | Max | Rating | Notes |
|-----------|------:|----:|--------|-------|
| Problem Clarity | {s1} | 20 | {r1} | {n1} |
| Business Impact | {s2} | 20 | {r2} | {n2} |
| Technical Context | {s3} | 20 | {r3} | {n3} |
| Reproducibility | {s4} | 20 | {r4} | {n4} |
| Prioritization | {s5} | 10 | {r5} | {n5} |
| Account Context | {s6} | 10 | {r6} | {n6} |
| **TOTAL** | **{score}** | **100** | **{band_emoji} {quality_band}** | |

Rating key: ✅ (≥80%) | ⚠️ (50–79%) | ❌ (<50%)

### Missing Fields
{missing_fields_checklist}

### Enrichment Recommendations
{enrichment_checklist_numbered}

─────────────────────────────────────────────────────────────────────
SECTION 4: TFS CONTEXT
─────────────────────────────────────────────────────────────────────
## TFS / Azure DevOps Details

| Field | Value |
|-------|-------|
| Work Item ID | {azure_devops_id} |
| State | {azure_devops_state} |
| Priority | {azure_devops_priority} |
| Project | {azure_devops_project} |
| Iteration | {ado_iteration} |
| Assigned To | {ado_assigned_to} |
| URL | [{azure_devops_url}]({azure_devops_url}) |

─────────────────────────────────────────────────────────────────────
SECTION 5: TEST COVERAGE ANALYSIS
─────────────────────────────────────────────────────────────────────
## Test Coverage Analysis

### Suite Status
{suite_status_block}
  If suite found:
    **Test Plan:** {test_plan_name} ({test_plan_id})
    **Total Test Cases:** {test_cases_total}
    **Coverage:** {coverage_pct}% ({scenarios_covered}/{scenarios_required} scenarios)
  If no suite:
    ⚠️ No test suite found for this product area. New suite generated below.

### Coverage Matrix
| Scenario | Status | Test Case | Risk |
|----------|--------|-----------|------|
{coverage_matrix_rows}

### Risk Summary
| Risk Level | Gap Count | Recommendation |
|------------|-----------|----------------|
| 🔴 HIGH | {high_gaps} | Resolve before release |
| 🟡 MEDIUM | {med_gaps} | Target current sprint |
| 🟢 LOW | {low_gaps} | Add to backlog |

─────────────────────────────────────────────────────────────────────
SECTION 6: TEST CASES (Missing / Generated)
─────────────────────────────────────────────────────────────────────
## Test Cases

{if gaps_exist}
### Required Test Cases (Gaps)
| ID | Title | Category | Priority | Risk | TFS Status |
|----|-------|----------|----------|------|------------|
{gap_test_case_rows}
{end_if}

{if suite_generated}
### Generated Test Suite — {suite_name}
| ID | Title | Category | Priority | Risk | TFS ID |
|----|-------|----------|----------|------|--------|
{generated_test_case_rows}

**TFS Creation:** {tfs_creation_status}
{end_if}

─────────────────────────────────────────────────────────────────────
SECTION 7: RECOMMENDATIONS
─────────────────────────────────────────────────────────────────────
## Recommendations

### Immediate Actions (before PME release)
{immediate_actions_checklist}

### PME Enrichment Needed
{pme_enrichment_checklist}

### Process Improvements
{process_recommendations}

─────────────────────────────────────────────────────────────────────
SECTION 8: APPENDIX
─────────────────────────────────────────────────────────────────────
## Appendix

<details>
<summary>Full PME Work Item (JSON)</summary>

```json
{pme_json_formatted}
```

</details>

<details>
<summary>Pipeline Run Metadata</summary>

| Field | Value |
|-------|-------|
| Run ID | {run_id} |
| Started | {start_time} |
| Completed | {end_time} |
| Mode | {mode} |
| Agents Used | {agents_list} |
| TFS Mode | {tfs_mode} (online/offline) |
| Warnings | {warnings_list} |

</details>

## Formatting Rules
- Dates: ISO 8601 (YYYY-MM-DD)
- Currency: $X,XXX,XXX (no decimals in summary, 2 decimals in detail)
- Scores: always show as X/Y (e.g., "78/100")
- Percentages: round to nearest integer
- Tables: always include header separator row
- Empty fields: show as "—" not blank or "null"
- Confidence warnings: prefix with ⚠️ and put in blockquote
- If quality_band is LOW or INSUFFICIENT: add a red banner after executive summary:
  > ❌ **Low-quality PME** — results may be incomplete. Enrich PME before acting on this report.
```
