# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP Alchemy is a Model Context Protocol (MCP) server that connects LLM clients (like Claude Desktop) to databases via SQLAlchemy. It exposes database introspection and query execution as MCP tools. Originally by Rune Kaagaard, this fork adds MS SQL Server extended properties and stored procedure support.

## Commands

```bash
# Install dependencies
uv sync

# Install a database driver (e.g., for SQL Server)
uv pip install pymssql

# Run the MCP server locally
uv run -m mcp_alchemy.server main

# Run tests (requires tests/Chinook_Sqlite.sqlite)
DB_URL="sqlite:///tests/Chinook_Sqlite.sqlite" uv run python tests/test.py
```

There is no pytest setup -- tests use a custom `test_func` runner in `tests/test.py` that compares function output against expected string constants.

## Architecture

The entire server is three files:

- **`mcp_alchemy/server.py`** -- The FastMCP server. Defines four MCP tools (`list_database_resources`, `filter_database_resources`, `schema_definitions`, `execute_query`), connection management with pooling/retry, and result formatting. Module-level globals (`ENGINE`, `EXECUTE_QUERY_MAX_CHARS`, `CLAUDE_LOCAL_FILES_PATH`) are initialized at import time; tests mutate them via `tests_set_global()`.

- **`mcp_alchemy/mssql_metadata.py`** -- SQL Server-specific helpers that query `sys.extended_properties` for table/column descriptions and documented stored procedures. Imported with a try/except fallback so the server works without SQL Server.

- **`tests/test.py`** -- Imports `*` from `mcp_alchemy.server` and tests against a SQLite Chinook database. Expected outputs are string constants compared with exact string equality.

`docs/solutions/` contains documented solutions to past problems, organized by category with YAML frontmatter (`module`, `tags`, `problem_type`). Relevant when debugging or upgrading dependencies.

## Key Design Details

- The server uses **FastMCP** (not raw MCP SDK) to register tools via `@mcp.tool()` decorators.
- Connection pooling is configured for long-running MCP servers: `pool_pre_ping=True`, `pool_size=1`, `pool_recycle=3600`, `AUTOCOMMIT` isolation.
- Query results use a vertical format (one field per line) with smart truncation at `EXECUTE_QUERY_MAX_CHARS` (default 4000).
- Optional `CLAUDE_LOCAL_FILES_PATH` integration saves full result sets as JSON files for large dataset analysis.
- The entry point is `mcp-alchemy` (defined in `[project.scripts]`), which calls `mcp_alchemy.server:main`.

## Environment Variables

- `DB_URL` (required) -- SQLAlchemy database URL
- `DB_ENGINE_OPTIONS` -- JSON string of additional SQLAlchemy engine kwargs
- `EXECUTE_QUERY_MAX_CHARS` -- Max output chars (default 4000)
- `CLAUDE_LOCAL_FILES_PATH` -- Directory for full result set JSON files
