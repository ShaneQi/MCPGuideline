# MCPGuideline

Conventions for building, securing, testing and deploying remote MCP servers,
taken from StocksMCP, MoinMoinMCP, LinkCliMCP and OFCloudMCP.

**→ [GUIDELINE.md](GUIDELINE.md)**

In short:

- Cloudflare Worker by default; a Node 22 container (VPS or Cloud Run) only
  when the server needs a CLI, a persistent disk or heavy CPU.
- Stateless Streamable HTTP at `POST /mcp`, with useful `instructions`.
- Cloudflare Access in front, and the shared `access.js` verifying the JWT at
  the origin; fail closed.
- Writes off unless `ALLOW_WRITES=true`; secrets never in the repo.
- `node --test` with faked upstreams; every push to `main` deploys.
