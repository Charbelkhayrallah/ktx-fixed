# ktx-fixed

An unofficial patched build of [`@kaelio/ktx`](https://github.com/Kaelio/ktx)
**v0.16.0**, published as **`@charbelkh/ktx` 0.16.5**. Six fixes, all in this
repo as a single patch you can apply to a pristine upstream checkout.

Not affiliated with or endorsed by Kaelio. Apache-2.0, same as upstream.

| # | Symptom | Fix |
|---|---|---|
| 1 | `ktx ingest` regenerates every AI description on every run | Drop wall-clock and row-count fields from the resume hashes |
| 2 | Adding one table discards all the other descriptions | Track a resume hash per table instead of per batch |
| 3 | Every ktx MCP tool fails in Claude Code before it runs | Make the MCP SDK advertise JSON Schema 2020-12 |
| 4 | Ctrl-C cannot stop an ingest | Stop leaking the terminal into raw mode |
| 5 | Postgres foreign keys all missing, silently, unless you own the tables | Read them from `pg_catalog` instead of `information_schema` |
| 6 | A run that ends with descriptions missing caches the gap as "complete", so every later `ktx ingest` reports success in seconds and never retries | Never cache an incomplete descriptions stage, and ignore one already cached |

### What is in this repo

| File | |
|---|---|
| `ktx-fixed.tgz` | the built package — what you install |
| `ktx-fixes.diff` | the complete source patch against upstream `v0.16.0` |
| `README.md` | this file |
| `LICENSE` | Apache-2.0 |

## Install

```bash
npm uninstall -g @kaelio/ktx
npm i -g https://raw.githubusercontent.com/Charbelkhayrallah/ktx-fixed/main/ktx-fixed.tgz
```

Check it worked — should print `@charbelkh/ktx 0.16.5`:

```bash
ktx --version
```

**Installing from a mirror of this repo?** A raw-file URL only works where the host serves
the file anonymously. On a private mirror it redirects to a sign-in page, npm downloads
that HTML instead of the tarball, and the install fails on a file that is not a gzip
archive. Clone and install from disk instead — the tarball is committed, so this needs no
token and no change of visibility:

```bash
npm uninstall -g @kaelio/ktx
git clone <this repo> ktx-fixed
npm i -g ./ktx-fixed/ktx-fixed.tgz
```

> **Uninstall the official one first.** Both provide the `ktx` command and will
> clash. If you still get `File exists: ...\ktx.cmd`, delete `ktx`, `ktx.cmd`
> and `ktx.ps1` from `%APPDATA%\npm`, then install again.

New to ktx? Run `ktx setup`. Docs: https://docs.kaelio.com/ktx

## Use

Exactly like normal ktx — same command, same flags, all
[official docs](https://docs.kaelio.com/ktx) apply.

```bash
ktx ingest <connection-id>
```

The difference is what happens on a **re-run**:

| | Official 0.16.0 | This build |
|---|---|---|
| Re-running `ktx ingest` | Regenerates every description | Reuses unchanged tables |
| Ctrl-C mid-ingest, then re-run | Starts over | Keeps finished tables |
| Adding one table | Re-runs all of them | Runs only the new one |

**Descriptions an LLM rate limit left empty are always retried.** Change 2
recovers the tables already described, and change 6 makes sure a run that ended
with gaps is never cached as complete — and that a gap cached by an earlier build
is ignored rather than replayed. So a plain re-run fills them:

```bash
ktx ingest <connection-id>
```

Up to 0.16.4 this was not reliable: a run that *finished* with tables still
undescribed stored that gap as a completed stage, and every later run handed it
back in seconds with a clean tick. `--stages descriptions` was the workaround. It
still forces a recompute if you want one, but should no longer be necessary.

That keeps the good descriptions and regenerates only the empty ones.

## What was changed

Four changes, all in [`ktx-fixes.diff`](ktx-fixes.diff) — 190 added lines across
six files, nothing else touched. Paths below are relative to the upstream repo
root.

### 1. Volatile fields removed from the resume hashes

`packages/cli/src/context/scan/enrichment-state.ts`

The resume hashes were computed over the whole schema snapshot, which carries
`extractedAt` — a wall-clock timestamp set fresh on every scan. An unchanged
schema therefore produced a new hash on every run, missed its cache, and forced
a full regeneration.

A new `stableSnapshotHashInput()` strips `extractedAt` before hashing, and is
applied to all three stage hashes (`computeKtxDescriptionsStageHash`,
`computeKtxEmbeddingsStageHash`, `computeKtxRelationshipsStageHash`). This
mirrors `stableLiveDatabaseHashContent` in `local-stage-ingest.ts`, which
already stripped it for the ingest work-unit hash.

The new per-table hash (change 2) had the same problem with `estimatedRows`, a
live row count that drifts on any active database, so
`computeKtxTableDescriptionHash()` excludes that too. A hash now changes only
when something a description actually depends on changes.

### 2. Per-table resume

`packages/cli/src/context/scan/local-enrichment-artifacts.ts`,
`packages/cli/src/context/scan/local-enrichment.ts`

The resume record was all-or-nothing on a single whole-batch hash, so touching
one table discarded every description in the batch. `DescriptionsProgressRecord`
gains a `tableHashes` map keyed by table ref, and each table is now recovered
independently.

The load-bearing part is that `load()` **drops its `inputHash` gate**. Upstream
returned the prior record only when `record.inputHash === inputHash`, so any
change to the stage hash threw away every description. It now always returns the
record, and the per-table hash alone decides what is recovered — `inputHash` is
still written by `flush()` but is never compared. That is what makes resume
survive a changed stage hash, which matters because the stage hash is *not*
stable: `stableSnapshotHashInput()` strips only `extractedAt`, so every table's
`estimatedRows` is still in it and any row-count drift re-keys the stage.

Together, changes 1 and 2 restore the behaviour ktx's own
[CLI reference](https://docs.kaelio.com/ktx/docs/cli-reference/ktx-ingest)
already documents. Reported upstream as
[#347](https://github.com/Kaelio/ktx/issues/347) and
[#348](https://github.com/Kaelio/ktx/issues/348).

### 3. MCP tool schemas advertise JSON Schema 2020-12, not draft-07

`packages/cli/scripts/patch-mcp-sdk-dialect.cjs` (new file),
`packages/cli/package.json`

Every ktx tool failed in Claude Code before its handler ran, on an
`invalid outputSchema: ... unsupported dialect` error.

**This is not a ktx bug, and ktx's own source is untouched.** The MCP SDK
hardcodes `'draft-7'` in `mapMiniTarget()` when no target is passed, and
`mcp.js` never passes one
([typescript-sdk#745](https://github.com/modelcontextprotocol/typescript-sdk/issues/745)
— still hardcoded in SDK 1.30.0, so upgrading the SDK does not help). Clients
that validate with a 2020-12-only validator (Claude Code uses Ajv 2020) reject
the dialect and refuse to run the tool.

The patch script flips that one branch inside `node_modules`. Because it edits a
dependency rather than ktx, it is wired to `postinstall` so it survives
reinstalls. It is idempotent and never fails an install — on any surprise it
warns and exits 0. Re-run it any time to apply it or to check it, then restart
the daemon:

```bash
node scripts/patch-mcp-sdk-dialect.cjs && ktx mcp stop && ktx mcp start
```

### 4. Ctrl-C can stop an ingest

`packages/cli/src/managed-local-embeddings.ts`

`ensureManagedLocalEmbeddingsDaemon` announced the embedding daemon with the
**animated** clack spinner. `@clack/core`'s `block()` seizes stdin through
`readline.createInterface`, which calls `setRawMode(true)`, and stopping the
spinner never restores cooked mode. Since this runs a few seconds into every
`ktx ingest` that uses the managed embeddings daemon, the console stayed **raw**
for the rest of the run — and a raw console never translates a Ctrl-C keypress
into a signal at all, so the ingest could not be interrupted no matter how many
times it was pressed.

Measured on Windows, with the console input mode read from outside the process:

| build | console input mode during an ingest |
|---|---|
| before | cooked, then flips to `0x0208` (raw) ~7s in and stays there |
| after | `0x01F7` (cooked) for the whole run |

`clack.ts` already documents this hazard directly above `createStaticCliSpinner`
(*"the animated clack spinner seizes stdin via `@clack/core`'s `block()` and
leaves it dirty"*); this call site simply used the wrong one of the two. The fix
switches to the static spinner and adds a `restoreCookedStdin()` call after it
stops, so a future clack change cannot reintroduce the leak. The only visible
difference is that the two daemon lines no longer animate.

Not yet reported upstream — worth doing, since it affects any ktx install using
`sentence-transformers` embeddings.

### 5. Postgres foreign keys are read from `pg_catalog`, not `information_schema`

`connectors/postgres/connector.ts`. On Postgres, ktx discovered **zero** foreign
keys whenever the connecting account did not **own** the tables. Primary keys
came back complete at the same time, so nothing looked broken — the semantic
layer simply had no declared relationships in it.

The two queries differ by a single join. The primary-key query reads
`information_schema.table_constraints` joined to `key_column_usage`; both are
visible to anyone holding *any privilege other than SELECT*. The foreign-key
query additionally joined `information_schema.constraint_column_usage`, which
Postgres exposes only for tables **owned by a currently enabled role**.
Ownership, not privilege. A fully-granted non-owner gets zero rows from that
view, and the inner join discards every foreign key.

Nothing warns. `tryConstraintQuery` reports a failure only when the query
*throws* `42501` / `42P01`, and a privilege-filtered view does not throw — it
returns no rows. So the scan writes `"foreignKeys": []` alongside
`"warnings": []`, and ktx falls back to *inferring* relationships with the LLM
and embeddings, spending tokens to re-derive keys the database already declares.

Measured against a 307-table Fineract schema holding 531 declared foreign keys,
connecting as a role with SELECT/INSERT/UPDATE/DELETE/REFERENCES/TRIGGER that
owns nothing:

| | before | after |
|---|---|---|
| primary keys captured | 299 / 299 | 299 / 299 |
| foreign keys captured | **0 / 531** | **531 / 531** |
| `constraint_column_usage` rows visible | 0 | 0 (no longer used) |
| relationship detection | inferred: 131 accepted, 431 rejected, 306 validation queries | declared keys used directly |

The fix queries `pg_catalog.pg_constraint`, unnesting `conkey`/`confkey` `WITH
ORDINALITY` so composite keys keep their column pairing. `pg_catalog` has no
ownership gate, so it works without granting anything, and the result shape is
unchanged.

> **Upgrading re-describes tables once.** Foreign keys are part of a table's
> hash, so tables that gain keys count as changed — 235 of 307 on the schema
> above. That pass is one-off and correct; afterwards the cache behaves normally.

Not yet reported upstream — worth doing, since it silently degrades every
Postgres project whose ktx account is not the table owner, which is the normal
setup for a read-only analytics user.

### 6. A descriptions stage that finished with gaps is no longer cached as complete

`connectors/../context/scan/local-enrichment.ts`, `types.ts`.

A run that ended with tables still undescribed — an LLM rate limit, a dropped
database connection, anything that leaves nulls — stored that result as a
**completed** stage keyed on the stage hash. Every later `ktx ingest` whose hash
matched found the completed row, handed the gap straight back, finished in
seconds and printed a clean tick. The gap became permanent, and the only record
was easy to miss: enrichment warnings are written to the run's
`scan-report.json` (its `warnings` array), **not** to `warnings.json`, which the
structural scan writes and which stays empty regardless. Tables never attempted
raise no `enrichment_failed` of their own either, so even that array can look
clean.

Observed on a 349-table project. Two rows cached as `status=completed`, each
carrying `217/349` descriptions and 132 nulls; the scan report showed
`resumedStages: ["descriptions", ...]`, which is pushed only in the
short-circuit branch. A plain `ktx ingest` took 13 seconds and reported success
with 132 descriptions still missing. `--stages descriptions` was the only escape.

The stage runner now takes an optional `cacheable(output)` predicate, applied on
**both** sides:

- **write** — a result that fails the predicate is returned but never stored, so
  a gap cannot be cached in the first place;
- **read** — a completed row that fails the predicate is treated as a miss, so
  rows written before this fix (or by another machine) stop being replayed and
  the project heals itself on the next run rather than needing a manual flag.

For descriptions the predicate is
`output.every((update) => isEnrichedDescriptionUpdate(update))`. Because
`generateDescriptions` returns the full table set in snapshot order — recovered,
freshly enriched, or `nullDescriptionUpdate` for anything still missing — a
single gap fails it. The stage then re-enters `compute()`, change 2's per-table
resume recovers everything already described, and only the missing tables reach
the LLM.

A `descriptions_incomplete` warning now names the count, so an incomplete run
says so instead of passing silently. Look for it in the run's
`scan-report.json` under `warnings` — not `warnings.json`, which carries the
structural scan's warnings and is a different file:

```json
{ "code": "descriptions_incomplete",
  "message": "43 of 349 tables have no AI description. This result was not cached, so re-running `ktx ingest` once the LLM is available will describe only those tables.",
  "metadata": { "undescribed": 43, "total": 349 } }
```

> **Consequence worth knowing.** While any table is undescribed, the descriptions
> stage is not cacheable, so every run re-enters it. That costs one cheap
> re-entry, not re-generation — the per-table hashes mean only the missing tables
> are sent. A table that can never be described keeps the stage uncacheable
> indefinitely; the `descriptions_incomplete` warning tells you which.

Not reported upstream yet.

## Known limitation: pin Claude Code to 2.1.233

Not fixed by this build. ktx 0.16.x asserts that only its own known built-in
tools are active in the Claude Code session it spawns. Newer Claude Code versions
ship additional tools — 2.1.238 adds `Monitor`, `PushNotification` and
`RemoteTrigger` — ktx does not disable them, its isolation assert fires, and
`ktx status` reports:

```
LLM  claude-code · sonnet  x Claude Code authentication is not usable.
     Claude Code runtime isolation failed: tools=Monitor,PushNotification,RemoteTrigger
```

even though `claude -p "reply with OK"` answers fine in the same shell. This is
**not** a login problem, and no amount of `/login` fixes it. Downgrading Claude
Code to 2.1.233 turns it into `OK local Claude Code session authenticated`.

To fix it in your own rebuild, add those names to `BUILTIN_TOOLS` in
`packages/cli/src/context/llm/claude-code-runtime.ts`. This build ships
upstream's list of 12 unchanged.

## Rebuild from source

**This README is self-contained.** With only this file and the upstream
repository you can reproduce this exact package — the complete patch is in
[Appendix: the complete patch](#appendix-the-complete-patch) at the end, and
`ktx-fixes.diff` in this repo is a byte-identical copy of it.

### Prerequisites

| | |
|---|---|
| Node | >= 22 (built with 22.18.0) |
| pnpm | 11.4.0 — upstream's `packageManager`; `corepack enable` picks it up |
| uv | only for regenerating the Python wheel; must be **exactly** the version upstream pins (`==0.11.11` for 0.16.0). Not needed if you copy the assets, see step 5. |

### Steps

**1. Get a pristine upstream checkout at the tag this patch targets.**

```bash
git clone https://github.com/Kaelio/ktx.git
cd ktx
git checkout v0.16.0        # commit a6dd8cf
```

**2. Write the patch to a file.** Either use `ktx-fixes.diff` from this repo, or
copy the fenced block in the appendix below into `ktx-fixes.diff` verbatim.

**3. Apply it.** Check first — it must apply with no fuzz:

```bash
git apply --check ktx-fixes.diff && git apply ktx-fixes.diff
git status --porcelain
```

Expected — six modified files plus one new one:

```
 M packages/cli/package.json
 M packages/cli/src/connectors/postgres/connector.ts
 M packages/cli/src/context/scan/enrichment-state.ts
 M packages/cli/src/context/scan/local-enrichment-artifacts.ts
 M packages/cli/src/context/scan/local-enrichment.ts
 M packages/cli/src/managed-local-embeddings.ts
?? packages/cli/scripts/patch-mcp-sdk-dialect.cjs
```

The patch is self-contained: it creates `patch-mcp-sdk-dialect.cjs` and makes the
`package.json` edits that go with it — `name`, `version`, `description`, the
`files` entry for the script, the `postinstall` hook, and dropping
`publishConfig.provenance`, which only works from upstream's own CI. `pnpm
install` runs that `postinstall` in your dev tree too; harmless and idempotent.

**4. Install and build.**

```bash
pnpm install
cd packages/cli
pnpm run build
```

`pnpm run build` must exit 0. It runs `tsc`, writes the telemetry schema, copies
runtime assets, and prepares the CLI bin.

> `pnpm install --ignore-scripts` also works and skips the postinstall; the
> script is re-run for real by consumers at install time either way.

**5. Make sure the Python runtime assets are present — this step is easy to miss.**

`pnpm run build` runs `scripts/copy-runtime-assets.mjs`, and `files: ["dist",
"assets"]` ships the result — but on a **pristine** clone there is nothing to
copy yet and the step fails silently. `npm pack` then produces **1210** files
instead of 1212, missing `assets/python/manifest.json` and
`assets/python/kaelio_ktx-0.16.0-py3-none-any.whl`, and the installed CLI dies
with `Missing bundled Python runtime manifest` the first time it needs local
embeddings.

Either regenerate them (needs the exact pinned `uv`):

```bash
cd ../..                     # repo root
pnpm run artifacts:build-runtime
cd packages/cli
```

or copy them from the previous tarball — the wheel is upstream's own Python
daemon and none of these fixes touch `python/`, so this is equivalent:

```bash
mkdir -p assets/python
tar -xzOf /path/to/previous/ktx-fixed.tgz package/assets/python/manifest.json   > assets/python/manifest.json
tar -xzOf /path/to/previous/ktx-fixed.tgz package/assets/python/kaelio_ktx-0.16.0-py3-none-any.whl   > assets/python/kaelio_ktx-0.16.0-py3-none-any.whl
```

On Windows/Git Bash, `tar` reads a leading `C:` as a hostname — copy the tarball
to a relative path first, or pass `--force-local`.

**6. Pack.**

```bash
npm pack
```

Produces `charbelkh-ktx-0.16.5.tgz`. Rename to `ktx-fixed.tgz` and commit it
here; the install command at the top of this README keeps working unchanged.

### Verify the rebuild

Run these against the tarball you just produced. All must pass:

```bash
T=ktx-fixed.tgz
tar -tzf $T | wc -l                                    # 1212 (1213 = see below)
tar -tzf $T | grep -c 'assets/python'                  # 2
tar -tzf $T | grep -c 'scripts/patch-mcp-sdk-dialect'  # 1
tar -xzOf $T package/package.json | grep '"version"'   # 0.16.5

# one grep per fix, against the compiled output
tar -xzOf $T package/dist/context/scan/enrichment-state.js            | grep -c stableSnapshotHashInput        # >0  fix 1
tar -xzOf $T package/dist/context/scan/enrichment-state.js            | grep -c computeKtxTableDescriptionHash # >0  fix 1
tar -xzOf $T package/dist/context/scan/local-enrichment-artifacts.js  | grep -c tableHashes                    # >0  fix 2
tar -xzOf $T package/dist/managed-local-embeddings.js                 | grep -c restoreCookedStdin             # >0  fix 4
tar -xzOf $T package/dist/managed-local-embeddings.js                 | grep -c createStaticCliSpinner         # >0  fix 4
tar -xzOf $T package/dist/connectors/postgres/connector.js            | grep -c pg_catalog.pg_constraint       # >0  fix 5
```

A count of **1213** means `pnpm run type-check` ran before packing: it writes
`dist/.tsbuildinfo.test`, and `files: ["dist", "assets"]` sweeps it in. Delete
that file and pack again.

Then install it and confirm the version:

```bash
npm uninstall -g @kaelio/ktx @charbelkh/ktx
npm i -g ./ktx-fixed.tgz
ktx --version                                          # @charbelkh/ktx 0.16.5
ktx admin runtime install --feature local-embeddings --yes
```

The embeddings runtime is installed **per ktx version** under
`~/.ktx/runtime/<version>/`, so it reports `missing` after every version bump
until you run that last command again.

### Porting to a newer upstream release

Apply the patch to that tag instead. All six changes are small and localised; if
a hunk stops applying, the numbered sections above say what each one has to
accomplish, in enough detail to redo it by hand.

### What has been verified

Against a pristine `git clone` of upstream at tag `v0.16.0` (commit `a6dd8cf`):

- `git apply --check ktx-fixes.diff` applies cleanly, all six files.
- After applying, `packages/cli/package.json` is **identical in every field** to
  the `package.json` inside `ktx-fixed.tgz`, and
  `packages/cli/scripts/patch-mcp-sdk-dialect.cjs` is byte-identical to the
  shipped copy. (Line endings and a byte-order mark may differ — the shipped
  copies were written on Windows. No content differs.)
- The shipped `dist/` carries all six fixes: `stableSnapshotHashInput` and
  `computeKtxTableDescriptionHash` in `dist/context/scan/enrichment-state.js`,
  `tableHashes` in `dist/context/scan/local-enrichment-artifacts.js`,
  `restoreCookedStdin` and `createStaticCliSpinner` in
  `dist/managed-local-embeddings.js`, `cacheable` in
  `dist/context/scan/local-enrichment.js`, `pg_catalog.pg_constraint` in
  `dist/connectors/postgres/connector.js`, and `scripts/patch-mcp-sdk-dialect.cjs`
  in the package.
- Fix 5 was exercised end to end against a live 307-table Fineract schema with
  531 declared foreign keys, connecting as a non-owner: `foreign-keys.json` went
  from `{"foreignKeys": []}` to 531 entries, `warnings.json` stayed empty, and
  primary keys were 299/299 before and after.
- Fix 6's read side was exercised against a real project holding three cached
  `scan:descriptions` rows with gaps (`217/349` twice, `348/349` once). Before,
  a plain `ktx ingest` short-circuited in 13 seconds; after, it rejected all
  three rows, re-entered the stage, recovered 217 by per-table hash and resumed
  generation at table 218. The write side follows from the same predicate.
- `pnpm run type-check` passes on a fresh apply — both `tsconfig.json` and
  `tsconfig.test.json`. Builds up to 0.16.4 failed the test half: change 2
  altered the `load()` return type without updating the two resume-store mocks
  in `local-enrichment.test.ts`, which this patch now repairs.

## This is temporary

Changes 1 and 2 are reported upstream. **Once they ship, drop this build** and go
back to the official one:

```bash
npm uninstall -g @charbelkh/ktx
npm i -g @kaelio/ktx
```

Nothing to clean up after that. The MCP SDK patch lives inside this package's own
`node_modules`, so it is removed along with the package and no global install is
left modified. Your ktx projects are untouched — `ktx.yaml`, `semantic-layer/`,
`wiki/` and `raw-sources/` are just files — and the shared Python runtime under
`~/.ktx/runtime` stays unless you delete it yourself.

## Appendix: the complete patch

Everything above is narrative; this is the artifact. It is byte-identical to
`ktx-fixes.diff` in this repo — reproduced here so this README alone is enough to
rebuild the package.

To use it, copy the contents of the fenced block into a file named
`ktx-fixes.diff` and follow [Rebuild from source](#rebuild-from-source). It
applies to a pristine upstream checkout at tag `v0.16.0` (commit `a6dd8cf`) and
touches nine files:

| File | Change |
|---|---|
| `packages/cli/package.json` | 3 — name, version, `files`, `postinstall`, drop `publishConfig.provenance` |
| `packages/cli/scripts/patch-mcp-sdk-dialect.cjs` *(new)* | 3 |
| `packages/cli/src/context/scan/enrichment-state.ts` | 1 |
| `packages/cli/src/context/scan/local-enrichment-artifacts.ts` | 2 |
| `packages/cli/src/context/scan/local-enrichment.ts` | 2, 6 |
| `packages/cli/src/context/scan/types.ts` | 6 — the `descriptions_incomplete` warning code |
| `packages/cli/src/managed-local-embeddings.ts` | 4 |
| `packages/cli/src/connectors/postgres/connector.ts` | 5 |
| `packages/cli/test/context/scan/local-enrichment.test.ts` | 2 — repairs the two resume-store mocks change 2 left type-broken |

If you edit one copy, regenerate the other — `git diff > ktx-fixes.diff` from the
patched checkout, having first `git add -N` the new script so it is included.
They must stay in sync; verify with:

```bash
diff <(sed -n '/^```diff$/,/^```$/p' README.md | sed '1d;$d') ktx-fixes.diff
```

```diff
diff --git a/packages/cli/package.json b/packages/cli/package.json
index af4bf65..97bbab8 100644
--- a/packages/cli/package.json
+++ b/packages/cli/package.json
@@ -1,7 +1,7 @@
 {
-  "name": "@kaelio/ktx",
-  "version": "0.16.0",
-  "description": "Standalone ktx context layer for data agents",
+  "name": "@charbelkh/ktx",
+  "version": "0.16.5",
+  "description": "Patched fork of @kaelio/ktx 0.16.0 - ktx ingest reuses already-generated AI descriptions instead of regenerating them (upstream issues #347, #348)",
   "author": {
     "name": "Kaelio",
     "url": "https://www.kaelio.com"
@@ -25,17 +25,18 @@
   },
   "files": [
     "dist",
-    "assets"
+    "assets",
+    "scripts/patch-mcp-sdk-dialect.cjs"
   ],
   "publishConfig": {
-    "access": "public",
-    "provenance": true
+    "access": "public"
   },
   "scripts": {
     "assets:demo": "node scripts/build-demo-assets.mjs",
     "build": "tsc -p tsconfig.json && node dist/telemetry/schema-writer.js src/telemetry/events.schema.json ../../python/ktx-daemon/src/ktx_daemon/telemetry/events.schema.json && node scripts/copy-runtime-assets.mjs && node ../../scripts/prepare-cli-bin.mjs",
     "clean": "node -e \"fs.rmSync('dist', { recursive: true, force: true })\"",
     "docs:commands": "pnpm run build && node dist/print-command-tree.js",
+    "postinstall": "node scripts/patch-mcp-sdk-dialect.cjs",
     "smoke": "vitest run test/standalone-smoke.test.ts test/example-smoke.test.ts --testTimeout 30000",
     "test": "vitest run --exclude test/standalone-smoke.test.ts --exclude test/example-smoke.test.ts --exclude test/setup-databases.test.ts --exclude test/scan.test.ts --exclude test/commands/connection-metabase-setup.test.ts --exclude test/setup-models.test.ts --exclude test/setup-sources.test.ts --exclude test/setup.test.ts --exclude test/connection.test.ts --exclude test/setup-embeddings.test.ts --exclude test/ingest.test.ts --exclude test/commands/connection-mapping.test.ts --exclude test/ingest-viz.test.ts --exclude test/demo.test.ts --exclude test/setup-project.test.ts --exclude test/sl.test.ts --exclude test/local-scan-connectors.test.ts --exclude test/commands/connection-notion.test.ts --exclude test/context/scan/local-scan.test.ts --exclude test/context/mcp/local-project-ports.test.ts --exclude test/context/ingest/local-stage-ingest.test.ts --exclude test/context/sl/pglite-sl-search-prototype.test.ts --exclude test/context/core/git.service.test.ts --exclude test/context/ingest/local-adapters.test.ts --exclude test/context/ingest/local-bundle-ingest.test.ts --exclude test/context/ingest/local-metabase-ingest.test.ts --exclude test/context/sl/local-sl.test.ts --exclude test/context/search/pglite-owner-process.test.ts --exclude test/context/scan/local-enrichment-artifacts.test.ts --exclude test/context/search/pglite-spike.test.ts --exclude test/context/wiki/local-knowledge.test.ts --exclude test/context/sl/local-query.test.ts --exclude test/context/scan/relationship-review-decisions.test.ts --exclude test/context/scan/relationship-profiling.test.ts",
     "test:slow": "vitest run test/setup-databases.test.ts test/scan.test.ts test/commands/connection-metabase-setup.test.ts test/setup-models.test.ts test/setup-sources.test.ts test/setup.test.ts test/connection.test.ts test/setup-embeddings.test.ts test/ingest.test.ts test/commands/connection-mapping.test.ts test/ingest-viz.test.ts test/demo.test.ts test/setup-project.test.ts test/sl.test.ts test/local-scan-connectors.test.ts test/commands/connection-notion.test.ts test/context/scan/local-scan.test.ts test/context/mcp/local-project-ports.test.ts test/context/ingest/local-stage-ingest.test.ts test/context/sl/pglite-sl-search-prototype.test.ts test/context/core/git.service.test.ts test/context/ingest/local-adapters.test.ts test/context/ingest/local-bundle-ingest.test.ts test/context/ingest/local-metabase-ingest.test.ts test/context/sl/local-sl.test.ts test/context/search/pglite-owner-process.test.ts test/context/scan/local-enrichment-artifacts.test.ts test/context/search/pglite-spike.test.ts test/context/wiki/local-knowledge.test.ts test/context/sl/local-query.test.ts test/context/scan/relationship-review-decisions.test.ts test/context/scan/relationship-profiling.test.ts --testTimeout 30000",
diff --git a/packages/cli/scripts/patch-mcp-sdk-dialect.cjs b/packages/cli/scripts/patch-mcp-sdk-dialect.cjs
new file mode 100644
index 0000000..f4ed9d3
--- /dev/null
+++ b/packages/cli/scripts/patch-mcp-sdk-dialect.cjs
@@ -0,0 +1,67 @@
+#!/usr/bin/env node
+// Make @modelcontextprotocol/sdk advertise JSON Schema 2020-12 instead of draft-07.
+//
+// The SDK converts ktx's Zod v4 schemas for tools/list via toJsonSchemaCompat(),
+// whose mapMiniTarget() returns 'draft-7' when no target is passed — and mcp.js
+// never passes one. Zod then stamps draft-07 on every schema the server emits, and
+// clients validating with a 2020-12-only validator (Claude Code uses Ajv 2020)
+// reject the dialect and refuse to run the tool at all.
+// Upstream bug: https://github.com/modelcontextprotocol/typescript-sdk/issues/745
+// (still hardcoded in SDK 1.30.0, so bumping the SDK does not help).
+//
+// Flips only the "caller passed no target" branch, so an explicit 'draft-7'
+// request is still honoured. ktx's own code is untouched.
+//
+// Wired to postinstall so it survives reinstalls. Idempotent — re-run any time:
+//   node scripts/patch-mcp-sdk-dialect.cjs
+// Never fails an install: on any surprise it warns and exits 0.
+
+const fs = require('node:fs');
+const path = require('node:path');
+
+const FROM =
+    /(function mapMiniTarget\([^)]*\)\s*\{\s*if\s*\(\s*!\s*\w+\s*\)\s*(?:\r?\n)?\s*return\s+)(['"])draft-7\2/;
+const DONE =
+    /function mapMiniTarget\([^)]*\)\s*\{\s*if\s*\(\s*!\s*\w+\s*\)\s*(?:\r?\n)?\s*return\s+(['"])draft-2020-12\1/;
+
+function findSdk() {
+    const pkg = path.join(__dirname, '..'); // scripts/ sits at the package root
+    const candidates = [
+        path.join(pkg, 'node_modules', '@modelcontextprotocol', 'sdk'), // nested
+        path.join(pkg, '..', '@modelcontextprotocol', 'sdk'), // hoisted, unscoped pkg
+        path.join(pkg, '..', '..', '@modelcontextprotocol', 'sdk'), // hoisted, scoped pkg
+    ];
+    return candidates.find((dir) => fs.existsSync(path.join(dir, 'package.json'))) ?? null;
+}
+
+try {
+    const sdk = findSdk();
+    if (!sdk) {
+        throw new Error('@modelcontextprotocol/sdk not found');
+    }
+    const { version } = require(path.join(sdk, 'package.json'));
+
+    let patched = 0;
+    let already = 0;
+    for (const build of ['esm', 'cjs']) {
+        const file = path.join(sdk, 'dist', build, 'server', 'zod-json-schema-compat.js');
+        if (!fs.existsSync(file)) continue;
+        const src = fs.readFileSync(file, 'utf8');
+        if (DONE.test(src)) {
+            already += 1;
+        } else if (FROM.test(src)) {
+            fs.writeFileSync(file, src.replace(FROM, '$1$2draft-2020-12$2'), 'utf8');
+            patched += 1;
+        }
+    }
+
+    if (patched > 0) {
+        console.log(`[ktx] mcp-sdk ${version}: tool schemas now advertise JSON Schema 2020-12`);
+    } else if (already > 0) {
+        console.log(`[ktx] mcp-sdk ${version}: dialect patch already applied`);
+    } else {
+        console.warn(`[ktx] mcp-sdk ${version}: dialect patch did not match — upstream may have fixed #745`);
+    }
+} catch (err) {
+    console.warn(`[ktx] mcp-sdk dialect patch skipped: ${err.message}`);
+}
diff --git a/packages/cli/src/connectors/postgres/connector.ts b/packages/cli/src/connectors/postgres/connector.ts
index c2c0f6d..b5b8a4f 100644
--- a/packages/cli/src/connectors/postgres/connector.ts
+++ b/packages/cli/src/connectors/postgres/connector.ts
@@ -697,29 +697,37 @@ export class KtxPostgresScanConnector implements KtxScanConnector {
     if (!primaryKeysResult.ok) {
       snapshotWarnings.push(primaryKeysResult.warning);
     }
+    // Foreign keys come from pg_catalog, not information_schema.
+    // information_schema.constraint_column_usage only exposes columns of tables
+    // OWNED BY a currently enabled role, so a fully-granted non-owner sees zero
+    // rows there and the join silently drops every foreign key -- while the
+    // primary-key query above, which never touches that view, still returns all
+    // of them. pg_catalog.pg_constraint has no ownership gate.
     const foreignKeysResult = await tryConstraintQuery(
       { schema, kind: 'foreign_key', isDeniedError },
       () =>
         this.queryRaw<PostgresForeignKeyRow>(
           `
       SELECT
-        tc.table_name,
-        kcu.column_name,
-        ccu.table_schema AS foreign_table_schema,
-        ccu.table_name AS foreign_table_name,
-        ccu.column_name AS foreign_column_name,
-        tc.constraint_name
-      FROM information_schema.table_constraints AS tc
-      JOIN information_schema.key_column_usage AS kcu
-        ON tc.constraint_name = kcu.constraint_name
-        AND tc.table_schema = kcu.table_schema
-      JOIN information_schema.constraint_column_usage AS ccu
-        ON ccu.constraint_name = tc.constraint_name
-        AND ccu.table_schema = tc.table_schema
-      WHERE tc.constraint_type = 'FOREIGN KEY'
-        AND tc.table_schema = $1
-        ${tableConstraintScopeClause}
-      ORDER BY tc.table_name, kcu.column_name
+        c.relname AS table_name,
+        a.attname AS column_name,
+        fn.nspname AS foreign_table_schema,
+        fc.relname AS foreign_table_name,
+        fa.attname AS foreign_column_name,
+        con.conname AS constraint_name
+      FROM pg_catalog.pg_constraint con
+      JOIN pg_catalog.pg_class c ON c.oid = con.conrelid
+      JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
+      JOIN pg_catalog.pg_class fc ON fc.oid = con.confrelid
+      JOIN pg_catalog.pg_namespace fn ON fn.oid = fc.relnamespace
+      JOIN LATERAL unnest(con.conkey) WITH ORDINALITY AS ck(attnum, ord) ON true
+      JOIN LATERAL unnest(con.confkey) WITH ORDINALITY AS fk(attnum, ord) ON ck.ord = fk.ord
+      JOIN pg_catalog.pg_attribute a ON a.attrelid = c.oid AND a.attnum = ck.attnum
+      JOIN pg_catalog.pg_attribute fa ON fa.attrelid = fc.oid AND fa.attnum = fk.attnum
+      WHERE con.contype = 'f'
+        AND n.nspname = $1
+        ${pgCatalogScopeClause}
+      ORDER BY c.relname, con.conname, ck.ord
       `,
           [schema, ...scopeValues],
         ),
diff --git a/packages/cli/src/context/scan/enrichment-state.ts b/packages/cli/src/context/scan/enrichment-state.ts
index 84b6224..3fbc3a4 100644
--- a/packages/cli/src/context/scan/enrichment-state.ts
+++ b/packages/cli/src/context/scan/enrichment-state.ts
@@ -1,6 +1,12 @@
 import { stableContentHash } from '../cache/content-result-cache.js';
 import type { KtxScanRelationshipConfig } from '../project/config.js';
-import type { KtxScanEnrichmentStage, KtxScanEnrichmentStateSummary, KtxScanMode, KtxSchemaSnapshot } from './types.js';
+import type {
+  KtxScanEnrichmentStage,
+  KtxScanEnrichmentStateSummary,
+  KtxScanMode,
+  KtxSchemaSnapshot,
+  KtxSchemaTable,
+} from './types.js';
 
 /**
  * Canonical enrichment-stage registry. The `--stages` CLI parser validates
@@ -99,13 +105,26 @@ export interface KtxRelationshipsStageHashInput {
   llmIdentity: KtxScanLlmIdentity;
 }
 
+/**
+ * [fix bug1] Schema-identifying view of a snapshot for the enrichment-stage
+ * hashes. Excludes `extractedAt` — a per-scan wall-clock timestamp set fresh on
+ * every introspection — so an unchanged schema keeps a stable hash across
+ * re-scans (the timestamp otherwise re-keys every stage and forces a full
+ * LLM re-generation). Mirrors `stableLiveDatabaseHashContent` in
+ * local-stage-ingest.ts, which already strips it for the ingest work-unit hash.
+ */
+function stableSnapshotHashInput(snapshot: KtxSchemaSnapshot): Omit<KtxSchemaSnapshot, 'extractedAt'> {
+  const { extractedAt: _extractedAt, ...stable } = snapshot;
+  return stable;
+}
+
 export function computeKtxDescriptionsStageHash(input: KtxDescriptionsStageHashInput): string {
-  return stableContentHash({ snapshot: input.snapshot, llmIdentity: input.llmIdentity });
+  return stableContentHash({ snapshot: stableSnapshotHashInput(input.snapshot), llmIdentity: input.llmIdentity });
 }
 
 export function computeKtxEmbeddingsStageHash(input: KtxEmbeddingsStageHashInput): string {
   return stableContentHash({
-    snapshot: input.snapshot,
+    snapshot: stableSnapshotHashInput(input.snapshot),
     embeddingIdentity: input.embeddingIdentity,
     descriptionDigest: input.descriptionDigest,
   });
@@ -113,12 +132,31 @@ export function computeKtxEmbeddingsStageHash(input: KtxEmbeddingsStageHashInput
 
 export function computeKtxRelationshipsStageHash(input: KtxRelationshipsStageHashInput): string {
   return stableContentHash({
-    snapshot: input.snapshot,
+    snapshot: stableSnapshotHashInput(input.snapshot),
     relationshipSettings: input.relationshipSettings,
     llmIdentity: input.llmIdentity,
   });
 }
 
+/**
+ * [fix bug2] Per-table description cache key. A table's AI description depends
+ * only on that table's own structure and the description LLM identity — never on
+ * which other tables are in the batch. Keying resume per table (instead of
+ * gating the whole batch on {@link computeKtxDescriptionsStageHash}) lets an
+ * added/removed table reuse every unchanged table's description.
+ */
+export function computeKtxTableDescriptionHash(input: {
+  table: KtxSchemaTable;
+  llmIdentity: KtxScanLlmIdentity;
+}): string {
+  // [fix bug3] `estimatedRows` is a live row count that drifts on every scan of an
+  // active database. Including it re-keys unchanged tables and drops their
+  // descriptions — the same class of bug as `extractedAt` above. Hash only the
+  // structure a description actually depends on.
+  const { estimatedRows: _estimatedRows, ...stableTable } = input.table;
+  return stableContentHash({ table: stableTable, llmIdentity: input.llmIdentity });
+}
+
 /**
  * Content digest of the resolved per-column description text the embeddings
  * stage consumes. Folding it into the embeddings hash content-addresses
diff --git a/packages/cli/src/context/scan/local-enrichment-artifacts.ts b/packages/cli/src/context/scan/local-enrichment-artifacts.ts
index fa18777..4488840 100644
--- a/packages/cli/src/context/scan/local-enrichment-artifacts.ts
+++ b/packages/cli/src/context/scan/local-enrichment-artifacts.ts
@@ -338,16 +338,25 @@ function descriptionsProgressPath(connectionId: string): string {
 interface DescriptionsProgressRecord {
   inputHash: string;
   descriptions: LocalDescriptionUpdates;
+  /** [fix bug2] Per-table hash (structure + llmIdentity) keyed by tableRefKey. */
+  tableHashes?: Record<string, string>;
+}
+
+/** [fix bug2] Prior descriptions plus their per-table hashes; the caller recovers per table. */
+export interface KtxScanDescriptionResumeLoad {
+  descriptions: LocalDescriptionUpdates;
+  tableHashes: Record<string, string>;
 }
 
 export interface KtxScanDescriptionResumeStore {
-  /** Prior enriched descriptions when the durable record matches `inputHash`, else null. */
-  load(inputHash: string): Promise<LocalDescriptionUpdates | null>;
+  /** The whole prior record (descriptions + per-table hashes), or null when none exists. */
+  load(): Promise<KtxScanDescriptionResumeLoad | null>;
   /** Persist the descriptions so far + the manifest shards that gained a table this batch. */
   flush(input: {
     inputHash: string;
     snapshot: KtxSchemaSnapshot;
     descriptionUpdates: LocalDescriptionUpdates;
+    tableHashes: Record<string, string>;
     changedTableNames: ReadonlySet<string>;
   }): Promise<void>;
 }
@@ -360,7 +369,7 @@ export function createKtxScanDescriptionResumeStore(deps: {
 }): KtxScanDescriptionResumeStore {
   const path = descriptionsProgressPath(deps.connectionId);
   return {
-    async load(inputHash) {
+    async load() {
       let content: string;
       try {
         ({ content } = await deps.project.fileStore.readFile(path));
@@ -369,18 +378,20 @@ export function createKtxScanDescriptionResumeStore(deps: {
       }
       try {
         const record = JSON.parse(content) as DescriptionsProgressRecord | null;
-        // A changed inputHash (schema or enrichment settings changed) ignores the
-        // prior record and recomputes — spec-19's inputHash-gated resume semantics.
-        if (!record || record.inputHash !== inputHash || !Array.isArray(record.descriptions)) {
+        // [fix bug2] Return the whole record + per-table hashes; the caller recovers
+        // each table whose own hash still matches (per-table resume), so an unrelated
+        // table change no longer discards the rest. A pre-fix record (no tableHashes)
+        // recovers nothing and safely recomputes.
+        if (!record || !Array.isArray(record.descriptions)) {
           return null;
         }
-        return record.descriptions;
+        return { descriptions: record.descriptions, tableHashes: record.tableHashes ?? {} };
       } catch {
         return null;
       }
     },
-    async flush({ inputHash, snapshot, descriptionUpdates, changedTableNames }) {
-      const record: DescriptionsProgressRecord = { inputHash, descriptions: descriptionUpdates };
+    async flush({ inputHash, snapshot, descriptionUpdates, tableHashes, changedTableNames }) {
+      const record: DescriptionsProgressRecord = { inputHash, descriptions: descriptionUpdates, tableHashes };
       await writeJsonArtifact(
         deps.project,
         path,
diff --git a/packages/cli/src/context/scan/local-enrichment.ts b/packages/cli/src/context/scan/local-enrichment.ts
index 8b4da12..f091a58 100644
--- a/packages/cli/src/context/scan/local-enrichment.ts
+++ b/packages/cli/src/context/scan/local-enrichment.ts
@@ -10,6 +10,7 @@ import {
   computeKtxEmbeddingsStageHash,
   computeKtxRelationshipsStageHash,
   computeKtxScanDescriptionDigest,
+  computeKtxTableDescriptionHash,
   KTX_SCAN_ENRICHMENT_STAGES,
   type KtxScanEmbeddingIdentity,
   type KtxScanEnrichmentStateStore,
@@ -352,6 +353,7 @@ async function generateDescriptions(input: {
   context: KtxScanContext;
   providers: KtxLocalScanEnrichmentProviders;
   inputHash: string;
+  llmIdentity: KtxScanLlmIdentity;
   resumeStore?: KtxScanDescriptionResumeStore | null;
   progress?: KtxProgressPort;
   warnings?: KtxScanWarning[];
@@ -384,11 +386,26 @@ async function generateDescriptions(input: {
   // Resume: recover already-enriched tables (inputHash-gated) and re-issue LLM
   // calls only for the remainder. A failed/skipped table carries null descriptions
   // and is not recovered, so it is retried.
-  const recovered = input.resumeStore ? ((await input.resumeStore.load(input.inputHash)) ?? []) : [];
+  // [fix bug2] Recover per table: each table gated on its own hash (structure +
+  // llmIdentity), so adding/removing an unrelated table no longer discards the rest.
+  const currentTableHashes = new Map<string, string>();
+  for (const t of input.snapshot.tables) {
+    currentTableHashes.set(
+      tableRefKey(tableRef(t)),
+      computeKtxTableDescriptionHash({ table: t, llmIdentity: input.llmIdentity }),
+    );
+  }
+  const prior = input.resumeStore ? await input.resumeStore.load() : null;
+  const priorTableHashes = prior?.tableHashes ?? {};
   const enrichedById = new Map<string, KtxScanDescriptionUpdate>();
-  for (const update of recovered) {
-    if (isEnrichedDescriptionUpdate(update)) {
-      enrichedById.set(tableRefKey(update.table), update);
+  for (const update of prior?.descriptions ?? []) {
+    const key = tableRefKey(update.table);
+    if (
+      isEnrichedDescriptionUpdate(update) &&
+      priorTableHashes[key] !== undefined &&
+      priorTableHashes[key] === currentTableHashes.get(key)
+    ) {
+      enrichedById.set(key, update);
     }
   }
   const remaining = input.snapshot.tables.filter((table) => !enrichedById.has(tableRefKey(tableRef(table))));
@@ -418,6 +435,9 @@ async function generateDescriptions(input: {
         inputHash: input.inputHash,
         snapshot: input.snapshot,
         descriptionUpdates: [...enrichedById.values()],
+        tableHashes: Object.fromEntries(
+          [...enrichedById.keys()].map((key) => [key, currentTableHashes.get(key) ?? '']),
+        ),
         changedTableNames,
       });
     } finally {
@@ -590,6 +610,16 @@ async function runEnrichmentStage<TOutput>(input: {
    * spec-20 per-table resume record.
    */
   forceRecompute?: boolean;
+  /**
+   * [fix bug6] Whether this run's output may be cached as a completed stage.
+   * A stage that finished but produced an incomplete result (e.g. descriptions
+   * the LLM never returned) must NOT be stored: findCompletedStage() would hand
+   * that gap back to every later run with a matching inputHash, which reports
+   * success and never retries. Returning false leaves no completed row, so the
+   * next run re-enters compute() and the per-table resume fills only the gaps.
+   * Omitted means always cacheable, preserving the previous behaviour.
+   */
+  cacheable?: (output: TOutput) => boolean;
   compute: () => Promise<TOutput>;
 }): Promise<TOutput> {
   if (!input.forceRecompute) {
@@ -598,7 +628,12 @@ async function runEnrichmentStage<TOutput>(input: {
       stage: input.stage,
       inputHash: input.inputHash,
     });
-    if (existing) {
+    // [fix bug6] A completed row whose payload is incomplete is treated as a miss.
+    // Rows written before this fix (or by another machine) still carry gaps; without
+    // this the short-circuit would keep handing them back forever and no re-run could
+    // ever heal them. Falling through recomputes, and the write guard below stops the
+    // gap being cached again.
+    if (existing && (input.cacheable?.(existing.output) ?? true)) {
       input.resumedStages.push(input.stage);
       input.completedStages.push(input.stage);
       return existing.output;
@@ -608,6 +643,10 @@ async function runEnrichmentStage<TOutput>(input: {
   try {
     const output = await input.compute();
     input.completedStages.push(input.stage);
+    // [fix bug6] Only cache a result that is actually complete.
+    if (input.cacheable && !input.cacheable(output)) {
+      return output;
+    }
     await input.stateStore?.saveCompletedStage({
       runId: input.runId,
       connectionId: input.connectionId,
@@ -756,12 +795,30 @@ export async function runLocalScanEnrichment(
             context: input.context,
             providers,
             inputHash: descriptionsHash,
+            llmIdentity,
             resumeStore: input.descriptionResumeStore,
             progress: descriptionProgress,
             warnings,
           }),
+        // [fix bug6] Never cache a descriptions result that still has gaps.
+        cacheable: (output) => output.every((update) => isEnrichedDescriptionUpdate(update)),
       });
       descriptionsRanThisInvocation = true;
+      // [fix bug6] Surface the gap. Without this the run prints a clean tick while
+      // tables sit undescribed, and nothing in warnings.json records it either:
+      // tables that were never attempted raise no `enrichment_failed` of their own.
+      const undescribed = descriptions.filter((update) => !isEnrichedDescriptionUpdate(update)).length;
+      if (undescribed > 0) {
+        warnings.push({
+          code: 'descriptions_incomplete',
+          message:
+            `${undescribed} of ${descriptions.length} tables have no AI description. ` +
+            'This result was not cached, so re-running `ktx ingest` once the LLM is ' +
+            'available will describe only those tables.',
+          recoverable: true,
+          metadata: { undescribed, total: descriptions.length },
+        });
+      }
       summary.dataDictionary = input.connector.sampleColumn ? 'completed' : 'skipped';
       summary.tableDescriptions = 'completed';
       summary.columnDescriptions = 'completed';
diff --git a/packages/cli/src/context/scan/types.ts b/packages/cli/src/context/scan/types.ts
index bf72558..ff498ff 100644
--- a/packages/cli/src/context/scan/types.ts
+++ b/packages/cli/src/context/scan/types.ts
@@ -382,6 +382,8 @@ interface KtxScanArtifactPaths {
 }
 
 type KtxScanWarningCode =
+  /** [fix bug6] The descriptions stage finished with tables still undescribed. */
+  | 'descriptions_incomplete'
   | 'connector_capability_missing'
   | 'sampling_failed'
   | 'statistics_failed'
diff --git a/packages/cli/src/managed-local-embeddings.ts b/packages/cli/src/managed-local-embeddings.ts
index 999b38c..ef14084 100644
--- a/packages/cli/src/managed-local-embeddings.ts
+++ b/packages/cli/src/managed-local-embeddings.ts
@@ -1,6 +1,6 @@
 import type { KtxEmbeddingConfig } from './llm/types.js';
 import type { KtxCliIo } from './cli-runtime.js';
-import { createCliSpinner } from './clack.js';
+import { createStaticCliSpinner } from './clack.js';
 import {
   ensureManagedPythonCommandRuntime,
   type KtxManagedPythonInstallPolicy,
@@ -54,6 +54,22 @@ export function managedLocalEmbeddingHealthConfig(input: {
   };
 }
 
+/**
+ * [fix bug6] Belt and braces for the raw-mode leak above: whatever this function
+ * did to stdin, leave the terminal able to deliver Ctrl-C. Cheap and idempotent -
+ * a no-op when stdin is not a raw TTY.
+ */
+function restoreCookedStdin(): void {
+  const stdin = process.stdin as NodeJS.ReadStream & { isRaw?: boolean };
+  if (stdin.isTTY === true && stdin.isRaw === true && typeof stdin.setRawMode === 'function') {
+    try {
+      stdin.setRawMode(false);
+    } catch {
+      // Never let a terminal tidy-up break an ingest.
+    }
+  }
+}
+
 export async function ensureManagedLocalEmbeddingsDaemon(
   options: ManagedLocalEmbeddingsOptions,
 ): Promise<ManagedLocalEmbeddingsDaemon> {
@@ -66,7 +82,16 @@ export async function ensureManagedLocalEmbeddingsDaemon(
     io: options.io,
     feature: 'local-embeddings',
   });
-  const spinner = createCliSpinner(options.io);
+  // [fix bug6] Use the STATIC spinner, not the animated one. The animated clack
+  // spinner seizes stdin through `@clack/core`'s `block()`, which calls
+  // `readline.createInterface` -> `setRawMode(true)`, and stopping it never
+  // restores cooked mode. Because this runs early in every `ktx ingest` that uses
+  // the managed embeddings daemon, the console stayed raw for the whole run - and
+  // a raw console never translates a Ctrl-C keypress into SIGINT, so the ingest
+  // could not be interrupted at all, no matter how many times it was pressed.
+  // clack.ts already documents this hazard above `createStaticCliSpinner`; this
+  // call site simply used the wrong one.
+  const spinner = createStaticCliSpinner(options.io);
   spinner.start('Starting ktx embedding daemon (first run downloads the model)…');
   let daemon: ManagedPythonDaemonStartResult;
   try {
@@ -78,10 +103,12 @@ export async function ensureManagedLocalEmbeddingsDaemon(
     });
   } catch (error) {
     spinner.error('ktx embedding daemon failed to start');
+    restoreCookedStdin();
     throw error;
   }
   const verb = daemon.status === 'started' ? 'Started' : 'Using';
   spinner.stop(`${verb} ktx daemon: ${daemon.baseUrl}`);
+  restoreCookedStdin();
 
   return {
     baseUrl: daemon.baseUrl,
diff --git a/packages/cli/test/context/scan/local-enrichment.test.ts b/packages/cli/test/context/scan/local-enrichment.test.ts
index 61cdbcb..bce2acb 100644
--- a/packages/cli/test/context/scan/local-enrichment.test.ts
+++ b/packages/cli/test/context/scan/local-enrichment.test.ts
@@ -10,12 +10,15 @@ import {
   loadOnDiskDescriptionUpdates,
   writeLocalScanEnrichmentArtifacts,
 } from '../../../src/context/scan/local-enrichment-artifacts.js';
-import type {
-  KtxScanEnrichmentCompletedStage,
-  KtxScanEnrichmentFailedStage,
-  KtxScanEnrichmentStageLookup,
-  KtxScanEnrichmentStateStore,
+import {
+  computeKtxTableDescriptionHash,
+  type KtxScanEnrichmentCompletedStage,
+  type KtxScanEnrichmentFailedStage,
+  type KtxScanEnrichmentStageLookup,
+  type KtxScanEnrichmentStateStore,
+  type KtxScanLlmIdentity,
 } from '../../../src/context/scan/enrichment-state.js';
+import { tableRefKey } from '../../../src/context/scan/table-ref.js';
 import {
   createDeterministicLocalScanEnrichmentProviders,
   runLocalScanEnrichment,
@@ -29,8 +32,34 @@ import {
   type KtxScanConnector,
   type KtxScanContext,
   type KtxSchemaSnapshot,
+  type KtxSchemaTable,
 } from '../../../src/context/scan/types.js';
 
+/**
+ * Build a description-resume record in the store's shape: the prior descriptions
+ * plus the per-table hash each one is gated on. The hash must be computed with the
+ * same function and the same llmIdentity the run uses, otherwise the table is not
+ * recovered and the run re-enriches it.
+ */
+function resumeRecord(
+  updates: { table: { catalog: string | null; db: string; name: string }; tableDescription: string; columnDescriptions: Record<string, string> }[],
+  tables: readonly KtxSchemaTable[],
+  llmIdentity: KtxScanLlmIdentity,
+) {
+  return {
+    descriptions: updates,
+    tableHashes: Object.fromEntries(
+      updates.map((update) => {
+        const table = tables.find((entry) => entry.name === update.table.name && entry.db === update.table.db);
+        if (!table) {
+          throw new Error(`resumeRecord: no table ${update.table.db}.${update.table.name} in the snapshot`);
+        }
+        return [tableRefKey(update.table), computeKtxTableDescriptionHash({ table, llmIdentity })];
+      }),
+    ),
+  };
+}
+
 function fakeScanEmbedding(options: { dimensions: number; maxBatchSize?: number }): KtxEmbeddingPort {
   return {
     dimensions: options.dimensions,
@@ -1404,13 +1433,19 @@ describe('local scan enrichment', () => {
     const providers = createDeterministicLocalScanEnrichmentProviders();
     const identity = { llmIdentity: { model: 'fake', baseUrlConfigured: false } };
     const resumeStore = {
-      load: vi.fn(async () => [
-        {
-          table: { catalog: null, db: 'public', name: 'customers' },
-          tableDescription: 'Recovered customers description',
-          columnDescriptions: { id: 'Recovered id' },
-        },
-      ]),
+      load: vi.fn(async () =>
+        resumeRecord(
+          [
+            {
+              table: { catalog: null, db: 'public', name: 'customers' },
+              tableDescription: 'Recovered customers description',
+              columnDescriptions: { id: 'Recovered id' },
+            },
+          ],
+          snapshot.tables,
+          identity.llmIdentity,
+        ),
+      ),
       flush: vi.fn(async () => {}),
     };
 
@@ -1485,13 +1520,20 @@ describe('local scan enrichment', () => {
     const generateObject = vi.spyOn(providers.llmRuntime, 'generateObject');
     // Only the analytics.orders description was flushed before the interruption.
     const resumeStore = {
-      load: vi.fn(async () => [
-        {
-          table: { catalog: null, db: 'analytics', name: 'orders' },
-          tableDescription: 'Recovered analytics orders',
-          columnDescriptions: { id: 'Recovered analytics id' },
-        },
-      ]),
+      load: vi.fn(async () =>
+        resumeRecord(
+          [
+            {
+              table: { catalog: null, db: 'analytics', name: 'orders' },
+              tableDescription: 'Recovered analytics orders',
+              columnDescriptions: { id: 'Recovered analytics id' },
+            },
+          ],
+          multiSchemaSnapshot.tables,
+          // runLocalScanEnrichment defaults to this when no llmIdentity is passed.
+          { model: null, baseUrlConfigured: false },
+        ),
+      ),
       flush: vi.fn(async () => {}),
     };
 
```

## License

Apache-2.0, same as upstream. Original work © Kaelio.
