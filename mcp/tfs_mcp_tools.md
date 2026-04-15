# TFS MCP Tools — Reference

> Complete reference of all TFS MCP tools available to PME pipeline agents.

---

## Tool Index

| Tool Name | Used By | Purpose |
|-----------|---------|---------|
| `tfs_get_work_item` | Orchestrator, Evaluator | Fetch any work item by ID |
| `tfs_get_work_item_links` | Evaluator | Get items linked to a work item |
| `tfs_get_comments` | Evaluator | Get work item discussion |
| `tfs_search_work_items` | Orchestrator | Find PMEs by project/query |
| `tfs_get_test_plans` | Coverage Analyst | List test plans for project |
| `tfs_get_test_suites` | Coverage Analyst | Get suites in a plan |
| `tfs_get_test_cases` | Coverage Analyst | Get cases in a suite |
| `tfs_search_test_cases` | Coverage Analyst | Search cases by keyword |
| `tfs_create_test_case` | Test Generator | Create new test case |
| `tfs_create_test_suite` | Test Generator | Create new suite in plan |
| `tfs_update_work_item` | Orchestrator | Link tests back to PME |

---

## Tool Specifications

### `tfs_get_work_item`

Fetches a complete work item including all fields, history, and attached files.

**Parameters:**
```json
{
  "id": "string | number",
  "project": "string (optional)",
  "expand": "string (default: 'all')"
}
```

**Returns:**
```json
{
  "id": 12345,
  "rev": 3,
  "fields": {
    "System.Id": 12345,
    "System.Title": "PME-100957 Lead2Lease Notifications filter",
    "System.State": "Active",
    "System.AssignedTo": "John Smith",
    "System.AreaPath": "Consumer Solutions\\Lead Management Program",
    "System.IterationPath": "Consumer Solutions\\2026 Q2",
    "Microsoft.VSTS.Common.Priority": 3,
    "Microsoft.VSTS.Common.Severity": "3 - Medium",
    "System.Description": "<full HTML description>",
    "System.Tags": "PME; Lead2Lease; Notifications",
    "System.CreatedDate": "2019-05-30T20:28:55Z",
    "System.ChangedDate": "2026-03-17T16:55:47Z"
  },
  "relations": [...],
  "_links": {...}
}
```

**Used by:** Orchestrator (initial fetch), PME Evaluator (full context)

---

### `tfs_get_work_item_links`

Returns all work items linked to a given item (parent features, child bugs, related items, test cases).

**Parameters:**
```json
{
  "id": "string | number",
  "project": "string (optional)",
  "link_type": "string (optional) — 'Related' | 'Parent' | 'Child' | 'TestedBy'"
}
```

**Returns:**
```json
{
  "relations": [
    {
      "rel": "System.LinkTypes.Related",
      "url": "https://dev.azure.com/org/project/_apis/wit/workitems/9876",
      "attributes": {
        "id": 9876,
        "name": "TC-8834: L2L Notifications default filter test",
        "isLocked": false
      }
    }
  ]
}
```

**Used by:** PME Evaluator (check for linked test cases, parent features)

---

### `tfs_get_comments`

Returns the discussion thread (comments) on a work item.

**Parameters:**
```json
{
  "id": "string | number",
  "project": "string (optional)",
  "top": "number (default: 50)"
}
```

**Returns:** Array of comment objects with author, date, and text.

**Used by:** PME Evaluator (additional context for scoring)

---

### `tfs_search_work_items`

Execute a WIQL (Work Item Query Language) query to find work items.

**Parameters:**
```json
{
  "project": "string",
  "wiql": "string — WIQL query",
  "top": "number (default: 50)"
}
```

**Example WIQL:**
```sql
SELECT [System.Id], [System.Title], [System.State]
FROM WorkItems
WHERE [System.WorkItemType] = 'Test Case'
AND [System.AreaPath] UNDER 'Consumer Solutions\Lead Management Program'
AND [System.State] <> 'Removed'
ORDER BY [System.ChangedDate] DESC
```

**Used by:** Coverage Analyst (find test cases for a product area)

---

### `tfs_get_test_plans`

List all test plans for a project.

**Parameters:**
```json
{
  "project": "string",
  "owner": "string (optional)",
  "is_active": "boolean (default: true)"
}
```

**Returns:**
```json
{
  "value": [
    {
      "id": 4521,
      "name": "Lead Management Program — Regression Suite",
      "project": {"name": "Consumer Solutions"},
      "area": {"name": "Lead Management Program"},
      "iteration": "Consumer Solutions\\2026 Q2",
      "owner": {"displayName": "QA Team"}
    }
  ],
  "count": 3
}
```

**Used by:** Coverage Analyst (find relevant test plan for PME's project)

---

### `tfs_get_test_suites`

Get all test suites within a test plan.

**Parameters:**
```json
{
  "project": "string",
  "plan_id": "number",
  "include_children": "boolean (default: true)"
}
```

**Returns:** Array of test suite objects with IDs, names, and test case counts.

**Used by:** Coverage Analyst (drill into plan structure)

---

### `tfs_get_test_cases`

Get all test cases in a specific test suite.

**Parameters:**
```json
{
  "project": "string",
  "plan_id": "number",
  "suite_id": "number",
  "include_steps": "boolean (default: true)"
}
```

**Returns:**
```json
{
  "value": [
    {
      "testCase": {
        "id": "8834",
        "name": "L2L Notifications default filter test"
      },
      "pointAssignments": [...],
      "workItem": {
        "id": 8834,
        "fields": {
          "System.Title": "L2L Notifications default filter test",
          "System.State": "Active",
          "Microsoft.VSTS.Common.Priority": 3,
          "System.AreaPath": "Consumer Solutions\\Lead Management Program\\L2L",
          "Microsoft.VSTS.TCM.Steps": "<steps>...</steps>"
        }
      }
    }
  ]
}
```

**Used by:** Coverage Analyst (semantic comparison with PME scenarios)

---

### `tfs_search_test_cases`

Search test cases by keyword within a project or area.

**Parameters:**
```json
{
  "project": "string",
  "keyword": "string",
  "area_path": "string (optional)",
  "top": "number (default: 50)"
}
```

**Used by:** Coverage Analyst (supplement suite search with keyword lookup)

---

### `tfs_create_test_case`

Create a new test case work item in TFS.

**Parameters:**
```json
{
  "project": "string",
  "title": "string",
  "area_path": "string",
  "iteration_path": "string (optional)",
  "priority": "number (1-4)",
  "steps": [
    {"action": "string", "expected": "string"}
  ],
  "preconditions": "string (optional)",
  "tags": "string (optional, semicolon-separated)",
  "linked_pme_id": "number (optional)"
}
```

**Returns:** Created work item object with TFS ID.

**Used by:** Test Generator (create test cases in TFS)

---

### `tfs_create_test_suite`

Create a new test suite under an existing test plan.

**Parameters:**
```json
{
  "project": "string",
  "plan_id": "number",
  "name": "string",
  "suite_type": "string — 'StaticTestSuite' | 'DynamicTestSuite'",
  "parent_suite_id": "number (optional)"
}
```

**Returns:** Created test suite object with ID.

**Used by:** Test Generator (create suite container before adding cases)

---

### `tfs_update_work_item`

Update fields on an existing work item (e.g., link a generated test suite back to the PME).

**Parameters:**
```json
{
  "id": "number",
  "project": "string (optional)",
  "fields": {
    "System.Tags": "string",
    "System.State": "string"
  },
  "relations_to_add": [
    {"rel": "Microsoft.VSTS.Common.TestedBy", "url": "..."}
  ]
}
```

**Used by:** Orchestrator (link generated tests back to PME work item)

---

## Error Handling

All tools follow this error response pattern:

```json
{
  "error": true,
  "code": "WORK_ITEM_NOT_FOUND | AUTH_FAILED | RATE_LIMITED | TFS_UNAVAILABLE",
  "message": "Human-readable error description",
  "retry_after": 30
}
```

The Orchestrator Agent handles all errors:
- `WORK_ITEM_NOT_FOUND`: Check PME ID, fallback to CSV
- `AUTH_FAILED`: Re-check PAT token configuration
- `RATE_LIMITED`: Wait `retry_after` seconds, retry
- `TFS_UNAVAILABLE`: Switch to offline mode after 3 retries
