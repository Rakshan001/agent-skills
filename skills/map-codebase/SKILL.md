---
name: map-codebase
description: Build the graphify knowledge graph and CODEBASE.md so agents navigate a compact map instead of reading the whole repo.
---

# Map this codebase

Produce Baton's two navigation artifacts so every later agent reads a map
instead of the whole repo.

## Steps

- `baton kb init` — register this repo with the knowledge base if it isn't
  already.
- `baton kb rebuild` — build (or incrementally update) the graphify knowledge
  graph and regenerate `CODEBASE.md`, the compact repo map.
- Open `CODEBASE.md` and sanity-check it: the top-level structure, the entry
  points, and the key modules should be recognisable. If a major area is
  missing, the graph may need a full rebuild: `baton kb rebuild --full`.
- Wire the graph into your agent over MCP (the dashboard's **Connect MCP**
  button, or `baton mcp`) so you can query symbols directly.

The map costs ~hundreds of tokens to read; the raw repo costs ~hundreds of
thousands. Always navigate from the map.
