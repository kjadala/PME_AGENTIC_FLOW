# Skill: pme-scoring

> **Type:** Skill (reusable prompt template)  
> **Invoked by:** PME Evaluator Agent, Orchestrator Agent  
> **Purpose:** Apply a consistent 100-point quality rubric to any PME work item

---

## When to Invoke

Invoke this skill whenever you need to score a PME's completeness and quality.
It is intentionally a skill (not an agent) because:
- The rubric is **deterministic** — same PME always scores the same way
- It is **reusable** across multiple agents without duplication
- It requires no tool calls — pure reasoning over provided PME data
- It must be **consistent** across all pipeline runs

---

## Skill Prompt

```
You are scoring a PME (Problem Management Escalation) on a 100-point quality rubric.
A PME is a formal escalation record that links a customer-reported defect or enhancement
to an engineering work item. High-quality PMEs enable accurate prioritization and faster resolution.

## PME Data to Score
{pme_json}

## Scoring Rubric (100 points total)

### DIMENSION 1: Problem Clarity (0–20 points)
Evaluate the `pme.summary` and `case.description` fields.

  Sub-criterion A — Summary Specificity (0–10 pts):
    10 pts: Summary names the specific feature, behavior, and expected vs actual outcome
     8 pts: Summary names feature + either actual or expected (not both)
     5 pts: Summary names the feature but is vague ("doesn't work", "has issue")
     2 pts: Summary is a ticket number or placeholder only
     0 pts: Empty or null

  Sub-criterion B — Root Cause / Symptom (0–10 pts):
    10 pts: Case description includes clear steps to reproduce + root cause analysis
     8 pts: Steps to reproduce present but no root cause
     5 pts: Problem described but no steps or root cause
     2 pts: Case description is a single sentence
     0 pts: Empty or null

### DIMENSION 2: Business Impact (0–20 points)
Evaluate `pme.customer_impact`, `pme.business_impact`, `pme.business_value`.

  Sub-criterion A — Customer Impact (0–7 pts):
    7 pts: Quantified (# users, # properties, revenue at risk)
    5 pts: Described qualitatively with specifics
    2 pts: Vague ("customers are affected")
    0 pts: Empty or null

  Sub-criterion B — Business Impact (0–7 pts):
    7 pts: Specific business consequence documented (churn risk, SLA breach, etc.)
    5 pts: General impact described
    2 pts: Only "important" or "urgent" mentioned
    0 pts: Empty or null

  Sub-criterion C — Business Value (0–6 pts):
    6 pts: ROI or revenue impact quantified
    4 pts: Value described qualitatively
    1 pt: Field exists but content is generic
    0 pts: Empty or null

### DIMENSION 3: Technical Context (0–20 points)
Evaluate `pme.domain`, `pme.support_product`, `pme.dev_tracking.*`, `case.area`, `case.sub_area`.

  Sub-criterion A — Domain Assignment (0–5 pts):
    5 pts: Domain set and aligns with support product
    3 pts: Domain set but does not fully align
    0 pts: Empty or null

  Sub-criterion B — Support Product (0–5 pts):
    5 pts: Specific product identified with ID
    3 pts: Product name present but no ID
    0 pts: Empty or null

  Sub-criterion C — Dev Tracking (0–10 pts):
    10 pts: Azure DevOps ID linked with URL + state
     8 pts: ADO ID linked, no URL or state
     5 pts: TFS ID only
     2 pts: Jira ID only
     0 pts: All null

### DIMENSION 4: Reproducibility (0–20 points)
Evaluate `case.description`, `case.area`, `case.sub_area`, `case.version`.

  Sub-criterion A — Steps to Reproduce (0–10 pts):
    10 pts: Numbered steps, each with clear action and expected result
     8 pts: Steps present but some ambiguity
     5 pts: Description has enough context to understand the issue
     2 pts: Minimal description (1-2 lines)
     0 pts: Empty or null

  Sub-criterion B — Area / Sub-Area (0–5 pts):
    5 pts: Both area and sub-area defined
    3 pts: Area only
    0 pts: Neither

  Sub-criterion C — Version / Environment (0–5 pts):
    5 pts: Specific version or environment noted (e.g., "ClassicOneSite v2.4")
    2 pts: "ClassicOneSite" or generic platform only
    0 pts: Empty or null

### DIMENSION 5: Prioritization (0–10 points)
Evaluate `pme.priority`, `pme.escalation_status`, and consistency between them.

  Sub-criterion A — Priority Set (0–5 pts):
    5 pts: Priority assigned (P1–P4) and plausible given impact
    3 pts: Priority assigned but seems inconsistent with documented impact
    0 pts: No priority set

  Sub-criterion B — Status Appropriateness (0–5 pts):
    5 pts: Escalation status matches documented progress
    3 pts: Status slightly out of sync (e.g., "New" with ADO ID already linked)
    0 pts: Status missing or clearly wrong

### DIMENSION 6: Account Context (0–10 points)
Evaluate `ultimate_parent_account.*`.

  Sub-criterion A — Account & ACV (0–5 pts):
    5 pts: UPA name, ACV, and recurring ACV all present
    3 pts: Account name only
    0 pts: Empty

  Sub-criterion B — Customer Segment (0–5 pts):
    5 pts: Segment identified (Strategic, Maintenance, Growth, etc.) + CSM assigned
    3 pts: Segment only, no CSM
    1 pt: CSM only, no segment
    0 pts: Both empty

## Output Format
Return a JSON object exactly matching this structure:
{
  "score_total": <integer 0-100>,
  "quality_band": "<HIGH|MEDIUM|LOW|INSUFFICIENT>",
  "score_breakdown": {
    "problem_clarity": {"score": <int>, "max": 20, "sub": {"summary_specificity": <int>, "root_cause": <int>}, "notes": "<string>"},
    "business_impact": {"score": <int>, "max": 20, "sub": {"customer_impact": <int>, "business_impact": <int>, "business_value": <int>}, "notes": "<string>"},
    "technical_context": {"score": <int>, "max": 20, "sub": {"domain": <int>, "support_product": <int>, "dev_tracking": <int>}, "notes": "<string>"},
    "reproducibility": {"score": <int>, "max": 20, "sub": {"steps": <int>, "area": <int>, "version": <int>}, "notes": "<string>"},
    "prioritization": {"score": <int>, "max": 10, "sub": {"priority_set": <int>, "status_appropriate": <int>}, "notes": "<string>"},
    "account_context": {"score": <int>, "max": 10, "sub": {"account_acv": <int>, "segment": <int>}, "notes": "<string>"}
  },
  "missing_fields": ["<field_name>", ...],
  "flags": ["<warning string>", ...],
  "enrichment_checklist": ["<action item>", ...],
  "proceed_recommendation": "<PROCEED|PROCEED_WITH_WARNINGS|FLAG_FOR_ENRICHMENT|HALT>"
}

## Quality Band Thresholds
  80–100 → HIGH         → PROCEED
  60–79  → MEDIUM       → PROCEED_WITH_WARNINGS
  40–59  → LOW          → FLAG_FOR_ENRICHMENT
  0–39   → INSUFFICIENT → HALT
```

---

## Usage Example

```python
# Inside PME Evaluator Agent
result = invoke_skill("pme-scoring", pme_json=pme_work_item_json)
score = result["score_total"]
band  = result["quality_band"]
```
