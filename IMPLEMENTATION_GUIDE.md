# Implementation Guide — PME Agentic Flow

> Step-by-step guide to build, configure, and run the PME Evaluation & Test Coverage Pipeline.

---

## Phase 1: Environment Setup (Day 1)

### Step 1.1 — Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude --version   # verify
```

### Step 1.2 — Configure TFS MCP Server

```bash
# Install MCP server (choose one based on your TFS version)
npm install -g @modelcontextprotocol/server-azuredevops   # Azure DevOps cloud
# OR
pip install mcp-server-tfs                                 # On-premise TFS

# Configure
mkdir -p ~/.claude
cat > ~/.claude/mcp_config.json << 'EOF'
{
  "mcpServers": {
    "tfs": {
      "command": "mcp-server-azuredevops",
      "args": [],
      "env": {
        "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/YOUR_ORG",
        "AZURE_DEVOPS_PAT": "YOUR_PAT_TOKEN",
        "AZURE_DEVOPS_DEFAULT_PROJECT": "Consumer Solutions"
      }
    }
  }
}
EOF
```

### Step 1.3 — Verify TFS Connection

```bash
# Start Claude Code and test MCP
claude
> /pme-evaluate PME-100957 --format json
# Should return work item data from TFS
```

---

## Phase 2: Register Commands (Day 1–2)

Claude Code commands are registered as prompt files. Copy the command specs to your Claude config:

```bash
# Create commands directory
mkdir -p ~/.claude/commands

# Register the 3 PME commands
# (Copy the "Prompt Template" sections from each command .md file)

cat > ~/.claude/commands/pme-evaluate.md << 'PROMPT'
---
name: pme-evaluate
description: Fetch a PME from TFS and score its quality on a 100-pt rubric
---
[paste Prompt Template from commands/pme-evaluate.md]
PROMPT

cat > ~/.claude/commands/pme-coverage.md << 'PROMPT'
---
name: pme-coverage
description: Run full PME pipeline — evaluate, coverage analysis, test generation, report
---
[paste Prompt Template from commands/pme-coverage.md]
PROMPT

cat > ~/.claude/commands/pme-report.md << 'PROMPT'
---
name: pme-report
description: Re-generate or retrieve a PME evaluation report
---
[paste Prompt Template from commands/pme-report.md]
PROMPT
```

---

## Phase 3: Build the Agent Code (Days 2–5)

### Architecture: Claude Agent SDK

Each agent is a Claude API call with:
- A system prompt (from the agent .md files)
- Tool access (via MCP for TFS tools, standard tools for files)
- Input/output structured as JSON

### 3.1 — Project Structure

```
pme_pipeline/
├── main.py                  ← CLI entry point
├── orchestrator.py          ← Orchestrator agent
├── agents/
│   ├── pme_evaluator.py     ← PME Evaluator agent
│   ├── coverage_analyst.py  ← Coverage Analyst agent
│   ├── test_generator.py    ← Test Generator agent
│   └── report_generator.py  ← Report Generator agent
├── skills/
│   ├── pme_scoring.py       ← Scoring skill loader
│   ├── test_coverage.py     ← Coverage mapping skill loader
│   └── report_format.py     ← Report format skill loader
├── mcp/
│   └── tfs_client.py        ← TFS MCP tool wrappers
├── output/                  ← Generated reports
└── requirements.txt
```

### 3.2 — Orchestrator Implementation

```python
# orchestrator.py
import anthropic
from agents.pme_evaluator import evaluate_pme
from agents.coverage_analyst import analyze_coverage
from agents.test_generator import generate_tests
from agents.report_generator import generate_report

client = anthropic.Anthropic()

def run_pipeline(pme_id: str, mode: str = "coverage",
                 score_threshold: int = 60, tfs_write: bool = False) -> dict:
    """
    Run the full PME evaluation and coverage analysis pipeline.
    
    Args:
        pme_id: PME work item ID (e.g., "PME-100957")
        mode: "evaluate" | "coverage" | "report"
        score_threshold: Minimum score to proceed (default 60)
        tfs_write: Whether to create test cases in TFS
    
    Returns:
        Complete pipeline result dict
    """
    results = {"pme_id": pme_id, "stages": {}, "status": "RUNNING"}
    
    # Stage 1: Fetch from TFS
    print(f"  [1/4] Fetching {pme_id} from TFS...")
    pme_data = fetch_from_tfs(pme_id)
    results["stages"]["fetch"] = {"status": "OK", "data": pme_data}
    
    # Stage 2: Evaluate
    print(f"  [2/4] Evaluating PME quality...")
    eval_result = evaluate_pme(pme_data)
    results["stages"]["evaluate"] = eval_result
    print(f"        Score: {eval_result['score_total']}/100 [{eval_result['quality_band']}]")
    
    if mode == "evaluate":
        return finalize(results, eval_result, None, None)
    
    # Route based on score
    if eval_result["score_total"] < 40:
        results["status"] = "HALTED_LOW_SCORE"
        return generate_report(results, mode="enrichment_only")
    
    # Stage 3: Coverage analysis
    print(f"  [3/4] Analyzing test coverage...")
    coverage_result = analyze_coverage(pme_data, eval_result)
    results["stages"]["coverage"] = coverage_result
    
    # Stage 4: Generate tests if needed
    generated_suite = None
    if not coverage_result.get("suite_found") or coverage_result.get("gaps_found", 0) > 0:
        print(f"  [4/4] Generating test cases...")
        generated_suite = generate_tests(pme_data, eval_result, coverage_result, tfs_write)
        results["stages"]["generation"] = generated_suite
    
    # Stage 5: Report
    print(f"  [5/5] Generating report...")
    report = generate_report(results, eval_result, coverage_result, generated_suite)
    results["stages"]["report"] = report
    results["status"] = "COMPLETE"
    
    return results


def fetch_from_tfs(pme_id: str) -> dict:
    """Fetch PME work item from TFS via MCP with retry logic."""
    from mcp.tfs_client import TFSClient
    client_tfs = TFSClient()
    
    for attempt in range(3):
        try:
            return client_tfs.get_work_item(pme_id)
        except ConnectionError:
            if attempt == 2:
                print("        ⚠️ TFS unavailable — switching to offline mode")
                return load_from_csv(pme_id)
            import time; time.sleep(2 ** attempt)
```

### 3.3 — PME Evaluator Agent

```python
# agents/pme_evaluator.py
import anthropic, json
from skills.pme_scoring import PME_SCORING_SKILL

client = anthropic.Anthropic()

SYSTEM_PROMPT = """You are the PME Evaluator Agent..."""
# (full content from agents/02_pme_evaluator.md System Prompt section)

def evaluate_pme(pme_data: dict) -> dict:
    """Score a PME work item using the pme-scoring skill."""
    
    # Build prompt with skill template
    scoring_prompt = PME_SCORING_SKILL.format(pme_json=json.dumps(pme_data, indent=2))
    
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        system=SYSTEM_PROMPT,
        messages=[
            {"role": "user", "content": scoring_prompt}
        ]
    )
    
    # Parse structured JSON response
    result = json.loads(response.content[0].text)
    return result
```

### 3.4 — Coverage Analyst Agent

```python
# agents/coverage_analyst.py
import anthropic, json
from mcp.tfs_client import TFSClient
from skills.test_coverage import TEST_COVERAGE_MAPPING_SKILL

client = anthropic.Anthropic()
tfs    = TFSClient()

def analyze_coverage(pme_data: dict, eval_result: dict) -> dict:
    """Fetch test suite from TFS and analyze coverage gaps."""
    
    project = pme_data.get("pme", {}).get("dev_tracking", {}).get("azure_devops_project", "")
    
    # Fetch test plans from TFS
    test_plans = tfs.get_test_plans(project)
    
    if not test_plans:
        return {"suite_found": False, "reason": f"No test plans in project: {project}"}
    
    # Find relevant plan for this PME's product area
    relevant_plan = find_relevant_plan(test_plans, pme_data)
    
    if not relevant_plan:
        return {"suite_found": False, "reason": "No matching test plan for PME product area"}
    
    # Fetch test cases
    suites = tfs.get_test_suites(project, relevant_plan["id"])
    all_cases = []
    for suite in suites:
        cases = tfs.get_test_cases(project, relevant_plan["id"], suite["id"])
        all_cases.extend(cases)
    
    # Apply coverage mapping skill via Claude
    mapping_prompt = TEST_COVERAGE_MAPPING_SKILL.format(
        pme_json=json.dumps(pme_data, indent=2),
        score_total=eval_result["score_total"],
        quality_band=eval_result["quality_band"],
        missing_fields=eval_result["missing_fields"],
        test_cases=json.dumps(all_cases, indent=2)
    )
    
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=8192,
        messages=[{"role": "user", "content": mapping_prompt}]
    )
    
    return json.loads(response.content[0].text)
```

### 3.5 — CLI Entry Point

```python
# main.py
import argparse
from orchestrator import run_pipeline

def main():
    parser = argparse.ArgumentParser(description="PME Evaluation & Coverage Pipeline")
    parser.add_argument("pme_id",          help="PME ID (e.g., PME-100957)")
    parser.add_argument("--mode",          default="coverage",
                        choices=["evaluate","coverage","report"])
    parser.add_argument("--threshold",     type=int, default=60)
    parser.add_argument("--tfs-write",     action="store_true")
    parser.add_argument("--output-dir",    default="./pme_reports/")
    args = parser.parse_args()
    
    print(f"\n{'='*55}")
    print(f"  PME Pipeline — {args.pme_id}")
    print(f"{'='*55}\n")
    
    result = run_pipeline(
        pme_id=args.pme_id,
        mode=args.mode,
        score_threshold=args.threshold,
        tfs_write=args.tfs_write
    )
    
    if result["status"] == "COMPLETE":
        print(f"\n  ✅ Complete — reports saved to {args.output_dir}")
    else:
        print(f"\n  ⚠️  Pipeline ended: {result['status']}")

if __name__ == "__main__":
    main()
```

---

## Phase 4: Skills Implementation (Days 3–4)

Skills are stored as Python string constants loaded from the .md files:

```python
# skills/pme_scoring.py
PME_SCORING_SKILL = """
You are scoring a PME on a 100-point quality rubric.
[... full prompt from skills/pme_scoring.md ...]
"""
```

Or load dynamically:

```python
# skills/loader.py
from pathlib import Path

def load_skill(name: str) -> str:
    """Load skill prompt from .md file."""
    skill_dir = Path(__file__).parent.parent / "skills"
    md_file   = skill_dir / f"{name}.md"
    content   = md_file.read_text(encoding="utf-8")
    
    # Extract prompt from code block
    start = content.find("```\n") + 4
    end   = content.find("\n```", start)
    return content[start:end]
```

---

## Phase 5: Testing (Days 4–5)

### Unit Tests

```python
# tests/test_pme_scoring.py
from agents.pme_evaluator import evaluate_pme
import json

def test_high_quality_pme():
    pme = json.load(open("tests/fixtures/high_quality_pme.json"))
    result = evaluate_pme(pme)
    assert result["score_total"] >= 80
    assert result["quality_band"] == "HIGH"

def test_low_quality_pme():
    pme = json.load(open("tests/fixtures/minimal_pme.json"))
    result = evaluate_pme(pme)
    assert result["score_total"] < 40
    assert result["proceed_recommendation"] == "HALT"
```

### Integration Test

```bash
# Run full pipeline on a known PME
python main.py PME-100957 --mode coverage

# Verify outputs
ls pme_reports/
# PME-100957_report_2026-04-09.md  ← should exist
# PME-100957_report_2026-04-09.xlsx
```

---

## Phase 6: Deploy to CI/CD (Week 2)

### GitHub Actions Workflow

```yaml
# .github/workflows/pme_pipeline.yml
name: PME Coverage Analysis

on:
  workflow_dispatch:
    inputs:
      pme_ids:
        description: 'Comma-separated PME IDs'
        required: true

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run PME Pipeline
        env:
          ANTHROPIC_API_KEY:    ${{ secrets.ANTHROPIC_API_KEY }}
          AZURE_DEVOPS_ORG_URL: ${{ secrets.AZURE_DEVOPS_ORG_URL }}
          AZURE_DEVOPS_PAT:     ${{ secrets.AZURE_DEVOPS_PAT }}
        run: |
          pip install -r requirements.txt
          for PME_ID in $(echo "${{ inputs.pme_ids }}" | tr ',' ' '); do
            python main.py $PME_ID --mode coverage --tfs-write
          done
      
      - name: Upload Reports
        uses: actions/upload-artifact@v4
        with:
          name: pme-reports
          path: pme_reports/
```

---

## Implementation Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Phase 1: Setup | Day 1 | TFS MCP connected, Claude CLI working |
| Phase 2: Commands | Day 1–2 | `/pme-evaluate` and `/pme-coverage` working in CLI |
| Phase 3: Agents | Day 2–5 | All 5 agents implemented and callable |
| Phase 4: Skills | Day 3–4 | 3 skills loaded and tested |
| Phase 5: Testing | Day 4–5 | Unit + integration tests passing |
| Phase 6: CI/CD | Week 2 | GitHub Actions pipeline running |

---

## Decision Log

| Decision | Chosen | Rationale |
|----------|--------|-----------|
| Agent framework | Claude Agent SDK | Native tool use, multi-agent support |
| Orchestration | Python + Claude API | Full control over routing logic |
| Scoring | LLM skill (not rules) | Rubric requires judgment on narrative fields |
| TFS integration | MCP server | Centralized auth, shared across all agents |
| Report format | MD + Excel | MD for humans, Excel for filtering/sharing |
| Test generation | Agent (not template) | Context-aware generation from PME narrative |
| Entry points | Slash commands | Discoverable, parameterized, IDE-friendly |
