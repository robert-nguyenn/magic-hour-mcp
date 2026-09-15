# Magic Hour MCP integration handoff

Use this when mounting the server into an existing FastAPI app.

For a numbered walkthrough, use `docs/detailed-step-by-step-integration.md`.
For instructions written for a coding agent, use `docs/ai-agent-go-live-instructions.md`.

## ChatGPT Business OAuth handoff (September 2026)

- Review branch: `chatgpt-integration`
- Stable test URL: `https://magic-hour-mcp-oauth-test.vercel.app`
- OAuth support includes protected-resource and authorization-server discovery,
  DCR, authorization-code/refresh-token grants, PKCE S256, public-client token
  authentication (`none`), and `mcp` / `offline_access` scopes.
- The server is OAuth-only; `/.well-known/openid-configuration` returns `404`.
  Do not add a placeholder OIDC document or userinfo endpoint.
- Unauthenticated valid MCP JSON-RPC discovery requests may reach FastMCP so a
  client can initialize and list tools. Actual `tools/call` requests remain
  challenged through OAuth middleware.

### September 15 update: server fixes and retest

The earlier conclusion that the zero-action draft was exclusively a ChatGPT
platform blocker was premature. Rhythm's commit `9aa762f` fixes two server-side
failures: request-body replay incorrectly signalled a client disconnect (empty
SSE responses), and tool listing assigned an unsupported SDK `securitySchemes`
field. The branch includes that commit and pins `fastmcp==3.4.7`.

Local validation also exposed two SDK naming mismatches in structured-error
handling. The SDK uses `McpError` and `request_handlers`, not `MCPError` and
`_request_handlers`. These are corrected so the server starts and invalid tool
arguments are rejected locally. Tests explicitly prohibit outgoing HTTP requests
for malformed tool calls.

The previous reproduction showed successful discovery and DCR (`201`) followed
by an empty action list. Support case **14263640** remains useful historical
context, but does not establish the cause or prove the server was correct.
The absence of a separate Scan Tools button alone is not a conclusive failure.

Next: deploy this branch to the Vercel **Preview/test** project, confirm that
`https://magic-hour-mcp-oauth-test.vercel.app` points to the new deployment, then
recreate the ChatGPT OAuth app. Follow the Connect/Refresh controls available in
that workspace, complete authorization, and verify that actions appear. This is
Rhythm's suggested retest path; success still requires an actual workspace test.
Do not publish a zero-action draft or merge PR #1 into main as part of this update.

### Local verification

September 15 results: all 55 tests pass with FastMCP 3.4.7 on both MCP SDK
1.29.1 (existing environment) and 1.30.0 (fresh Python 3.12 environment).
The fresh environment passes `pip check`; the web typecheck/build also passes.
The existing frontend dependencies report two moderate npm audit advisories;
those are outside this focused Python compatibility change.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e .
npm --prefix web ci
npm --prefix web run build
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -m pip check
```

The web build is required before the HTTP-view/asset tests. Tests use Python's
built-in `unittest`; installing `pytest` is unnecessary. The FastMCP pin does not
lock every transitive dependency. Record resolved SDK versions with test results
when upgrading; avoid inferring deployment health from an installed version alone.

## What to mount

- Import `app` from `mcp_magichour.server`
- Mount it at `/mcp`
- Final MCP endpoint: `/mcp/`

The internal FastMCP app is intentionally configured for `/`, not `/mcp`.

## What this server generates

At startup, the server reads `docs/openapi.json` and builds tools with `FastMCP.from_openapi()`.

OpenAPI `operationId` values are normalized to descriptive snake_case tool names. Examples:

- `video_assets_generate_presigned_url`
- `ai_image_generator_create_image`
- `image_projects_retrieve_details`
- `video_projects_retrieve_details`

`video_assets_generate_presigned_url` is the shared `/v1/files/upload-urls` endpoint. It accepts `video`, `audio`, and `image` asset items.

Do not hand-register Magic Hour endpoints in the host backend. Update
`docs/openapi.json` and restart the MCP server to pick up new endpoints.

The repo adds these custom helpers:

- `wait_for_video_project`
- `wait_for_image_project`
- `wait_for_audio_project`
- `fetch_image_download`
- `fetch_audio_download`
- `fetch_video_download`
- `upload_file_to_presigned_url`

OpenAPI policies add agent guidance by endpoint group rather than by individual endpoint.

## Required lifespan wiring

In FastAPI and Starlette, `lifespan` means app startup and shutdown logic.

Mounted ASGI sub-apps do not run their lifespan automatically. Mounting the MCP
app adds its routes but does not run startup code.

For this server, `mcp_magichour.server.lifespan` starts the MCP session manager. You must merge it into the host app lifespan or tool calls will fail at runtime.

```python
from contextlib import AsyncExitStack, asynccontextmanager

from fastapi import FastAPI

from mcp_magichour.server import app as mcp_app
from mcp_magichour.server import lifespan as mcp_lifespan


@asynccontextmanager
async def combined_lifespan(app: FastAPI):
    async with AsyncExitStack() as stack:
        # If the host app already has a lifespan, enter it first.
        # await stack.enter_async_context(existing_lifespan(app))
        await stack.enter_async_context(mcp_lifespan(app))
        yield


app = FastAPI(lifespan=combined_lifespan)
app.mount("/mcp", mcp_app)
```

## Auth model

`/mcp` reads the incoming Magic Hour API key directly from:

```text
Authorization: Bearer <magic_hour_api_key>
```

The bearer token is the user's Magic Hour API key. The MCP server does not look
up users or tenants, and the route does not inherit the host app's session or
JWT auth. Add rate limits, gateway auth, or analytics in front of `/mcp`.

Never log the raw `Authorization` header.

This setup supports developer clients. See `docs/future-oauth-support.md` for
connector authentication.

## Environment

By default, requests go to the production Magic Hour API.

Use an alternate API base for local or staging tests that should not spend credits:

```text
MAGIC_HOUR_API_BASE_URL=https://api.sideko.dev/v1/mock/magichour/magic-hour/0.66.0
```

Optional:

```text
MAGIC_HOUR_OPENAPI_PATH=docs/openapi.json
```

## Validation

Use MCP Inspector against `/mcp/` with the bearer header, then verify:

1. `ping` returns `pong`.
2. `video_assets_generate_presigned_url` returns `upload_url`, `expires_at`, and
   `file_path` for a valid key.
3. A create tool returns `{id, credits_charged}`.
4. Its `wait_for_*_project` helper reaches a terminal state.
5. A bad key returns `401` without affecting the host app.

Real create calls may spend credits. Use `exact_download_urls` exactly as
returned and never append expiration metadata.

## Checklist

- Mount the app at `/mcp` and merge its lifespan.
- Preserve `Authorization` through the proxy.
- Never log bearer tokens.
- Add host rate limits and request-size limits.
- Override `MAGIC_HOUR_API_BASE_URL` outside production if needed.
- Run the five validation checks above.
