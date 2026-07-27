# ktx-fixed

`ktx` **0.16.0** with two fixes so `ktx ingest` **reuses the AI descriptions it
already generated** instead of regenerating them on every run.

Unofficial patched build of [Kaelio/ktx](https://github.com/Kaelio/ktx).
Not affiliated with or endorsed by Kaelio.

## Install

```bash
npm uninstall -g @kaelio/ktx
npm i -g https://raw.githubusercontent.com/Charbelkhayrallah/ktx-fixed/main/ktx-fixed.tgz
```

Check it worked — should print `@charbelkh/ktx 0.16.1`:

```bash
ktx --version
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
empty, a plain re-run won't retry them. Fill them in with:

```bash
ktx ingest <connection-id> --stages descriptions
```

That keeps the good descriptions and regenerates only the empty ones.

## What was changed

Three files in `packages/cli/src/context/scan/` — full patch in
[`ktx-fixes.diff`](ktx-fixes.diff).

**1. `extractedAt` removed from the stage hashes** (`enrichment-state.ts`).
The resume hashes were computed over the whole schema snapshot, which carries
`extractedAt` — a wall-clock timestamp set fresh on every scan. So an unchanged
schema produced a new hash every run and missed its cache, forcing a full
regeneration.

**2. Per-table resume** (`local-enrichment-artifacts.ts`, `local-enrichment.ts`).
The resume record was all-or-nothing on a whole-batch hash, so touching one
table discarded every description. It now tracks a hash per table and recovers
each one independently.

Together these restore behaviour ktx's own
[CLI reference](https://docs.kaelio.com/ktx/docs/cli-reference/ktx-ingest)
already documents.

## This is temporary

Both bugs are reported upstream: [#347](https://github.com/Kaelio/ktx/issues/347)
and [#348](https://github.com/Kaelio/ktx/issues/348). **Once they ship, drop this
build** and go back to:

```bash
npm i -g @kaelio/ktx
```

## Rebuilding (only if upstream releases a new version)

```bash
git clone https://github.com/Kaelio/ktx.git
cd ktx
git apply /path/to/ktx-fixes.diff
pnpm install && cd packages/cli && pnpm run build && npm pack
```

Rename the resulting `.tgz` to `ktx-fixed.tgz`, commit it here, and the install
command above keeps working unchanged.

## License

Apache-2.0, same as upstream. Original work © Kaelio.
