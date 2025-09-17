[![Add to Cursor](https://fastmcp.me/badges/cursor_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)
[![Add to VS Code](https://fastmcp.me/badges/vscode_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)
[![Add to Claude](https://fastmcp.me/badges/claude_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)
[![Add to ChatGPT](https://fastmcp.me/badges/chatgpt_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)
[![Add to Codex](https://fastmcp.me/badges/codex_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)
[![Add to Gemini](https://fastmcp.me/badges/gemini_dark.svg)](https://fastmcp.me/MCP/Details/1066/swi-prolog)

# SWI-Prolog MCP Server

An MCP server that lets tools-enabled LLMs work directly with SWI‑Prolog. It supports loading Prolog files, adding/removing facts and rules, listing symbols, and running queries with two modes: deterministic pagination and true engine backtracking.

## Table of Contents

- [🤖 How Claude Desktop™ Uses swipl-mcp-server to Solve the 4-Queens Problem](#-how-claude-desktop-uses-swipl-mcp-server-to-solve-the-4-queens-problem)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
 - [State & Lifecycle](#state--lifecycle)
- [Tools](#tools)
- [Examples](#examples)
- [Architecture](#architecture)
- [Troubleshooting](#troubleshooting)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## 🤖 How Claude Desktop™ Uses swipl-mcp-server to Solve the 4-Queens Problem

🎯 **Watch Claude go from curious newcomer to chess puzzle master!** ✨ 

These screenshots capture an *exciting* Claude Code session where our AI friend discovers the swipl-mcp-server, gets acquainted with its tools, and then tackles the **legendary 4-queens problem** like a digital detective—*no hints, no help*, just ***pure logical reasoning*** in action! 🚀

<div style="display: flex; flex-direction: column; gap: 10px;">
  <img src="images/make-yourself-familiar.png" alt="Claude exploring swipl-mcp-server capabilities" width="600">
  <img src="images/make-yourself-familiar-2.png" alt="Claude loading Prolog predicates for 4-queens" width="600">
  <img src="images/make-yourself-familiar-3.png" alt="Claude implementing the 4-queens solution" width="600">
  <img src="images/make-yourself-familiar-4.png" alt="Claude finding all 4-queens solutions" width="600">
</div>



## Requirements

- Node.js ≥ 18.0.0
- SWI‑Prolog installed and available in PATH

## Quick Start

### Claude Desktop (basic)
```json
{
  "mcpServers": {
    "swipl": {
      "command": "npx",
      "args": ["@vpursuit/swipl-mcp-server"]
    }
  }
}
```

Optional timeouts (development):
```json
{
  "mcpServers": {
    "swipl": {
      "command": "npx",
      "args": ["@vpursuit/swipl-mcp-server"],
      "env": {
        "SWI_MCP_READY_TIMEOUT_MS": "10000",
        "SWI_MCP_QUERY_TIMEOUT_MS": "120000",
        "MCP_LOG_LEVEL": "debug",
        "DEBUG": "swipl-mcp-server"
      }
    }
  }
}
```

### MCP Inspector
```bash
npx @modelcontextprotocol/inspector --transport stdio npx @vpursuit/swipl-mcp-server
```

Configuration file location:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%/Claude/claude_desktop_config.json`

## Configuration

### Environment Variables

Configure timeouts, logging, and behavior via environment variables:

- `SWI_MCP_READY_TIMEOUT_MS`: server startup timeout (ms), default 5000
- `SWI_MCP_QUERY_TIMEOUT_MS`: query execution timeout (ms), default 30000
- `MCP_LOG_LEVEL`: `debug` | `info` | `warn` | `error` | `silent` (default `warn`)
- `DEBUG`: enable debug logs, set to `swipl-mcp-server`
- `SWI_MCP_TRACE`: optional low-level trace of child I/O and protocol
- `SWI_MCP_PROLOG_PATH`: override Prolog server script path

## State & Lifecycle

- Transport: `stdio`. The MCP client owns the connection lifecycle.
- Shutdown: the server exits on `SIGINT`/`SIGTERM` or when the client closes stdio. On stdio close, a small grace (~25ms) allows final responses to flush before exit.
- Stateful per connection: asserted facts/rules live in memory for the lifetime of the MCP connection (one Node process and one SWI‑Prolog child). When the client disconnects and the server exits, in‑memory state is reset on next start.
- Client guidance: keep a single stdio connection open for workflows that depend on shared state across multiple tool calls; avoid closing stdin immediately after a request.
- Durability (optional): if persistent Knowledge Base is desired across restarts, use `knowledge_base_dump` to save to `~/.swipl-mcp-server/` and `knowledge_base_load` (or `knowledge_base_assert_many`) to restore on startup. See docs/lifecycle.md for patterns.

## Tools

- Core: `help`, `license`, `capabilities`
- Knowledge base: `knowledge_base_load`, `knowledge_base_assert`, `knowledge_base_assert_many`, `knowledge_base_retract`, `knowledge_base_retract_many`, `knowledge_base_clear`, `knowledge_base_dump`
- Query: `query_start`, `query_startEngine`, `query_next`, `query_close`
- Symbols: `symbols_list`

## Examples

- Load and query (files must be in `~/.swipl-mcp-server/`):
  - `knowledge_base_load { filename: "~/.swipl-mcp-server/family.pl" }`
  - `query_start { query: "parent(X, mary)" }` → `query_next()` until no more solutions → `query_close()`
- Engine mode:
  - `query_startEngine { query: "member(X, [1,2,3])" }` → `query_next()` repeatedly → `query_close()`
- Database operations:
  - Single: `knowledge_base_assert { fact: "parent(john, mary)" }`
  - Multiple: `knowledge_base_assert_many { facts: ["parent(john, mary)", "parent(mary, alice)"] }`
  - Remove single: `knowledge_base_retract { fact: "parent(john, mary)" }`
  - Remove multiple: `knowledge_base_retract_many { facts: ["parent(john, mary)", "parent(mary, alice)"] }`
  - Clear all: `knowledge_base_clear {}`

See docs/examples.md for many more, including arithmetic, list ops, collections, and string/atom helpers.

## Architecture

- Single persistent SWI‑Prolog process with two query modes (standard via `call_nth/2`, engine via SWI engines)
- Term-based wire protocol: Node wraps requests as `cmd(ID, Term)`, replies as `id(ID, Reply)`; back‑compatible with bare terms
- Enhanced security model with file path restrictions, library(sandbox) validation, and dangerous predicate blocking

Details: see docs/architecture.md.

### Session State Machine

```
        startQuery                   nextSolution (eof)
 idle  ─────────────▶  query  ─────────────────────────▶  query_completed
   ▲                         │                 closeQuery │
   │       error             │                           ▼
   └─────────────────────────┘                    closing_query ──▶ idle

        startEngine                   nextEngine (eof)
 idle  ─────────────▶  engine ─────────────────────────▶  engine_completed
   ▲                         │                 closeEngine │
   │       error             │                           ▼
   └─────────────────────────┘                    closing_engine ─▶ idle
```

- Exactly one session type can be active at a time (query or engine).
- The `*_completed` states keep context so that subsequent `next` calls respond with "no more solutions" until explicitly closed.
- Transient `closing_*` states serialize shutdown before new sessions begin.
- Invalid transitions are logged when `SWI_MCP_TRACE=1`.

## Security

The server implements multiple security layers to protect your system:

### File Path Restrictions
- **Allowed Directory**: Files can only be loaded from `~/.swipl-mcp-server/`
- **Blocked Directories**: System directories (`/etc`, `/usr`, `/bin`, `/var`, etc.) are automatically blocked
- **Example**: `knowledge_base_load { filename: "/etc/passwd" }` → `Security Error: Access to system directories is blocked`

### Dangerous Predicate Detection
- **Pre-execution Blocking**: Dangerous operations are caught before execution
- **Blocked Predicates**: `shell()`, `system()`, `call()`, `assert()`, `halt()`, etc.
- **Example**: `knowledge_base_assert { fact: "malware :- shell('rm -rf /')" }` → `Security Error: Operation blocked - contains dangerous predicate 'shell'`

### Additional Protections
- Library(sandbox) validation for built-in predicates
- Timeout protection against infinite loops
- Module isolation in dedicated `knowledge_base` namespace

See [SECURITY.md](SECURITY.md) for complete security documentation.

## Troubleshooting

- "Prolog not found": ensure `swipl --version` works; SWI‑Prolog must be in PATH
- Startup timeout: increase `SWI_MCP_READY_TIMEOUT_MS`
- Query timeout: increase `SWI_MCP_QUERY_TIMEOUT_MS`
- Session conflicts: close current session before starting a different mode
- `Security Error: ...`: file access blocked or dangerous predicates detected; see Security
- Custom script path: set `SWI_MCP_PROLOG_PATH`
- Query sessions: after exhausting solutions, `query_next` returns "No more solutions available" until explicitly closed

## Development

- Install deps: `npm install`
- Build: `npm run build`
- Run dev server: `npm run server`
- Tests: `npm test` (see CONTRIBUTING.md for details)

Publishing and release workflows are documented in docs/deployment.md.

## Contributing

See CONTRIBUTING.md for local setup, workflow, and the PR checklist.

For security practices, reporting, and hardening guidance, see SECURITY.md.

Further reading:
- [docs/architecture.md](docs/architecture.md) — components, modes, and wire protocol
- [docs/lifecycle.md](docs/lifecycle.md) — server lifecycle, state, and persistence patterns
- [docs/deployment.md](docs/deployment.md) — release, packaging, and install from source
- [docs/examples.md](docs/examples.md) — copy‑paste examples

## License

BSD‑3‑Clause. See LICENSE for details.
