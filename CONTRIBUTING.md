Contributing
============

Thanks for your interest in improving the SWI‑Prolog MCP Server! This guide covers local setup, workflow, and what to check before opening a pull request.

Prerequisites
-------------
- Node.js ≥ 18
- SWI‑Prolog installed and on PATH (run: swipl --version)
- macOS/Linux recommended (Windows WSL works too)

Setup
-----
- Clone: git clone https://github.com/vpursuit/swipl-mcp-server.git
- Install: cd swipl-mcp-server && npm install

Quick Dev Loop
--------------
- Build then run (recommended): npm run server
- Or: npm run build && npm start
- Inspect via MCP Inspector (stdio): npx @modelcontextprotocol/inspector node build/index.js

Workflow Checklist (Before PR)
------------------------------
- Sanity
  - Confirm swipl --version works locally
  - Run unit tests: npm test (and optionally npm run test:coverage)
  - If you changed Prolog integration, sessions, or protocol, verify with MCP Inspector

- Build & Package
  - Build TypeScript: npm run build
  - Package smoke: npm run build:package, then npm pack dist/, then inspect the tarball contents

- Changes that need extra attention
  - Tool schemas/protocol
    - Update src/schemas.ts (zod + JSON schemas) and ensure src/index.ts registers JSON schemas
    - Update README/README-npm.md if inputs/outputs or protocol envelopes change
    - Add/adjust tests (e.g., test/jsonSchemas.test.ts, tool tests) to cover new shapes
  - Tool handlers
    - Keep responses text-first and add a JSON content item second (for structured clients)
    - Update tests that parse text or expect specific ordering
  - Security-impacting changes
    - Call out blacklist/sandbox/engine changes
    - Update docs/SECURITY.md as needed

- Hygiene
  - Follow TypeScript strict mode and style (2 spaces, semicolons, double quotes, relative .js imports in src/)
  - Avoid any; where unavoidable (e.g., SDK typing gap for JSON schemas), add a brief note
  - Never commit secrets, credentials, or machine-specific paths

- Commit/PR
  - Commit messages: short, imperative (e.g., "Fix engine session cleanup")
  - PRs: clear description, linked issues, repro steps; attach logs/Inspector screenshots for engine/session changes

Release
-------
- Maintainers publish the package. prepublishOnly runs build:package to create a minimal dist
- To test a candidate locally: npm pack dist/ and link dist/ with npm link

Thank you for contributing! 🙌
