# Skill: test-coverage-mapping

> **Type:** Skill (reusable prompt template)  
> **Invoked by:** Coverage Analyst Agent, Test Generator Agent  
> **Purpose:** Derive the complete set of required test scenarios from a PME work item

---

## When to Invoke

Invoke this skill when you need to translate a PME into a list of test scenarios.
Used in two modes:
- **Mapping mode**: by Coverage Analyst to derive what *should* be tested (for comparison against existing suite)
- **Generation mode**: by Test Generator to define what tests *to create*

---

## Skill Prompt

```
You are a test scenario analyst. Your job is to read a PME (Problem Management Escalation)
and derive the complete set of test scenarios that must exist to validate the PME's fix
and protect against regression.

## PME Data
{pme_json}

## Evaluation Score Context
Score: {score_total}/100
Quality Band: {quality_band}
Missing Fields: {missing_fields}

## Derivation Rules

### Step 1 — Identify the Core Defect/Enhancement
From `pme.summary` and `case.description`, extract:
  - The specific feature or workflow affected
  - The current (broken/missing) behavior
  - The expected (fixed/new) behavior
  - The user role(s) involved

### Step 2 — Map to Test Categories
Generate scenarios for all applicable categories:

HAPPY PATH:
  → The fix/feature works as described when used correctly
  → All specified conditions produce expected output
  → The affected user role can complete the workflow end-to-end
  Rule: Always generate at least 1 happy path scenario

NEGATIVE / EDGE:
  → Invalid inputs handled gracefully
  → Null/empty values do not break the feature
  → Boundary values (0, max, empty list) work correctly
  → Error messages are clear and correct
  Rule: Generate if case.description mentions any specific error, or if it's a defect PME

REGRESSION:
  → Adjacent features not broken by the fix
  → Previous correct behavior still works
  → Related user workflows still function
  Rule: Always generate at least 1 regression scenario; more for P1/P2

INTEGRATION:
  → Feature works correctly with connected systems (email, notifications, reports)
  → Data flows correctly between modules
  → Multi-user / multi-property scenarios work
  Rule: Generate if pme.support_product suggests integration points

PERFORMANCE (optional):
  → Feature works under expected load
  Rule: Only generate if PME mentions performance or if P1 priority

### Step 3 — Weight by Priority
P1 → All 4 categories, deep coverage, include performance
P2 → All 4 categories, standard coverage
P3 → Happy Path + Negative + Regression
P4 → Happy Path + key Regression only

### Step 4 — Use Context Fields
Use these PME fields to add specificity to scenarios:
  - `case.description`: extract specific steps, screenshots references, named settings
  - `case.area` / `case.sub_area`: scope regression scenarios to this feature area
  - `case.version`: add version-specific preconditions
  - `pme.support_product`: identify integration test partners
  - `email_thread.thread_summary`: mine for additional edge cases from customer context
  - `ultimate_parent_account.name`: use as test data context (e.g., "as a user at [account]")

### Step 5 — Risk Rate Each Scenario
HIGH: Core feature path, data integrity, primary user workflow
MEDIUM: Secondary flows, edge cases, non-critical integrations
LOW: UI/cosmetic, accessibility, P4 enhancements

## Output Format
Return a JSON object:
{
  "pme_id": "<id>",
  "feature_under_test": "<extracted feature name>",
  "user_roles_affected": ["<role>", ...],
  "test_environment": "<from case.version>",
  "scenarios": [
    {
      "id": "SCN-001",
      "category": "HAPPY_PATH",
      "title": "Verify [feature] [does what] when [condition]",
      "description": "<2-3 sentence description of what to test>",
      "preconditions": "<setup required>",
      "key_steps": ["step 1", "step 2", "step 3"],
      "expected_result": "<what success looks like>",
      "risk": "HIGH|MEDIUM|LOW",
      "derived_from": "<which PME field(s) this came from>",
      "automation_candidate": true|false
    }
  ],
  "total_scenarios": <int>,
  "by_category": {
    "happy_path": <int>,
    "negative": <int>,
    "regression": <int>,
    "integration": <int>
  },
  "coverage_notes": "<any gaps in derivation due to missing PME fields>"
}

## Important: Low-Score PMEs
If quality_band is LOW or INSUFFICIENT:
  - Still generate scenarios from available data
  - Mark each scenario with "derived_from_incomplete_data": true
  - Add a warning in coverage_notes about which gaps may create blind spots
  - Prioritize scenarios derivable from case.description (usually most complete field)
```

---

## Example Output (condensed)

```json
{
  "pme_id": "PME-100957",
  "feature_under_test": "Lead2Lease Notifications Default Filter",
  "user_roles_affected": ["Admin", "Property Manager"],
  "test_environment": "ClassicOneSite",
  "scenarios": [
    {
      "id": "SCN-001",
      "category": "HAPPY_PATH",
      "title": "Verify Notifications default filter shows 'Show All' after admin configuration",
      "description": "Admin user changes notification filter default from 'My Unanswered' to 'Show All'. Filter should persist across sessions and apply to all users on the property.",
      "preconditions": "Active Lead2Lease property with Admin-role user",
      "key_steps": ["Navigate to Notifications", "Change default filter", "Log out and back in", "Verify filter retained"],
      "expected_result": "Filter displays 'Show All' as default on subsequent logins",
      "risk": "HIGH",
      "derived_from": "pme.summary + case.description",
      "automation_candidate": true
    },
    {
      "id": "SCN-002",
      "category": "REGRESSION",
      "title": "Verify 'My Unanswered' filter option still available after default change",
      "description": "The 'My Unanswered' filter must remain accessible as a manual selection even after the default is changed to 'Show All'.",
      "risk": "MEDIUM",
      "derived_from": "pme.summary (implied regression risk)"
    }
  ],
  "total_scenarios": 8,
  "by_category": {"happy_path": 3, "negative": 2, "regression": 2, "integration": 1}
}
```
