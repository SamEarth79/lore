# LORE-STORY-001-006: Public read-only MCP server

## Description

As a judge with an MCP client, I want to connect to Lore's `/mcp` endpoint and call the same tools the agent uses, so that I can verify the graph does the reasoning.

## Acceptance criteria

1. FastMCP is built from the tool registry and mounted at `/mcp` (streamable HTTP, `stateless_http=True`).
2. Only the 12 read tools are listed; no write function is exposed.
3. All MCP calls run against `lore_base` only; there is no way to target a sandbox or another graph.
4. If CORS is enabled for browser inspectors, it is scoped to `/mcp` only with `allow_credentials=False`.
5. A test using the MCP Python SDK client lists tools and calls `resolve_entity` and `get_owner` successfully, returning the same envelope as in-process calls.
6. README snippet shows how to connect Claude Desktop / MCP Inspector to the deployed `/mcp` URL.

## Wireframe

none

## Requirements implemented

- gather.md: Scope: tool catalogue exposed via an MCP server

## Depends on

- LORE-STORY-001-005

## Agents likely needed

- [ ] frontend
- [x] backend
- [ ] infrastructure

## Status

- [ ] Implemented
- [ ] Tested
- [ ] Committed
