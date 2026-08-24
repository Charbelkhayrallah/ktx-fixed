# ktx-fixed

An unofficial patched build of [`@kaelio/ktx`](https://github.com/Kaelio/ktx)
**v0.16.0**, published as **`@charbelkh/ktx` 0.16.3**. Four fixes, all in this
repo as a single patch you can apply to a pristine upstream checkout.

Not affiliated with or endorsed by Kaelio. Apache-2.0, same as upstream.

| # | Symptom | Fix |
|---|---|---|
| 1 | `ktx ingest` regenerates every AI description on every run | Drop wall-clock and row-count fields from the resume hashes |
| 2 | Adding one table discards all the other descriptions | Track a resume hash per table instead of per batch |
| 3 | Every ktx MCP tool fails in Claude Code before it runs | Make the MCP SDK advertise JSON Schema 2020-12 |
| 4 | Ctrl-C cannot stop an ingest | Stop leaking the terminal into raw mode |

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

Check it worked — should print `@charbelkh/ktx 0.16.3`:

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

**One exception.** If a run hit an LLM rate limit and left some descriptions
empty, a plain re-run will not retry them. Fill them in with:

```bash
ktx ingest <connection-id> --stages descriptions
```

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

Only needed to verify this build, or when upstream ships a new version. Requires
Node >= 22 and pnpm 11.4.0 (upstream's `packageManager`).

```bash
git clone https://github.com/Kaelio/ktx.git
cd ktx
git checkout v0.16.0
git apply /path/to/ktx-fixes.diff
pnpm install
cd packages/cli
pnpm run build
npm pack
```

That produces `charbelkh-ktx-0.16.3.tgz`. Rename it to `ktx-fixed.tgz`, commit it
here, and the install command at the top keeps working unchanged.

The patch is self-contained: it carries the new `patch-mcp-sdk-dialect.cjs` and
the `package.json` edits it needs — `name`, `version`, `description`, the `files`
entry for the script, the `postinstall` hook, and dropping
`publishConfig.provenance`, which only works from upstream's own CI. `pnpm
install` will run that `postinstall` in your dev tree too; harmless, and
idempotent.

Bundling the Python runtime assets needs no special step: `pnpm run build`
already runs `scripts/copy-runtime-assets.mjs`, and `files: ["dist", "assets"]`
ships the result.

To port the patch to a newer upstream release, apply it to that tag instead. All
four changes are small and localised; if a hunk stops applying, the sections
above say what each one has to accomplish.

### What has been verified

Against a pristine `git clone` of upstream at tag `v0.16.0` (commit `a6dd8cf`):

- `git apply --check ktx-fixes.diff` applies cleanly, all six files.
- After applying, `packages/cli/package.json` is **identical in every field** to
  the `package.json` inside `ktx-fixed.tgz`, and
  `packages/cli/scripts/patch-mcp-sdk-dialect.cjs` is byte-identical to the
  shipped copy. (Line endings and a byte-order mark may differ — the shipped
  copies were written on Windows. No content differs.)
- The shipped `dist/` carries all four fixes: `stableSnapshotHashInput` and
  `computeKtxTableDescriptionHash` in `dist/context/scan/enrichment-state.js`,
  `tableHashes` in `dist/context/scan/local-enrichment-artifacts.js`,
  `restoreCookedStdin` and `createStaticCliSpinner` in
  `dist/managed-local-embeddings.js`, and `scripts/patch-mcp-sdk-dialect.cjs` in
  the package.

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

## License

Apache-2.0, same as upstream. Original work © Kaelio.
