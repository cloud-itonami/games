# games

**A thin edge for a game-platform API that has no back end here.** It accepts
eight XRPC methods — game titles, scores, leaderboards, play sessions,
achievements — validates nothing, stores nothing, and forwards each call to an
MCP router on another host. The repo holds the *edge and the contract*: a
SvelteKit Worker that translates XRPC to MCP, a BPMN file naming the eight
processes, and the identity metadata that declares the actor. **The eight
methods themselves are not implemented anywhere in this repo.**

It is `kind :app` (`README.edn`), extracted verbatim from `etzhayyim/root` at
`60-apps/etzhayyim-project-games` (`migration.edn`, source revision
`7c815d8a21`, 17 files / 71,434 bytes).

The name says the subject, not the role. Read it as *the games edge*, not as a
game or a game engine — nothing here renders, simulates, or plays anything.

## Status: not live — measured 2026-08-13

**Nothing this repo declares is reachable on the internet today,** and neither
is either of the two upstreams it would forward to. This is a measurement taken
at the tip this README landed on, not an inference from the code:

| host | declared in | resolves? |
|---|---|---|
| `etzhayyim.com` | zone for both routes | **yes** |
| `games.etzhayyim.com` | `wrangler.jsonc` route, `kotodama.jsonld`, `src/app.ts` | **NXDOMAIN** |
| `t9nt3w5u.etzhayyim.com` | `wrangler.jsonc` route | **NXDOMAIN** |
| `mcp.etzhayyim.com` | `AGENTGATEWAY_MCP_ROUTER_URL` — **the deployed edge's upstream** | **NXDOMAIN** |
| `dispatcher.etzhayyim.com` | `src/app.ts` upstream (that file is not deployed) | **NXDOMAIN** |

The parent zone resolves and a control lookup against an unrelated host
succeeded, so these are genuine absences and not a local DNS fault.

**Re-measure before trusting this table** — it is dated, and dated tables rot:

```bash
nbb docs/check-surface.cljs      # exit 1 today; exit 0 would mean this table is stale
```

The consequences are concrete:

- **The app is not deployed.** Neither route host exists.
- **The upstream is absent**, so `/xrpc/*` cannot succeed even locally. A local
  `POST /xrpc/com.etzhayyim.apps.games.listTitles` returns
  `500 {"message":"Internal Error"}` — that is the upstream fetch failing, not a
  bug in the route.

## What is actually in here

Four pieces. Only the second one is deployed.

### 1. `appview/games-mcp-component/src/app.ts` — a Worker that is **not deployed**

A handler that answers `/health` and `/_app/meta`, and proxies
`/xrpc/com.etzhayyim.apps.games.*` (GET or POST) to `DISPATCHER_URL`, attaching
an `x-internal-secret` header.

**Nothing references this file.** `wrangler.jsonc` sets
`"main": "svelte/.svelte-kit/cloudflare/_worker.js"`, which is built only from
`svelte/src/routes/`. A repo-wide search for `app.ts` outside `node_modules/`
and build output returns no hits.

Two things follow, both verified against the built artifact:

- **`/health` and `/_app/meta` do not exist in the deployable Worker.** Both
  return **404**. Any monitor pointed at them is measuring the absence of a
  route, not the health of a service.
- **The two edges do not agree on the upstream.** This file targets
  `dispatcher.etzhayyim.com` and speaks plain XRPC; the deployed one targets
  `mcp.etzhayyim.com` and speaks MCP `tools/call`. They are not two paths to the
  same place.

It is also unchecked: `npm run typecheck` cannot run (see
[Known gaps](#known-gaps)).

### 2. `appview/games-mcp-component/svelte/` — the SvelteKit BFF that **is** deployed

Two routes, and they are the whole public surface:

| route | methods | behaviour |
|---|---|---|
| `/` | GET | A static self-description page. Reports `Routes 0` — that field is hardcoded in `+page.svelte`, not derived. |
| `/xrpc/[...path]` | POST, OPTIONS | Wraps the request body as MCP `{"method":"tools/call","params":{"name":<nsid>,"arguments":<body>}}` and forwards to `AGENTGATEWAY_MCP_ROUTER_URL`, then unwraps `result.structuredContent`. |

`GET /xrpc/...` returns **405** — only `POST` and `OPTIONS` are exported. This
differs from the undeployed `src/app.ts`, which accepted GET.

The nsid is **not validated against the eight declared methods**. Any
`/xrpc/<anything>` is forwarded to the router, which is left to reject it.

### 3. `bpmn/games.bpmn` — the eight processes

Eight `bpmn:process` definitions, each a start → single `serviceTask` → end,
with a `zeebe:taskDefinition` naming the method. The set matches the eight
capabilities in `wrangler.jsonc` and `kotodama.jsonld` exactly:

```
createTitle  listTitles  recordScore  listScores
getLeaderboard  createSession  listSessions  getAchievements
```

These are process *declarations*. No worker in this repo implements a
`taskDefinition`, and no engine here executes them.

### 4. Identity — `kotodama.jsonld`, `README.edn`, `migration.edn`, `NOTICE`

Declares `did:web:games.etzhayyim.com`, an autonomous bot actor operated by
`amanomibashira`, classification `public`, protocols `xrpc` + `mcp`, and a
`subscribeRepos` trigger on the `…games.title` and `…games.score` collections.
Licensed Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1.

## Where the logic actually lives

Not here, and — as of the measurement above — not anywhere reachable:

| concern | where the config says it lives |
|---|---|
| the eight methods | behind `mcp.etzhayyim.com` (MCP router → pod-side LangServer) |
| process execution | a Zeebe-compatible engine consuming `bpmn/games.bpmn` |
| persistence | none declared — no D1, KV, R2 or DO binding in `wrangler.jsonc` |

## Known gaps

Each of these was observed, not inferred. None is fixed by this README.

1. **`npm run typecheck` cannot run.** There is no `tsconfig.json` in
   `appview/games-mcp-component/`, so `tsc --noEmit` gets no input files, prints
   its help text, and **exits 1**. It has never checked `src/app.ts`. (The
   `svelte/` subpackage has its own `tsconfig.json` and its check does work.)
2. **The test suite is a placeholder.** `test/games.test.ts` asserts
   `expect(true).toBe(true)`. It passes, and it would keep passing if every
   source file in the repo were deleted.
3. **`svelte/package-lock.json` is not committed** while the outer package's is,
   so the deployed artifact's dependency tree is unpinned.
4. **The org and the identity disagree.** The repo is `cloud-itonami/games`;
   `migration.edn` names its destination `etzhayyim/com-etzhayyim-app-games`,
   `README.edn` names it `com-etzhayyim-app-games`, and every host and DID is
   under `etzhayyim.com`. `MIGRATION-TODO.md` records the codemod as pending.
5. **`src/app.ts` is dead** — see §1. Wiring it, deleting it, or reconciling the
   two upstreams is an owner decision; this README only records that the choice
   has not been made.

## Verify any of the above

Start with [`docs/operator-quickstart.md`](docs/operator-quickstart.md). Every
command in it was executed against tip `0b068de`, and the output shown there is
the output observed.
