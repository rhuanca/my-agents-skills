---
name: MCP REST Adapter
id: mcp-rest-adapter
description: |
  An MCP adapter skill that exposes a generic REST API interface to MCP tools.
  Provides standardized operations for GET/POST/PUT/DELETE, authentication handling,
  response validation, and error reporting. Designed to integrate with the project's
  `FastMCP` server as a drop-in adapter for REST-backed services.
version: 0.1.0
author: Databricks MCP Server Team
---

## Purpose

Provide a reusable, well-documented MCP skill that calls external REST APIs and
returns results in a consistent, MCP-friendly format. The skill implements a
generic REST client surface (get/create/update/delete) and examples for common
use-cases.

## Inputs (parameters)

- `method` (string): HTTP method to call. One of `GET`, `POST`, `PUT`, `DELETE`.
- `path` (string): Path or full URL of the resource (if the full URL is provided,
  `base_url` is ignored).
- `base_url` (string, optional): Base URL to prepend to `path` when `path` is a
  relative path.
- `headers` (object, optional): Additional HTTP headers to send.
- `params` (object, optional): Query parameters for `GET` requests.
- `json` (object, optional): JSON body to send for POST/PUT requests.
- `timeout` (number, optional): Request timeout in seconds (default: 30).

## Outputs

- `status` (int): HTTP status code returned by the API.
- `ok` (boolean): True when status is 2xx.
- `body` (object|string): Parsed JSON body when available, otherwise raw text.
- `error` (string|null): Error message on failure.

## Behavior and error handling

- Validates method and essential parameters.
- Parses JSON responses when possible; returns raw text otherwise.
- Returns structured error details on network failures or non-JSON responses.

## Example usage (MCP tool wrapper)

Use a project-local MCP server to expose the adapter's operations as MCP tools. Example (Python):

```py
from your_project.mcp_adapter.adapter import MCPRestAdapter
from your_project.server import mcp

adapter = MCPRestAdapter(base_url="https://api.example.com", default_headers={"Authorization": "Bearer ${TOKEN}"})

@mcp.tool()
def rest_call(method: str, path: str, base_url: str = None, headers: dict = None, params: dict = None, json: dict = None, timeout: int = 30) -> str:
  result = adapter.call(method=method, path=path, base_url=base_url, headers=headers, params=params, json=json, timeout=timeout)
  return str(result)
```

## Security and permissions

- The adapter does not store credentials by default; pass them via headers or
  environment variables. When deploying, ensure secrets are injected securely.

## Extensibility

- Implement custom response validators or transformers by subclassing
  `MCPRestAdapter` and overriding `_transform_response`.

## Notes

- This skill file is intentionally generic — when you provide a target REST API
  spec, replace the example paths and add specialized helper functions (e.g.,
  `get_user`, `create_job`) that map directly to your API endpoints.

## Modular Architecture (how this maps to the project)

This skill follows the modular architecture used across this repository. The
design breaks responsibilities into small, composable modules so they can be
reused and tested independently.

-- **Adapter package**: The core adapter lives in the `mcp_adapter` (project-local)
  package. Responsibilities:
  - `adapter.py`: `MCPRestAdapter` — a lightweight HTTP wrapper with structured
    responses and extension hooks (`_transform_response`). Use this to implement
    API-specific behaviors by subclassing.
  - `__init__.py`: Export the adapter class for consumers.
  - `README.md`: Package-level usage examples and notes.

-- **Server integration**: Expose adapter calls as MCP tools in the same style as
  other services (see `your_project/server.py`). Typical pattern:

  - Create a module-level adapter instance with sensible defaults.
  - Wrap calls with `@mcp.tool()` to expose them to MCP clients.
  - Keep thin wrappers in `server.py` that translate MCP tool args to adapter
    calls; business logic and HTTP details remain in the adapter or a service
    helper module.

- **Service helpers**: For API domains (e.g., Jobs, SQL, Workspace) provide a
  `services/*` module that offers high-level functions used by the MCP tool
  wrappers. This keeps `server.py` focused on exposure and routing.

- **Extension points**:
  - Subclass `MCPRestAdapter` to implement API-specific authentication flows,
    response validation, pagination helpers, or retry logic.
  - Add transformer/validator functions in `services/` to post-process responses
    before returning them to MCP callers.

- **Example tree (high-level)**

```
your_project/
├─ server.py                # MCP tool wrappers (exposes tools via FastMCP)
├─ mcp_adapter/
│  ├─ __init__.py           # exports MCPRestAdapter
│  ├─ adapter.py            # MCPRestAdapter implementation and hooks
│  └─ README.md             # usage and extension notes
├─ services/
│  ├─ sql_service.py        # high-level SQL helpers that use the adapter
│  ├─ jobs_service.py       # job-related helpers
│  └─ ...
└─ mcp_rest_adapter_skill.md # this skill description
```

## How to extend for a specific REST API

1. Create `your_project/services/<api>_service.py` with high-level
  helpers that call `MCPRestAdapter` or a subclass.
2. Optionally implement `MyApiAdapter(MCPRestAdapter)` to encapsulate auth,
   base URLs, and any common transforms.
3. Add thin MCP tool wrappers in `server.py` that call your service helpers and
   return MCP-friendly results.

Example adapter subclass skeleton:

```py
from your_project.mcp_adapter.adapter import MCPRestAdapter

class MyApiAdapter(MCPRestAdapter):
  def __init__(self, api_key: str, base_url: str = None):
    super().__init__(base_url=base_url, default_headers={"Authorization": f"Bearer {api_key}"})

  def _transform_response(self, response):
    # validate shape, raise informative errors, normalize fields
    return super()._transform_response(response)
```

This modular layout keeps HTTP concerns isolated, makes unit testing easier,
and mirrors the existing `services/*` structure in the project.

## Deployment & Transports

This skill and the adapter are transport-agnostic at the code level — the MCP
server's transport layer determines how MCP clients connect. Ensure your
adapter code follows these rules to work correctly with all supported
transports:

- Keep adapter functions stateless and idempotent (no global mutable state).
- Return small, serializable structures (strings, dicts) for `stdio` transport
  compatibility. For binary content, encode as Base64 when using `stdio`.
- Avoid streaming large raw byte streams from the adapter when using `stdio`;
  prefer returning an external link or chunked transfer through the
  `streamable-http` transport.

Supported transports and examples

- `stdio` — useful for local tooling and subprocess-based integrations. Run the
  server in stdio mode using your module entry point:

```bash
python -m your_project --transport stdio
```

- `streamable-http` — exposes MCP over HTTP with support for streaming
  responses. Run locally and listen on a host/port:

```bash
python -m your_project --transport streamable-http --host 0.0.0.0 --port 8000
```

Note on `uv` / `uvicorn`

The project uses `uvicorn` (often referred to as `uv`) as the ASGI server
when running the `streamable-http` transport in production or containerized
deployments. You can run the ASGI app directly with `uvicorn` for improved
performance and control over workers, logging, and reload behavior. Example:

```bash
uvicorn your_project.server:app --host 0.0.0.0 --port 8000 --workers 4
```

If your project entrypoint already configures and runs `uvicorn` when
`--transport streamable-http` is provided, prefer that wrapper for consistent
startup behavior.

Docker

Build and run the adapter service in Docker. The container accepts environment
variables to control transport and networking (examples below assume the
project's `Dockerfile` wires `MCP_TRANSPORT`, `MCP_PORT`, and `MCP_HOST` to the
module entrypoint):

```bash
docker build -t mcp-adapter .
docker run -e MCP_TRANSPORT=streamable-http -e MCP_PORT=8000 -e MCP_HOST=0.0.0.0 -p 8000:8000 mcp-adapter
```

Security and secrets in Docker

- Inject secrets via environment variables or mounted files; avoid baking
  credentials into the image. For example, pass `API_KEY` as an env var and
  have your adapter read it at runtime.

Notes on stream behavior

- When using `streamable-http`, the MCP server can forward streaming responses
  (chunked, SSE) to clients; design adapter helpers to surface incremental
  updates or provide an endpoint that supports streaming when necessary.


