# memory

The long-term memory the assistant queries for home-specific facts that aren't in device state — who owns which car, where the spare key lives, trash day. The [primary prompt](../prompts/primary-prompt.md) requires the model to call the memory retrieval tool before answering any home-specific question, so this service is what backs that rule.

Two containers, wired together by [docker-compose.yml](docker-compose.yml):

| Container | Image | Role |
| --- | --- | --- |
| `memory-mcp` | `ghcr.io/doobidoo/mcp-memory-service` | The memory store itself — an MCP server over a `sqlite-vec` vector database. |
| `mcp-proxy` | `ghcr.io/tbxark/mcp-proxy` | Sits in front of the memory server and filters its tool list down to search-only before Home Assistant ever sees it. |

## Why the proxy

The memory service exposes a large set of tools — not just search, but write, delete, and a number of others unrelated to what the voice assistant needs. The proxy's job is the `toolFilter`: it lets the model see **only** `memory_search` and hides everything else.

That matters most for the write path. Writing a memory is something you want to do deliberately and verify — a fact the assistant stores incorrectly gets recalled wrong forever. So memories are written by hand through a separate LLM interface, where the text can be checked before it's saved, rather than exposed to the voice assistant, which would have to guess when a passing remark is worth persisting. Filtering to search-only takes that decision off the voice path entirely: the assistant can recall facts but can never mutate the store.

## memory-mcp

Runs the memory service in HTTP mode so the proxy can reach it over the network rather than stdio.

- **`MCP_STREAMABLE_HTTP_MODE=1`, `MCP_SSE_HOST=0.0.0.0`, `MCP_SSE_PORT=8000`** — serve over HTTP on port 8000 (published as `14002` on the host for debugging; the proxy reaches it internally on `8000`).
- **`MCP_OAUTH_ENABLED=false`** — no auth; the service is only reachable on the Docker network and the host LAN.
- **`MCP_MEMORY_STORAGE_BACKEND=sqlite_vec`** with **`MCP_MEMORY_SQLITE_PATH=/data/memory.db`** — a single-file vector DB. `./memory` is mounted to `/data`, so the database persists across restarts.
- **`MCP_MEMORY_SQLITE_PRAGMAS=journal_mode=WAL,busy_timeout=15000`** — WAL mode and a 15-second busy timeout to survive concurrent reads/writes without locking errors.

## mcp-proxy

Reads [proxy.json](proxy.json) and republishes the memory server at its own address (port `9090` internally, published as `14000` on the host) with the tool filter applied.

- **`mcpProxy.baseURL`** — the public address the proxy advertises. Set this to whatever URL Home Assistant will use to reach the proxy (a reverse-proxied hostname, or `http://<host>:14000` on the LAN).
- **`mcpServers.memory.url`** — points at the memory container on the Docker network (`http://memory-mcp:8000/mcp`).
- **`toolFilter`** — `mode: "allow"` with `list: ["memory_search"]` is the whole point of the proxy: only the search tool passes through to the model. Add tool names to the list if you decide you want the assistant to reach more of the memory service's capabilities.

## Running it

```
docker compose up -d
```

Both containers restart automatically. The memory DB lives in `./memory/memory.db`; back that file up to preserve stored facts.

## Connecting it to Home Assistant

1. Add the [Model Context Protocol](https://www.home-assistant.io/integrations/mcp/) integration (Settings → Devices & Services → Add Integration → Model Context Protocol). This is the **client** integration — it lets HA consume an external MCP server. (Don't confuse it with *MCP Server*, which points the other way, exposing Home Assistant's own tools to outside clients.)
2. Give it the proxy's endpoint — the `baseURL` from [proxy.json](proxy.json) (or `http://<host>:14000`).
3. After the integration is set up, **enable it for your conversation agent**: open the LLM/conversation integration's options and add the memory server to its list of exposed tools/services. Until you do this the model can't see `memory_search`, even though the integration is connected.

Once exposed, the `memory_search` tool shows up to the model and the primary prompt's memory rules (`mode: "hybrid"`, `limit: 2`) take effect.
