---
name: effect-source-setup
description: Clone or update a local copy of the Effect-TS source (the effect-smol monorepo), pinned to the exact `effect` version installed in the current project, for use as an offline documentation/reference checkout. Use when setting up, refreshing, or locating the local Effect source — typically as a prerequisite to querying Effect internals (see the effect-expert agent).
---

# effect-source-setup

Effect v4 is developed in the [`Effect-TS/effect-smol`](https://github.com/Effect-TS/effect-smol)
monorepo. The most reliable documentation for a given version _is its own
source and tests_ — the published docs lag and most third-party knowledge is
for Effect v3. This skill makes a local, version-pinned checkout of that source
available as a reference.

It does **not** modify the target project. Its only writes are the clone into a
cache directory outside any project.

This is **one-time-per-version provisioning**. Run it once for a given effect
version; after that the `effect-expert` agent consults the prebuilt index on
every query and only re-runs this script on a cache miss (a new version).

## What it does

1. Walks up from the project directory to find `node_modules/effect/package.json`.
2. Reads the installed `effect` version (e.g. `4.0.0-beta.78`).
3. Computes the matching release tag `effect@<version>` (effect-smol tags
   per-package, no `v` prefix; sibling packages like `@effect/sql` and
   `@effect/platform-node` release in lockstep, so this one tag covers the
   whole monorepo).
4. Shallow-clones that tag into a version-keyed cache directory if not already
   present (idempotent — re-running is a fast no-op).
5. Builds a **search index** in a version-keyed state directory (outside the git
   checkout, so the checkout stays pristine):
   - `symbols.tsv` — `Symbol<TAB>relative/path.ts:line` for every top-level
     export across all packages' `src` (~13k entries): **where things are
     defined**. Built with pure `find`+`awk`. Meant to be **grepped, not read
     whole** — grepping it pinpoints a definition in one cheap step instead of
     ripgrepping the entire tree.
   - `tests.tsv` — `ImportedSymbol<TAB>test/path.ts:line` extracted from every
     `*.test.ts` file's imports (~2.7k entries): **which tests exercise a
     symbol**. Only effect-project imports are kept (`effect`, `effect/*`,
     `@effect/*` — including `@effect/vitest`); external libs (`vitest`,
     `@testcontainers`, `node:*`) and relative test-helper imports are dropped
     as noise. Grepping a symbol here surfaces runnable demonstrations (the
     `/test/` vs `/src/` path distinguishes example from definition).
   - `titles.tsv` — `Test title<TAB>test/path.ts:line` for every `it`/`test`/
     `describe` case (~8k entries): **which test demonstrates a scenario**.
     Grep behavioral keywords ("transaction", "interrupt", "retry") to land on
     a specific runnable example — useful when the question isn't tied to one
     symbol name.
   - `breadcrumbs.md` — initialized empty; the agent appends learned
     `topic → files` navigation hints over time. These are pointers, never
     cached answers.

   Both are version-keyed, so a version bump starts a fresh index and stale line
   numbers are never trusted.

## How to run it

```bash
bash ~/.claude/skills/effect-source-setup/setup.sh [--reindex] [PROJECT_DIR]
bash ~/.claude/skills/effect-source-setup/setup.sh --breadcrumb "<line>" [PROJECT_DIR]
```

- `PROJECT_DIR` defaults to `$PWD`. Pass the project root if invoking from
  elsewhere.
- `--reindex` rebuilds `symbols.tsv`, `tests.tsv`, and `titles.tsv` even if they
  already exist (use after manually editing the checkout, or to refresh).
- `--breadcrumb "<line>"` appends one deduped navigation hint to this version's
  `breadcrumbs.md` and exits without cloning/indexing. (Routing breadcrumb
  writes through this script keeps them on a single allowlistable command — see
  Permissions below.)
- Override the cache location with `EFFECT_SOURCE_CACHE` (default
  `~/.cache/effect-smol`).

The script prints machine-parseable result lines to **stdout** (progress goes
to stderr):

```
EFFECT_VERSION=4.0.0-beta.78
EFFECT_TAG=effect@4.0.0-beta.78
EFFECT_SOURCE=/Users/<you>/.cache/effect-smol/effect@4.0.0-beta.78
EFFECT_STATE=/Users/<you>/.cache/effect-smol/.state/effect@4.0.0-beta.78
EFFECT_SYMBOLS=/Users/<you>/.cache/effect-smol/.state/effect@4.0.0-beta.78/symbols.tsv
EFFECT_TESTS=/Users/<you>/.cache/effect-smol/.state/effect@4.0.0-beta.78/tests.tsv
EFFECT_TITLES=/Users/<you>/.cache/effect-smol/.state/effect@4.0.0-beta.78/titles.tsv
EFFECT_BREADCRUMBS=/Users/<you>/.cache/effect-smol/.state/effect@4.0.0-beta.78/breadcrumbs.md
```

Use `EFFECT_SOURCE` as the root for reading/searching the source, `EFFECT_SYMBOLS`
to locate definitions fast, `EFFECT_TESTS` to find tests that exercise a given
symbol, and `EFFECT_TITLES` to find tests by scenario/behavior (usage examples).

## Layout of the checkout

- `LLMS.md` — canonical AI-oriented docs and conventions (read this first).
- `MIGRATION.md` — v3 → v4 changes (important: most external Effect knowledge
  is v3).
- `ai-docs/`, `cookbooks/` — additional guides and recipes.
- `packages/effect/src/*.ts` — core source; one file per module
  (`Effect.ts`, `Stream.ts`, `Layer.ts`, `Config.ts`, `Context.ts`, …).
- `packages/effect/test/*.test.ts` — runnable usage examples (often the
  clearest reference for _how_ an API is used).
- `packages/effect/typetest/` — type-level tests.
- Other packages under `packages/`: `sql`, `platform-node`,
  `platform-node-shared`, `ai`, `atom`, `opentelemetry`, etc.

## Permissions

The setup script patches `~/.claude/settings.json` automatically on every run
(idempotent). After running the skill once, the `effect-expert` agent operates
fully prompt-free:

- **`additionalDirectories`** — grants the native `Read`/`Grep`/`Glob` tools
  prompt-free access to the entire cache dir. The agent uses these tools (not
  Bash) for all index lookups and source reads.
- **`allow` rules** — two entries covering the tilde and absolute forms of this
  script, so the one Bash call the agent makes (to provision + record breadcrumbs)
  never prompts.

If `python3` is unavailable the script prints the entries to add manually, but on
any modern macOS or Linux dev machine it runs silently.

## Troubleshooting

- **"could not find node_modules/effect/package.json"** — install deps in the
  project first (`effect` must be in `node_modules`).
- **"tag … was not found"** — the installed version is a canary/pre-release
  that upstream hasn't tagged. Check the nearest tagged version with
  `git ls-remote --tags https://github.com/Effect-TS/effect-smol.git 'effect@*'`.
- **Stale checkout** — delete the version-keyed dir under the cache root and
  re-run, or just bump the project's effect version (a new version produces a
  new cache dir).
- **Rebuild / reset the index** — `setup.sh --reindex` rebuilds `symbols.tsv`,
  `tests.tsv`, and `titles.tsv`. To clear learned breadcrumbs, delete
  `~/.cache/effect-smol/.state/effect@<version>/breadcrumbs.md` (it is recreated
  empty on the next run).
