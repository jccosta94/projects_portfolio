# MCP integration

> **Status:** outline draft.

## The MCP servers in this system
- Firebase MCP — project introspection and Firestore CRUD
- Gmail MCP — outbound email for dealer hand-off
- Airtable MCP — CRM read/write
- Gemini Image MCP — image generation
- Veo MCP — video generation
- Computer-use MCP — desktop control (Cowork interactive workflows)
- Excalidraw MCP — diagram authoring
- Custom website MCP — landing-page edits

## Why MCP instead of direct API clients
- One auth surface per server, not per call site.
- Tools are discoverable and the LLM picks the right one without rewiring.
- Servers are reusable across the desktop agent (Cowork) and the scheduled agent (OpenClaw).

## Integration patterns
- Stdio for desktop-local servers (e.g., Gemini stdio for Cowork).
- HTTP/SSE for shared servers consumed by both desktop and Docker (e.g., Gemini SSE on :8901 for OpenClaw).
- Same server, two transports — documented in the change log.

## The launchd-supervised pattern (macOS)
- Long-running MCP servers (Gemini Image MCP) are supervised by launchd, not run manually.
- A manual start returns errno 48 (port in use) because launchd has already respawned the server.
- Pattern: configure once, never start by hand.
