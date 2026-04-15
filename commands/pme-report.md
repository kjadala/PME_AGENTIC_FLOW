# Command: /pme-report

> **Type:** User-invocable slash command  
> **Purpose:** Re-generate or retrieve the report from the last pipeline run, or create a report from existing pipeline outputs without re-running analysis  
> **Use when:** You want to reformat, re-export, or share a report from a previous run

---

## Usage

```bash
/pme-report                              # Re-generate report from last run
/pme-report PME-100957                   # Re-generate for specific PME
/pme-report PME-100957 --format excel    # Excel only
/pme-report PME-100957 --format md       # Markdown only
/pme-report PME-100957 --share           # Output to clipboard-ready format
/pme-report --list                       # List all available reports
```

---

## Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `pme_id` | No | Last run | PME ID to re-generate report for |
| `--format` | No | `both` | `both` \| `md` \| `excel` \| `json` |
| `--output-dir` | No | `./pme_reports/` | Output directory |
| `--share` | No | false | Format for sharing (no file paths, clean tables) |
| `--section` | No | all | `summary` \| `scorecard` \| `coverage` \| `tests` \| `recommendations` |
| `--list` | No | false | List all available cached reports |

---

## Prompt Template

```
Re-generate or retrieve a PME evaluation report.

PME ID: {pme_id}
Format: {format}
Section: {section}
Share mode: {share}

Steps:
1. Check if cached pipeline results exist for {pme_id} in {output_dir}
2. If cached: load and re-apply report-format skill to regenerate
3. If not cached: prompt user to run /pme-coverage {pme_id} first
4. Output in requested format

Do NOT re-run the evaluation or TFS fetch — use cached results only.
Apply report-format skill for consistent formatting.

If --share flag: remove file paths, run timestamps, and internal IDs.
Format tables for clean copy-paste into email/Slack/Confluence.
```

---

## List Mode Output

```
/pme-report --list

Available Reports (./pme_reports/)
═══════════════════════════════════════════════════════════════════════
  PME-ID          Score  Coverage  Generated        Files
  ─────────────────────────────────────────────────────────────────
  PME-100957       78/100   50%   2026-04-09 14:32  .md .xlsx
  PME-101427       62/100   73%   2026-04-09 11:15  .md .xlsx
  PME-102873       45/100   N/A   2026-04-08 16:44  .md
  PME-103078       81/100   91%   2026-04-08 09:30  .md .xlsx

To re-generate: /pme-report <PME-ID>
```
