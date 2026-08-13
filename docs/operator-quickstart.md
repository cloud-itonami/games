# Operator quickstart

Get from a fresh clone to *"I have seen what this repo actually does"* in about
five minutes.

**Every command below was executed against tip `0b068de` on 2026-08-13**, in
this order, and the output shown is the output observed. Two steps are expected
to fail — that is stated where it happens, and a failure there is the correct
result, not a broken quickstart. One step at the end was deliberately **not**
run; it is marked.

Read [`../README.md`](../README.md) first if you have not. The short version:
this repo is an edge that forwards eight XRPC methods to an MCP router, the
methods are implemented elsewhere, and nothing it declares currently resolves.

## 0. Prerequisites

Versions used for the run recorded here. Nothing is pinned by the repo, so
newer versions are likely fine and older ones untested.

```
node    v26.3.0
npm     11.16.0
nbb     (on PATH)
```

`wrangler` is only needed for step 6, which was not run.

Set a shell variable for the app directory — every step after the first needs
it:

```bash
APP=appview/games-mcp-component
```

## 1. Is any of this live? (≈5s)

Run from the repo root:

```bash
nbb docs/check-surface.cljs
```

Observed — **exit 1, and exit 1 is the expected result today**:

```
SCANNED	15 files	DECLARED-HOSTS	5
control	registry.npmjs.org	resolves

dispatcher.etzhayyim.com  NXDOMAIN  <- appview/games-mcp-component/src/app.ts
           etzhayyim.com  resolves  <- .../kotodama.jsonld, .../wrangler.jsonc
     games.etzhayyim.com  NXDOMAIN  <- .../kotodama.jsonld, .../src/app.ts, .../wrangler.jsonc
       mcp.etzhayyim.com  NXDOMAIN  <- .../svelte/src/routes/xrpc/[...path]/+server.ts, .../wrangler.jsonc
  t9nt3w5u.etzhayyim.com  NXDOMAIN  <- .../wrangler.jsonc

4 of 5 declared hosts do not exist.
```

The script has three exit codes, and they were each demonstrated before this
document was written:

| exit | meaning |
|---|---|
| `0` | every declared host resolves — the README's status table is stale |
| `1` | at least one does not resolve — **today's state** |
| `3` | **could not answer**: the control host failed (no working DNS here), or zero hosts were extracted (wrong directory) |

Exit `3` exists so that "I could not measure" is never reachable from the same
exit code as "I measured, all fine".

## 2. Install and test the outer package (≈10s)

```bash
cd $APP
npm install --no-audit --no-fund
npm test
```

Observed — `added 65 packages`, then:

```
 RUN  v4.1.8

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

**Do not read this as coverage.** The single test is
`expect(true).toBe(true)`. It would still pass with every source file in the
repo deleted. See known gap 2 in the README.

## 3. Watch the typecheck fail (≈3s) — expected

```bash
npm run typecheck        # still in $APP
```

Observed: **exit 1**, and instead of type errors it prints the `tsc` help text
(`--removeComments`, `--strict`, `--types`, …).

That is the whole finding: there is no `tsconfig.json` in `$APP`, so
`tsc --noEmit` receives no input files and falls back to printing usage.
`src/app.ts` has never been typechecked by this script.

Confirm the cause directly:

```bash
ls tsconfig.json          # No such file or directory
ls svelte/tsconfig.json   # exists — the subpackage has one, and its check works
```

## 4. Build the artifact that actually deploys (≈20s)

`wrangler.jsonc` sets `"main": "svelte/.svelte-kit/cloudflare/_worker.js"`, so
this is the step that produces the deployable Worker. `src/app.ts` is not part
of it.

```bash
cd svelte
npm install --no-audit --no-fund
npm run check
```

Observed — `added 92 packages`, then `svelte-check`:

```
COMPLETED 148 FILES 0 ERRORS 0 WARNINGS 0 FILES_WITH_PROBLEMS
```

Then build. **Inside the `com-junkawasaki` superproject, route the build
through the shared resource governor** — concurrent agent sessions on this
machine must not run two heavy builds at once:

```bash
node /path/to/com-junkawasaki/scripts/resource-guard.mjs run build -- npm run build
```

Standalone clones outside that superproject have no such script; use
`npm run build` directly.

Observed — exit 0, `✓ built in 6.46s`, `Using @sveltejs/adapter-cloudflare`.
Confirm the artifact wrangler expects now exists:

```bash
ls -la .svelte-kit/cloudflare/_worker.js     # 4335 bytes on the recorded run
```

## 5. Probe the real surface locally (≈30s)

Serve the build you just made:

```bash
npm run preview -- --port 4173        # still in $APP/svelte
```

In another shell, the six requests that define this repo's behaviour. Every
status below was observed:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:4173/
# 200   — the static self-description page, <title>games-mcp-component</title>

curl -s -o /dev/null -w '%{http_code}\n' http://localhost:4173/health
# 404   — NOT a bug. /health only exists in the undeployed src/app.ts.

curl -s -o /dev/null -w '%{http_code}\n' http://localhost:4173/_app/meta
# 404   — same reason.

curl -s -o /dev/null -w '%{http_code}\n' \
  http://localhost:4173/xrpc/com.etzhayyim.apps.games.listTitles
# 405   — the SvelteKit route exports POST and OPTIONS only.

curl -s -o /dev/null -D - -X OPTIONS \
  http://localhost:4173/xrpc/com.etzhayyim.apps.games.listTitles
# 204 + access-control-allow-methods: POST,OPTIONS
#     — the one call that succeeds end to end, because it never touches upstream.

curl -s -X POST -H 'content-type: application/json' -d '{}' \
  http://localhost:4173/xrpc/com.etzhayyim.apps.games.listTitles
# 500 {"message":"Internal Error"}
#     — expected. This is the fetch to mcp.etzhayyim.com failing (step 1), not
#       a fault in the route. It cannot succeed until that host exists.
```

Those first four statuses are the check worth keeping: **the endpoints named in
`src/app.ts` are not on the deployed surface.** If you have a monitor pointed at
`/health`, it is watching a route that does not exist.

Stop the preview server when done.

## 6. Deploy — NOT RUN

```bash
npx wrangler deploy        # from $APP
```

This was **not** executed, and you should not execute it to "check the
quickstart". Deploying would publish `games.etzhayyim.com` and
`t9nt3w5u.etzhayyim.com`, and the result would be a surface whose only working
call is the CORS preflight — every real method would 500 against a router that
does not resolve. Fix the upstream first.

The workspace also requires any deploy to run from a checkout that contains
`origin/main`, since deploys have no fast-forward check and the last writer
wins.

## What you now know

- The repo is an edge, not a service. The eight methods live behind
  `mcp.etzhayyim.com` (step 1) and are implemented nowhere in this tree.
- `src/app.ts` is dead code — the deployed Worker is built from `svelte/`
  (steps 4, 5), and the two disagree about both upstream and protocol.
- Neither the typecheck (step 3) nor the test suite (step 2) would notice if
  `src/app.ts` broke.
- Nothing is live, and nothing can be until `mcp.etzhayyim.com` exists (step 1).
