# TFS MCP Server — Setup Guide

> **Purpose:** Configure the TFS/Azure DevOps MCP server that all PME pipeline agents use to access work items, test plans, and test cases.

---

## What is TFS MCP?

The TFS MCP (Model Context Protocol) server is a bridge between Claude agents and Azure DevOps / TFS. It exposes TFS REST API operations as structured tools that Claude can call, handling authentication, error handling, and response normalization automatically.

---

## Prerequisites

- Azure DevOps / TFS instance (cloud or on-premise)
- Personal Access Token (PAT) with the following scopes:
  - `Work Items (Read & Write)`
  - `Test Management (Read & Write)`
  - `Project and Team (Read)`
- Node.js 18+ or Python 3.10+ (depending on MCP server implementation)
- Claude Code CLI with MCP support

---

## Option 1: Azure DevOps MCP (Cloud)

### Install

```bash
# Using npm
npm install -g @modelcontextprotocol/server-azuredevops

# Or using pip if Python-based
pip install mcp-server-azuredevops
```

### Configuration

Add to your `.claude/mcp_config.json`:

```json
{
  "mcpServers": {
    "tfs": {
      "command": "mcp-server-azuredevops",
      "args": [],
      "env": {
        "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/YOUR_ORG",
        "AZURE_DEVOPS_PAT": "YOUR_PAT_TOKEN_HERE",
        "AZURE_DEVOPS_DEFAULT_PROJECT": "Consumer Solutions"
      }
    }
  }
}
```

---

## Option 2: On-Premise TFS (TFS 2018+)

### Configuration

```json
{
  "mcpServers": {
    "tfs": {
      "command": "mcp-server-tfs",
      "args": [],
      "env": {
        "TFS_COLLECTION_URL": "http://YOUR-TFS-SERVER:8080/tfs/DefaultCollection",
        "TFS_PAT": "YOUR_PAT_TOKEN_HERE",
        "TFS_AUTH_TYPE": "basic",
        "TFS_DEFAULT_PROJECT": "Consumer Solutions"
      }
    }
  }
}
```

---

## Option 3: Custom MCP Wrapper (if no package available)

If a pre-built TFS MCP server is not available for your environment, create a minimal wrapper:

### `tfs_mcp_server.py`

```python
"""
Minimal TFS MCP Server — wraps Azure DevOps REST API as MCP tools.
"""
from mcp.server import Server
from mcp.server.stdio import stdio_server
import httpx, os, json

ORG_URL = os.environ["AZURE_DEVOPS_ORG_URL"]
PAT     = os.environ["AZURE_DEVOPS_PAT"]
AUTH    = ("", PAT)

server = Server("tfs-mcp")

@server.tool()
async def tfs_get_work_item(id: str, project: str = None) -> dict:
    """Fetch a work item by ID."""
    url = f"{ORG_URL}/{project or '_apis'}/_apis/wit/workitems/{id}?$expand=all&api-version=7.0"
    async with httpx.AsyncClient() as client:
        r = await client.get(url, auth=AUTH)
        r.raise_for_status()
        return r.json()

@server.tool()
async def tfs_get_test_plans(project: str) -> list:
    """List all test plans for a project."""
    url = f"{ORG_URL}/{project}/_apis/testplan/plans?api-version=7.0"
    async with httpx.AsyncClient() as client:
        r = await client.get(url, auth=AUTH)
        r.raise_for_status()
        return r.json().get("value", [])

@server.tool()
async def tfs_get_test_suites(project: str, plan_id: int) -> list:
    """Get all test suites in a test plan."""
    url = f"{ORG_URL}/{project}/_apis/testplan/Plans/{plan_id}/suites?api-version=7.0"
    async with httpx.AsyncClient() as client:
        r = await client.get(url, auth=AUTH)
        r.raise_for_status()
        return r.json().get("value", [])

@server.tool()
async def tfs_get_test_cases(project: str, plan_id: int, suite_id: int) -> list:
    """Get all test cases in a test suite."""
    url = f"{ORG_URL}/{project}/_apis/testplan/Plans/{plan_id}/Suites/{suite_id}/TestCase?api-version=7.0"
    async with httpx.AsyncClient() as client:
        r = await client.get(url, auth=AUTH)
        r.raise_for_status()
        return r.json().get("value", [])

@server.tool()
async def tfs_create_test_case(project: str, suite_id: int, title: str,
                                steps: list, area_path: str, priority: int = 2) -> dict:
    """Create a new test case work item."""
    url = f"{ORG_URL}/{project}/_apis/wit/workitems/$Test Case?api-version=7.0"
    body = [
        {"op": "add", "path": "/fields/System.Title",       "value": title},
        {"op": "add", "path": "/fields/Microsoft.VSTS.Common.Priority", "value": priority},
        {"op": "add", "path": "/fields/System.AreaPath",    "value": area_path},
        {"op": "add", "path": "/fields/Microsoft.VSTS.TCM.Steps", "value": _format_steps(steps)},
    ]
    async with httpx.AsyncClient() as client:
        r = await client.post(url, auth=AUTH, json=body,
                              headers={"Content-Type": "application/json-patch+json"})
        r.raise_for_status()
        return r.json()

def _format_steps(steps: list) -> str:
    """Format steps list as TFS XML."""
    xml = '<steps id="0" last="0">'
    for i, s in enumerate(steps, 1):
        xml += f'<step id="{i}" type="ValidateStep">'
        xml += f'<parameterizedString isformatted="true">{s["action"]}</parameterizedString>'
        xml += f'<parameterizedString isformatted="true">{s.get("expected","")}</parameterizedString>'
        xml += '</step>'
    xml += '</steps>'
    return xml

if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(server))
```

### Register in MCP config

```json
{
  "mcpServers": {
    "tfs": {
      "command": "python",
      "args": ["C:/path/to/tfs_mcp_server.py"],
      "env": {
        "AZURE_DEVOPS_ORG_URL": "https://dev.azure.com/YOUR_ORG",
        "AZURE_DEVOPS_PAT": "YOUR_PAT_TOKEN"
      }
    }
  }
}
```

---

## Verify Connection

```bash
# Test MCP connection
claude --mcp-debug

# Or run a quick test
/pme-evaluate PME-100957 --format json
# Should return TFS work item data if connected
```

---

## PAT Token Permissions Required

| Permission | Scope | Required For |
|------------|-------|-------------|
| Work Items | Read | Fetch PME details |
| Work Items | Write | Update PME links |
| Test Management | Read | Fetch test plans/cases |
| Test Management | Write | Create test cases |
| Project & Team | Read | Resolve project names |

---

## Security Notes

- **Never commit PAT tokens** to source control
- Store in environment variables or a secrets manager
- Use minimum required scopes
- Rotate PAT tokens every 90 days
- For CI/CD: use a service account PAT, not a personal one
