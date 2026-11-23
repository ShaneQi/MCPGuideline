# MCP server development and deployment guideline

How to build, secure, test and ship a remote MCP server, drawn from the four
servers that already follow it:

| Repo | What it does | Runtime | Auth | Writes |
|---|---|---|---|---|
| [StocksMCP](https://github.com/ShaneQi/StocksMCP) | Stock/ETF/crypto quotes from Yahoo Finance | Cloudflare Worker (TypeScript, MCP SDK + `agents`) | Cloudflare Access | none |
| [MoinMoinMCP](https://github.com/ShaneQi/MoinMoinMCP) | Read/write a MoinMoin 1.9 wiki over XML-RPC | Cloudflare Worker (JS, no deps) | Cloudflare Access | `ALLOW_WRITES` |
| [LinkCliMCP](https://github.com/ShaneQi/LinkCliMCP) | Read-only bank/card data via Stripe's link-cli | Container (Node 22) on VPS or Cloud Run | Cloudflare Access | none, by construction |
| [OFCloudMCP](https://github.com/ShaneQi/OFCloudMCP) | Read/write an OmniFocus database over WebDAV | Container (Node 22) on VPS or Cloud Run | Cloudflare Access | `ALLOW_WRITES` + `dry_run` |

"MUST" marks things every server here already does and new ones should too;
"should" marks the default that can be broken with a reason written in the
README.

---

## 1. Pick the runtime

Use a **Cloudflare Worker** by default. Use a **container** only when the
server needs something a Worker can't give it:

| Need | Choose |
|---|---|
| Stateless calls to an HTTP API | Worker (StocksMCP, MoinMoinMCP) |
| A child process / CLI binary (e.g. `link-cli`) | Container (LinkCliMCP) |
| State that must survive restarts on a disk (rotating refresh tokens, a large parsed cache) | Container with a volume at `/data` (LinkCliMCP, OFCloudMCP) |
| Long CPU-heavy work (PBKDF2 with ~1.3 M rounds, replaying thousands of files) | Container (OFCloudMCP) |

MoinMoinMCP started as a Python container and moved to a Worker once it was
clear it only needed HTTP calls: prefer the smaller runtime when you can.

Either way, write the request handling as a **fetch-style handler**
(`export default { fetch(request, env, ctx) }`) in `src/index.js`. A Worker
exports it directly; a container wraps it in a ~100-line `src/server.js`
using `node:http`. That keeps the code portable between the two and lets
tests call `worker.fetch(new Request(...), env)` without a network.

## 2. Repository layout

```
src/
  index.js      fetch handler: routes, /health, Access check, config check
  server.js     (containers only) node:http wrapper, startup checks, shutdown
  access.js     Cloudflare Access JWT verification — identical in every repo
  mcp.js        stateless MCP JSON-RPC: initialize, ping, tools/list, tools/call
  tools.js      tool definitions, argument validation, result formatting
  errors.js     UserError (message safe to show Claude)
  <backend>.js  the upstream client (moin.js, linkcli.js, dav.js, …)
test/           node --test; fakes for the upstream; access-helper.js
.github/workflows/  deploy.yml (Worker) or container.yml (container)
wrangler.toml   (Worker)  |  Dockerfile, .dockerignore, compose.example.yml (container)
README.md
```

- Language: plain ES modules (`"type": "module"`), Node ≥ 22. TypeScript is
  fine (StocksMCP) but not required.
- **Keep dependencies minimal.** MoinMoinMCP has none; OFCloudMCP has two
  (`fast-xml-parser`, `fflate`). Use WebCrypto (`crypto.subtle`) and `fetch`
  instead of libraries. Pin a CLI you shell out to to an exact version
  (`"@stripe/link-cli": "1.3.0"`).
- Commit `package-lock.json`; CI uses `npm ci`.
- `.gitignore`: `node_modules/`, `.wrangler/`, `.dev.vars`, `.env`,
  `compose.yml`, and any local dump/data folder (it holds real user data).

## 3. MCP protocol layer

- **Transport:** Streamable HTTP at `POST /mcp`, **stateless**, JSON responses
  only (no SSE, no sessions, no Durable Objects). Notifications get `202`.
  Batches are answered as arrays. Non-POST on `/mcp` gets `405` with
  `Allow: POST`.
- Either use the official SDK with a stateless handler (StocksMCP:
  `createMcpHandler` from `agents/mcp/server`), or the ~70-line hand-written
  `mcp.js` shared by the other three. The hand-written one supports
  `initialize`, `ping`, `tools/list`, `tools/call`, answers unknown methods
  with `-32601`, bad JSON with `-32700`, and negotiates
  `protocolVersion` from `["2025-06-18", "2025-03-26", "2024-11-05"]`.
- **`initialize` MUST return `instructions`**: a short paragraph that tells
  Claude what the data means and how to use the tools safely — units and sign
  conventions (LinkCliMCP: "Amounts are integer cents… on a card negative =
  charge"), "read before editing; put_page replaces the entire page"
  (MoinMoinMCP), which tools change real data (OFCloudMCP), and how to recover
  (LinkCliMCP: "call sign_in, give the user the verification_url…"). Put
  lessons learned from real use here; mirror them in the README.
- `serverInfo` has a `name`, a human `title` and a `version`. Keep the title
  truthful (don't say "read-only" on a server with write tools).

## 4. Tool design

- **Names:** `snake_case` verbs: `list_*`, `get_*`, `search_*`, plus clear
  write verbs (`put_page`, `add_task`, `complete_task`). Give every tool a
  `title` and a description that states inputs' formats and limits.
- **Annotations MUST be accurate:** read tools
  `readOnlyHint: true`; write tools `readOnlyHint: false` and
  `destructiveHint` true for anything that deletes or overwrites;
  `openWorldHint` true when the tool talks to an outside system.
- **Validate every argument** yourself (type, range, regex, length) and
  return a precise message (`"from must be a date like 2026-11-01."`).
  Arguments that reach a command line must not start with `-` or contain
  newlines/NUL (LinkCliMCP `text()`).
- **Errors:** throw `UserError` (or a `ToolError` with a stable code prefix
  such as `INVALID_ARGUMENTS:` / `WRITES_DISABLED:`) for anything Claude can
  act on; it comes back as `{ isError: true, content: [text] }`. Anything
  else becomes `Unexpected error: …` and is logged with its stack. Protocol
  errors (unknown tool) are JSON-RPC errors, not tool results.
- **Results:** a short human-readable text line first, then structured JSON
  if useful (StocksMCP returns both). Include **ids** Claude can pass back
  (`[id:…]` in OFCloudMCP) and **URLs** the user can click (`url` fields in
  MoinMoinMCP).
- **Do the paging for Claude** when the upstream paginates
  (`list_transactions` follows pages).
- **Be gentle on upstreams:** cache briefly (StocksMCP edge-caches 15 s),
  batch calls (`system.multicall`), refresh in the background
  (`REFRESH_MINUTES`) rather than on every request, set timeouts on every
  outbound call.
- Log one line per call: `tool <name>: ok|error|failed (<ms> ms)`.

## 5. Authentication: Cloudflare Access

Every server MUST sit behind **Cloudflare Access** and verify the Access JWT
**itself**, on every `/mcp` request — all four do, including StocksMCP, whose
data is public. Access decides
who gets in (identity provider, service token, …); the origin check makes
sure nobody gets around Access via the `run.app` URL, the VPS IP or a
`workers.dev` URL.

- **Use the shared `src/access.js` unchanged.** It is byte-identical in
  MoinMoinMCP, LinkCliMCP and OFCloudMCP (check with `md5sum src/access.js`).
  StocksMCP has a line-for-line TypeScript port, `src/access.ts`, with the
  same behaviour plus `AccessEnv`/`AccessConfig` types. Fix bugs in all
  copies, the port included, in the same change. It:
  - reads `CF_ACCESS_TEAM_DOMAIN` (`team`, `team.cloudflareaccess.com` or a
    full URL) and `CF_ACCESS_AUD` (comma-separated for several apps);
  - takes the token from `Cf-Access-Jwt-Assertion`, else `Authorization: Bearer`;
  - accepts **RS256 only**, matches `kid` against
    `https://<team>/cdn-cgi/access/certs`, checks `iss`, `aud`, `exp`/`nbf`
    with 60 s leeway;
  - caches keys for an hour, re-fetches on an unknown `kid` at most once a
    minute, keeps using cached keys if a refresh fails, shares one in-flight
    fetch, 10 s fetch timeout.
- **Fail closed.** Missing Access settings → `/mcp` answers `500` (Worker),
  and a container **exits at startup**. Never fall back to "no auth".
- No or bad token → `401` with `WWW-Authenticate: Bearer` and body
  `Unauthorized`. The reason goes only to the log, as `access denied: …`.
- `/health` (and `/healthz`) and `/` stay open and reveal nothing. Use
  `/health` in probes: some hosts (Cloud Run) reserve paths ending in `z`.
- **No static bearer tokens.** LinkCliMCP replaced its `ACCESS_TOKEN` with
  Access; don't reintroduce shared secrets in URLs or headers.
- **SDK-based servers check first, then hand off.** StocksMCP runs the
  Access check in its own `fetch` before calling the SDK's `createMcpHandler`,
  so the SDK never sees an unauthenticated request.
- **Client sign-in:** with an Access application on the hostname (StocksMCP
  uses a "Cloudflare account members" policy), Access answers unauthenticated
  requests with `401` and points MCP clients at its OAuth sign-in, so
  claude.ai and Claude Code log in through Access when they connect. Clients
  that can only send fixed headers use an Access service token
  (`CF-Access-Client-Id` / `CF-Access-Client-Secret`) with a Service Auth
  policy.
- For local `wrangler dev`, put `CF_ACCESS_TEAM_DOMAIN` and `CF_ACCESS_AUD` in
  `.dev.vars` (git-ignored).
- Even a server with only public data stays behind Access: the URL, the
  upstream's rate limit and the free-plan request quota are worth protecting.
  An unauthenticated server needs a reason written in its README.

## 6. Safety rules for data and secrets

- **Writes are off by default.** Write tools work only when
  `ALLOW_WRITES=true`; otherwise they return `WRITES_DISABLED` (MoinMoinMCP)
  or a preview (OFCloudMCP), and their description says writing is disabled.
  Offer `dry_run: true` for anything non-trivial.
- **Read-only by construction** where the upstream can do dangerous things:
  allowlist the exact commands the server may run (LinkCliMCP `ALLOWED`)
  rather than blocklisting.
- **Verify before writing.** OFCloudMCP builds the change, reads it back with
  its own reader, and only uploads if it matches; uploads under a temp name
  and `MOVE`s into place; refuses to write over unmerged state. Do the same
  kind of check for any format you write that isn't yours.
- **Mark what you wrote** so the user can tell it apart (OFCloudMCP adds an
  `OFCloudMCP` signature line to notes of items it creates).
- **Redact** tokens from every result and error (`access_token`,
  `refresh_token`, token-shaped strings). Strip secrets from the env handed to
  child processes.
- **One writer per state.** Rotating, single-use refresh tokens break if two
  processes refresh them: run one container per volume, `--max-instances=1`
  on Cloud Run, and serialize calls through one queue. On shutdown
  (`SIGTERM`), stop accepting requests and wait for in-flight work so a
  refreshed token is saved.
- Secrets never live in the repo, `wrangler.toml`, the Dockerfile or workflow
  files. The `/data` volume holds live credentials or decrypted user data:
  `chmod 700`, owned by the `node` user, out of shared backups — say so in the
  README.

## 7. Configuration

- All configuration is **environment variables**, documented in one README
  table (name, default, meaning).
- Validate config up front and say exactly what is wrong
  (`"Set both MOIN_USER and MOIN_PASSWORD, or neither"`;
  `"ALLOW_WRITES must be true or false"`). Config errors answer `500` with the
  message, never a partial or open service.
- **Workers:** variables and secrets are set in the Cloudflare dashboard, not
  in `wrangler.toml`. `wrangler.toml` keeps them across deploys:

  ```toml
  workers_dev = false
  preview_urls = false

  [unsafe.metadata]
  keep_bindings = ["plain_text", "json", "secret_text", "secret_key"]
  ```

  No `routes` in `wrangler.toml`: the custom domain is added in the dashboard
  (Workers → *name* → Settings → Domains & Routes) and deploys leave it alone.
  Local dev secrets go in `.dev.vars` (git-ignored).
- **Containers:** an `.env` next to the compose file, `chmod 600`. On Cloud
  Run, non-secret settings via `--set-env-vars`, secrets via
  `--set-secrets`.

## 8. Testing

- `node --test test/*.test.js` (`npm test`), no test framework dependency.
- Test through the real entry point: build a `Request`, call
  `worker.fetch(req, env)`, assert on status and JSON-RPC body.
- **Fake the upstream**, never call the real one in CI: a fake CLI script
  (`test/fake-link-cli.mjs`, selected via `LINK_CLI_BIN`), an in-memory WebDAV
  server, generated fixtures (OFCloudMCP's encrypted `.ofocus`, made and
  cross-checked by independent Python scripts).
- **Test auth**: `test/access-helper.js` generates an RSA key, installs it with
  `setAccessKeys()`, and signs Access-style JWTs. Cover missing token, wrong
  signature, wrong `aud`/`iss`, expired, unknown `kid`, and missing config.
- Test the safety rules explicitly: writes refused when disabled, disallowed
  commands unreachable, tokens redacted, env tokens not passed to children.
- Before trusting a reverse-engineered format, verify against real data and
  record what was verified and what is still assumed (OFCloudMCP's
  maintainer notes).

## 9. Deployment

Every push to `main` deploys. Markdown-only pushes are skipped
(`paths-ignore: ["**.md"]`), and `workflow_dispatch` allows a manual run.

### 9a. Cloudflare Worker (`.github/workflows/deploy.yml`)

```yaml
on:
  push: { branches: [main], paths-ignore: ["**.md"] }
  workflow_dispatch:
concurrency: { group: deploy, cancel-in-progress: false }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4        # only if there are dependencies
        with: { node-version: 22, cache: npm }
      - run: npm ci                        # only if there are dependencies
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          wranglerVersion: "4"
          command: deploy
```

- Repository secrets: `CLOUDFLARE_API_TOKEN` ("Edit Cloudflare Workers"
  template) and `CLOUDFLARE_ACCOUNT_ID`.
- A deploy replaces only the code: bindings stay (`keep_bindings`), the
  custom domain stays, `workers.dev` and preview URLs stay off so the custom
  domain behind Access is the only way in.
- Anything not in `wrangler.toml` that the dashboard controls per deploy is
  reset by it: StocksMCP has no `[observability]` block, so each deploy turns
  Workers observability off. Put it in `wrangler.toml` if you want it kept.
- Add a test/typecheck step before deploy once the repo has tests.
- Manual deploy: `npx wrangler login && npx wrangler deploy`.

### 9b. Container (`.github/workflows/container.yml`)

On `main`, `v*` tags and pull requests:

1. **test** job: `npm ci && npm test`.
2. **image** job (`needs: test`): buildx for `linux/amd64,linux/arm64`; push
   to GHCR (`GITHUB_TOKEN`, `packages: write`) and to Google Artifact Registry
   (`GCP_SA_KEY`). Tags from `docker/metadata-action`: `latest` on the default
   branch, `sha-<short>`, the semver on `v*` tags. PRs build but don't push.
   `provenance: false`, `sbom: false`, GHA cache.
3. On `main`: `gcloud run services update <service> --image <…>@<digest>` —
   **only the image** changes; env, secrets and volumes are kept. The service
   is created once by hand (command in the README).

Registry destination and Cloud Run service go in the workflow's `env:` block.
The service account needs Artifact Registry Writer, Cloud Run Developer on the
service, and Service Account User on the runtime account. If you only use a
VPS, delete the GAR and Cloud Run steps.

**Dockerfile** (same shape in both container repos):

```dockerfile
FROM node:22-alpine
ENV NODE_ENV=production PORT=8787
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force   # add --ignore-scripts if a dep's postinstall misbehaves
COPY src ./src
RUN mkdir -p /data && chown node:node /data && chmod 700 /data
USER node
VOLUME /data
EXPOSE 8787
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD wget -qO- http://127.0.0.1:8787/health || exit 1
CMD ["node", "src/server.js"]
```

`.dockerignore` excludes `node_modules`, `.env`, `.dev.vars`, local data,
`test`, `scripts`, `*.md`, `.git`, `.github`, `compose*.yml`.

**VPS:** `compose.example.yml` (copied to the git-ignored `compose.yml`)
binds `127.0.0.1:8787` only, uses `restart: unless-stopped`, `env_file: .env`
and a named volume at `/data`. Publish it through Cloudflare (Tunnel, or a
proxied hostname in front of a reverse proxy) with an Access application.
Update with `docker compose pull && docker compose up -d`. GHCR packages start
private: make the package public or `docker login ghcr.io` with a
`read:packages` token.

**Cloud Run:** `--port 8787`, `--allow-unauthenticated` (the container
enforces Access; Cloud Run IAM would block Claude), a Cloudflare-proxied
custom domain, and for stateful servers `--max-instances=1` plus a Cloud
Storage volume at `/data` (runtime account needs Storage Object User).
Background timers don't run while scaled to zero; design for refresh-on-call.

### 9c. Verify a deploy

```sh
curl https://<host>/health                     # → ok
curl -X POST https://<host>/mcp                # → stopped by Access
curl -X POST http://127.0.0.1:8787/mcp         # (origin) → 401
curl -X POST https://<host>/mcp \
  -H "CF-Access-Client-Id: <id>" -H "CF-Access-Client-Secret: <secret>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Locally, `npx @modelcontextprotocol/inspector` against `http://localhost:8787/mcp`.

### 9d. Connect a client

- claude.ai / Desktop / mobile / Cowork: Settings → Connectors → Add custom
  connector → `https://<host>/mcp`, authenticating as the Access policy
  allows.
- Claude Code: `claude mcp add --transport http <name> https://<host>/mcp`.

## 10. README

Each repo's README is the operator's manual and the next maintainer's (or
agent's) briefing. Sections, in order:

1. One paragraph: what it gives Claude, where it runs, how it is protected.
   Tool list, split into read and write tools.
2. **How it works** — the upstream, the data flow, anything stateful.
3. **Running it** — Worker: dashboard config table + deploy. Container: VPS
   steps (image, `.env`, start, Access, add to Claude, updating) and Cloud Run.
4. **Authentication** — the Access checks and where to find
   `CF_ACCESS_TEAM_DOMAIN` / `CF_ACCESS_AUD`.
5. **Configuration reference** table and endpoints.
6. **Data notes from real use** (also in the MCP `instructions`).
7. **Troubleshooting** — each real error message and its fix.
8. **Continuous integration.**
9. **Maintainer notes** — code map table, upstream behaviour the code relies
   on (and what to recheck when bumping it), verified vs assumed formats,
   gotchas (e.g. bump the cache key when the model shape changes).
10. **Development** — `npm ci`, `npm test`, `npm start`/`npm run dev`.

## 11. Workflow conventions

- Work on a branch, merge to `main` by pull request; `main` deploys.
- Commit messages: imperative, describe the behaviour change
  ("Keep workers.dev and preview URLs off on every deploy").
- When a convention changes in one server (auth, deploy settings), apply it to
  all of them in matching PRs ("Match LinkCliMCP and OFCloudMCP's Cloudflare
  Access handling") and update this guideline.

---

## New-server checklist

- [ ] Runtime chosen per §1; fetch-style `src/index.js`
- [ ] Stateless `POST /mcp`, `initialize` with useful `instructions`
- [ ] Every tool: title, description, accurate annotations, validated args
- [ ] `UserError` vs unexpected errors; one log line per call
- [ ] Shared `access.js` copied unchanged; fails closed; `/health` open
- [ ] Writes behind `ALLOW_WRITES` (default off), `dry_run` where it matters
- [ ] Secrets only in dashboard / `.env` / Secret Manager; tokens redacted
- [ ] `node --test` suite with faked upstream and Access JWT tests
- [ ] Worker: `workers_dev = false`, `preview_urls = false`, `keep_bindings`
- [ ] Container: multi-arch image, non-root, `/data` volume, healthcheck, `127.0.0.1` bind
- [ ] CI deploys on push to `main`, skips Markdown-only pushes
- [ ] README sections per §10
- [ ] Deploy verified per §9c

## Known gaps in the current repos

Places where a repo doesn't yet match this guideline:

- **StocksMCP** has no tests (the Access check in particular), and its
  deploy workflow doesn't run `npm run typecheck`. Its `access.ts` is a port,
  so it can drift from the shared `access.js`.
- **MoinMoinMCP** has no `package.json` or tests (removed in the Python →
  Worker move); its deploy has no test gate.
- **OFCloudMCP**'s `serverInfo.title` and Dockerfile comment still say
  "read-only" although it has write tools; `callTool` doesn't log per-call
  timing like LinkCliMCP.
- Error class naming differs (`UserError` in the container repos, `ToolError`
  with code prefixes in MoinMoinMCP); pick one when next touching them.
- `compatibility_date` differs between the two Workers (`2025-11-01` vs
  `2026-06-11`); bump deliberately and note it.
