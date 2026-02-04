# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Google Workspace MCP Server - A comprehensive Model Context Protocol (MCP) server providing natural language control over Google Workspace APIs (Gmail, Drive, Docs, Sheets, Calendar, Chat, Forms, Slides, Tasks, Contacts, Apps Script). Built with FastMCP and supports both OAuth 2.0 (single-user) and OAuth 2.1 (multi-user) authentication modes.

## Development Commands

### Setup & Running
```bash
# Install dependencies and run (recommended)
uv run main.py

# Run with specific transport mode
uv run main.py --transport stdio           # Default, for MCP clients
uv run main.py --transport streamable-http # For HTTP/web interfaces

# Run with authentication modes
uv run main.py --single-user              # Single-user OAuth 2.0 mode
MCP_ENABLE_OAUTH21=true uv run main.py    # Multi-user OAuth 2.1 mode
```

### CLI Mode
The server includes a CLI mode for direct tool invocation (used by coding agents):
```bash
# List all available tools
uv run main.py --cli

# Run a specific tool (reads JSON args from stdin)
uv run main.py --cli search_gmail_messages

# Run with inline arguments
uv run main.py --cli search_gmail_messages --args '{"query": "subject:test", "max_results": 5}'

# Get tool help
uv run main.py --cli search_gmail_messages --help
```

### Testing
```bash
# Run all tests
uv run pytest

# Run tests for specific service
uv run pytest tests/gappsscript/test_apps_script_tools.py

# Run with verbose output
uv run pytest -v
```

### Code Quality
```bash
# Format code
uv run ruff format .

# Lint code
uv run ruff check .
```

## Architecture Overview

### Core Architecture Pattern

**Service Decorator System**: The codebase uses a sophisticated decorator pattern for automatic Google service injection and authentication:

```python
@server.tool()  # Registers as MCP tool
@handle_http_errors("tool_name", is_read_only=True, service_type="gmail")
@require_google_service("gmail", "gmail_read")  # Injects authenticated service
async def your_tool(service: Any, user_google_email: str, param: str):
    """Tool implementation"""
    result = await asyncio.to_thread(service.messages().list(...).execute)
    return result
```

**Multi-Service Pattern**: For tools requiring multiple services:
```python
@require_multiple_services([
    {"service_type": "drive", "scopes": "drive_read", "param_name": "drive_service"},
    {"service_type": "docs", "scopes": "docs_read", "param_name": "docs_service"}
])
async def multi_service_tool(drive_service, docs_service, user_google_email: str):
    # Both services automatically injected with proper authentication
```

### Key Components

**auth/** - Authentication System
- `service_decorator.py`: Core decorator providing automatic service injection with 30-minute TTL caching
- `scopes.py`: Centralized OAuth scope definitions and SCOPE_GROUPS mapping
- `oauth_config.py`: Unified OAuth configuration (OAuth 2.0 vs 2.1, redirect URIs, transport modes)
- `credential_store.py`: Abstract credential storage API with LocalDirectoryCredentialStore implementation
- `google_auth.py`: Google OAuth flow implementation
- `mcp_session_middleware.py`: FastMCP session tracking for multi-user OAuth 2.1
- `oauth21_session_store.py`: Session persistence for OAuth 2.1 with pluggable backends (file, memory, Valkey)

**core/** - Server Infrastructure
- `server.py`: FastMCP server initialization with SecureFastMCP subclass for OAuth 2.1 middleware
- `tool_registry.py`: Dynamic tool registration and filtering
- `tool_tier_loader.py`: Loads tools by tier (core/extended/complete) from tool_tiers.yaml
- `tool_tiers.yaml`: YAML configuration defining which tools belong to each tier per service
- `cli_handler.py`: CLI mode implementation for direct tool invocation
- `context.py`: Request context management (session IDs, user info)
- `utils.py`: Shared utilities (error handling, Office XML parsing, credentials directory checks)

**g{service}/** - Service-Specific Modules
Each Google Workspace service has its own module with tools and helpers:
- `gmail/`, `gdrive/`, `gdocs/`, `gsheets/`, `gcalendar/`, `gchat/`, `gforms/`, `gslides/`, `gtasks/`, `gcontacts/`, `gsearch/`, `gappsscript/`
- Pattern: `{service}_tools.py` contains MCP tool implementations
- Complex services have additional helpers: `gdocs/docs_helpers.py`, `gdocs/docs_structure.py`, `gdocs/managers/`

### Authentication Flow

1. **OAuth 2.0 (Single-user)**: Traditional OAuth with local credential storage
   - Flow: Tool call → Check local credentials → If missing/expired, start OAuth flow → Store credentials → Retry tool call
   - Storage: `~/.google_workspace_mcp/credentials/` or `GOOGLE_MCP_CREDENTIALS_DIR`

2. **OAuth 2.1 (Multi-user)**: Modern OAuth with session-based authentication
   - Flow: MCP client sends bearer token → Middleware extracts session → Validates credentials → Injects into context
   - Requires: `MCP_ENABLE_OAUTH21=true` and proper FastMCP Google provider configuration
   - Storage backends: File (default), Memory, or Valkey/Redis
   - Stateless mode: `WORKSPACE_MCP_STATELESS_MODE=true` disables local credential storage

3. **Transport-Aware Callbacks**:
   - Stdio mode: Starts minimal HTTP server for OAuth callback (port 8000 by default)
   - HTTP mode: Uses existing HTTP server for callbacks

### Tool Tier System

Tools are organized in three tiers (defined in `core/tool_tiers.yaml`):
- **core**: Essential tools for basic functionality
- **extended**: Additional useful tools
- **complete**: Advanced/specialized tools

Configure enabled tiers via:
```bash
WORKSPACE_MCP_TIER=core              # Only core tools
WORKSPACE_MCP_TIER=extended          # Core + extended
WORKSPACE_MCP_TIER=complete          # All tools (default)
```

### Service Discovery & API Enablement

The server includes automatic Google API enablement checking and guidance (`core/api_enablement.py`). When a tool requires an API that isn't enabled, it provides the user with direct links to enable it in Google Cloud Console.

### Scope Management

Centralized in `auth/scopes.py`:
- Individual scope constants (e.g., `GMAIL_READONLY_SCOPE`, `DRIVE_FILE_SCOPE`)
- `SCOPE_GROUPS`: Maps service operations to required scopes
- `SCOPES`: Combined list of all scopes used by enabled tools
- Dynamic scope calculation based on enabled tool tiers

## Adding New Tools

1. Create or edit the appropriate `g{service}/{service}_tools.py` file
2. Use the decorator pattern:
```python
@server.tool()
@handle_http_errors("new_tool", is_read_only=False, service_type="gmail")
@require_google_service("gmail", "gmail_modify")  # service + scope group from SCOPE_GROUPS
async def new_tool(service: Any, user_google_email: str, param: str):
    """
    Tool description for MCP clients.

    Args:
        param: Parameter description

    Returns:
        Result description
    """
    result = await asyncio.to_thread(service.api_method().execute)
    return result  # Return native Python objects (dicts, lists, strings)
```
3. If new scopes needed, add to `auth/scopes.py` SCOPE_GROUPS
4. Add tool to appropriate tier in `core/tool_tiers.yaml`
5. Import the module in `main.py` to register tools

## Testing

Tests use pytest with mocked Google API clients:
- Tests import internal `_impl` functions (not decorated versions)
- Mock the `service` parameter with unittest.mock.Mock
- Test return values and API call patterns
- See `tests/gappsscript/test_apps_script_tools.py` for examples

## Configuration

Key environment variables:
- `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`: OAuth credentials
- `MCP_ENABLE_OAUTH21`: Enable OAuth 2.1 multi-user mode (default: false)
- `MCP_SINGLE_USER_MODE`: Enable single-user convenience mode (default: false)
- `WORKSPACE_MCP_STATELESS_MODE`: Disable local credential storage for OAuth 2.1 (default: false)
- `WORKSPACE_MCP_TIER`: Tool tier to load (core/extended/complete, default: complete)
- `GOOGLE_MCP_CREDENTIALS_DIR`: Custom credentials storage path
- `WORKSPACE_MCP_BASE_URI`: Base URI for OAuth callbacks (default: http://localhost)
- `PORT` or `WORKSPACE_MCP_PORT`: Server port (default: 8000)
- `WORKSPACE_EXTERNAL_URL`: External URL for reverse proxy scenarios
- `WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND`: OAuth 2.1 storage backend (file/memory/valkey)

## Entry Points

- `main.py`: Primary entry point with CLI support
- `fastmcp_server.py`: FastMCP Cloud-specific entry point (enforces OAuth 2.1 + stateless defaults)

## Important Notes

- **Never commit**: `.env`, `client_secret.json`, `.credentials/` directory
- **Error handling**: `@handle_http_errors` decorator provides consistent error responses
- **Service caching**: Services cached for 30 minutes to reduce auth overhead
- **Async pattern**: All API calls wrapped with `asyncio.to_thread()` for non-blocking execution
- **Return types**: Return native Python objects (dict/list/str), not JSON strings
- **Scope minimization**: Tools request only necessary OAuth scopes via SCOPE_GROUPS
