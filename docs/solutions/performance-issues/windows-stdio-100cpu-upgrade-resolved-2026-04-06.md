---
title: Windows stdio 100% CPU busy-wait resolved after FastMCP 3.1.1 upgrade
date: 2026-04-06
category: performance-issues
module: server
problem_type: performance_issue
component: tooling
symptoms:
  - sustained 100% CPU usage on single core while server idle
  - Windows-specific (Linux/macOS unaffected)
  - occurs immediately on server startup via stdio transport
  - process accumulates CPU seconds with zero client interaction
root_cause: async_timing
resolution_type: dependency_update
severity: high
tags:
  - windows
  - cpu
  - stdio
  - fastmcp
  - anyio
  - dependency-upgrade
---

# Windows stdio 100% CPU busy-wait resolved after FastMCP 3.1.1 upgrade

## Problem

MCP servers using FastMCP's `mcp.run()` with stdio transport experienced sustained 100% CPU usage on a single core while completely idle on Windows. This affected all Windows MCP servers launched by Claude Desktop and caused significant power consumption across multiple configured servers.

## Symptoms

- Single CPU core pinned at ~100% utilization
- Occurs when the MCP server is idle (no messages being processed)
- Windows-specific: Linux/macOS use epoll/kqueue which block efficiently
- Process consuming 200-300+ CPU seconds while showing no user activity
- Manifests immediately on server startup, persists indefinitely

## What Didn't Work

The original bug report (`docs/bugs/WINDOWS_STDIO_100CPU_BUSY_WAIT.md`) identified three proposed solutions, none of which were implemented:

1. **Patching stdin_reader with sleep** -- Required modifying the upstream `mcp` SDK package (`mcp/server/stdio.py`), not something mcp-alchemy could do unilaterally.
2. **Win32 native async stdin** -- Proposed replacing `anyio.wrap_file()` with Windows native overlapped I/O, but required significant upstream refactoring.
3. **Buffered memory streams** -- Changing `anyio.create_memory_object_stream(0)` to capacity 1 was deemed low-impact and still required an upstream change.

All three solutions targeted the `mcp` SDK, which mcp-alchemy imports but cannot patch.

## Solution

Upgrade FastMCP from 2.14.5 to 3.1.1. No code changes required.

**Before (`pyproject.toml`):**
```toml
dependencies = [
    "fastmcp==2.14.5",
    "hatchling>=1.28.0",
    "mcp[cli]>=1.26.0",
    "pyodbc>=5.3.0",
    "sqlalchemy>=2.0.46",
]
```

**After (`pyproject.toml`):**
```toml
dependencies = [
    "fastmcp==3.1.1",
    "hatchling>=1.28.0",
    "pyodbc>=5.3.0",
    "sqlalchemy>=2.0.46",
]
```

Then run `uv sync` to regenerate the lockfile.

Key transitive dependency changes:
- Added: `aiofile==3.9.0`, `caio==0.9.25` (C-based async I/O), `watchfiles==1.1.1`
- Removed: `diskcache`, `fakeredis`, `croniter`, `shellingham`, `typer`, and others
- `mcp[cli]>=1.26.0` removed as redundant (FastMCP 3 includes `mcp` transitively)

## Why This Works

The underlying `mcp` SDK code is unchanged -- `mcp/server/stdio.py` still uses `anyio.wrap_file()` + `async for line in stdin`, and `mcp` remains at v1.26.0 with `anyio` at v4.7.0.

Yet the symptom is definitively resolved. The likely mechanism is that new transitive dependencies (particularly `caio==0.9.25`, a C-based async I/O library, and `aiofile==3.9.0`) changed how async file operations behave on Windows internally, bypassing the tight polling loop. The exact mechanism is unconfirmed, but the empirical result is clear:

```
Before upgrade: ~100% CPU per idle process
After upgrade:  0.0-0.5% CPU per idle process (4 processes, ~15 min runtime)
```

Verification data:
```
PID 60252: 0.0s CPU over 886s = 0.0% CPU  (venv python)
PID 5840:  4.3s CPU over 886s = 0.5% CPU  (uv python)
PID 46792: 0.1s CPU over 886s = 0.0% CPU  (uv cached entry point)
PID 35536: 4.2s CPU over 886s = 0.5% CPU  (uv cached entry point)
```

## Prevention

- **Keep FastMCP up to date**: Transitive dependency improvements in newer versions may further optimize Windows stdio handling.
- **Monitor upstream `mcp` SDK**: Watch https://github.com/modelcontextprotocol/python-sdk for direct Windows stdin fixes (issues #1333, #526 are related).
- **Test Windows server startup**: When upgrading dependencies, verify CPU usage of idle MCP servers on Windows as a regression check.

## Related Issues

- `docs/bugs/WINDOWS_STDIO_100CPU_BUSY_WAIT.md` -- Original bug diagnosis and root cause analysis
- https://github.com/modelcontextprotocol/python-sdk/issues/1333 -- Unbuffered memory streams in stdio_server
- https://github.com/modelcontextprotocol/python-sdk/issues/526 -- MCP Server on Windows never exits
