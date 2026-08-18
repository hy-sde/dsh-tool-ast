# dsh-tool-ast — structural search & rewrite for DeepSeek Harness

A standalone package, installable as **one plugin** (two tools) for the DeepSeek
Harness CLI:

| package | tools | installed by users? |
|---|---|---|
| `@hy-sde-org/dsh-tool-ast` | `ast_grep` (structural code search) + `ast_edit` (structural rewrite, preview-first) | yes |

`ast_grep` and `ast_edit` are a full parity port of oh-my-pi's coding-agent
ast tools onto the harness tool contract (`ctx.tools`, `ctx.fs`,
`ctx.subprocess`, `ctx.systemPrompt`). The engine is the **packaged
`@ast-grep/cli` native binary** — it ships inside the npm dependency, so the
plugin works on stock DeepSeek Harness deployments with **zero upstream
changes** and no system `ast-grep` install.

## The two tools

- **`ast_grep`** — syntax-aware structural search. Find every function, call,
  class, or declaration matching a tree pattern instead of a text substring:
  `console.log($MSG)` finds every console.log call, `fn($X)` finds every
  function `fn` with one argument. Patterns bind metavariables (`$NAME`,
  `$_`, `$$$NAME`) so a search can be turned directly into a rewrite.
- **`ast_edit`** — structural rewrite. Replace every node matching a pattern
  with a template that references the captured metavariables
  (`console.log($MSG)` → `log.debug($MSG)`). **It always PREVIEWS first**
  (`apply` defaults to `false`); pass `apply: true` to write the files.
  Rewrites are 1:1 structural substitutions — never text search-and-replace.

Both share one engine, one error vocabulary (`AST_*`), one set of caps, and
one filesystem seam — which is why they ship in one package rather than two.

## Relationship to the `edit` tool

For coding agents, `edit` (targeted, literal, line-anchored text changes) and
`ast_edit` (structural AST changes) are **complementary, not alternatives**:

| | `edit` / `ast_edit` in omp | `edit` / `ast_edit` in this package |
|---|---|---|
| pairing | auto-added together when `edit` is requested | both tools ship in this one package |
| apply model | `edit` applies directly; `ast_edit` previews + staged resolve | `ast_edit` previews by default; `apply: true` writes |
| guidance | "For one-off text edits, prefer the Edit tool" | idiomatic: `ast_edit` for codemods → `edit` for follow-ups |

Both mutation paths write through the same `ctx.fs` seam (observation
watermark, version guard, sandbox policy), so they compose safely in one
session. The harness companion-preset pattern (`code-edit`) mounts a rich
`edit` beside `ast_edit`; this package's bundle (below) provides
`read`/`write` beside `ast_grep`/`ast_edit`, and can be pointed at the rich
`@hy-sde-org/dsh-tool-edit` if you want the full editor.

## Install

```bash
pnpm install --global @deepseek-ai/dsh
```

### Direct from npm (published)

```bash
# both tools arrive in one command
dsh plugin --profile web add @hy-sde-org/dsh-tool-ast
```

`dsh plugin add` reconciles the profile's bundle list from the installed
`dsh.bundle.patch` export, so after installation the `hy-sde-ast-fs`
composition below is immediately active in the named profile.

You can also just depend on the package from your own tooling:

```bash
npm install @hy-sde-org/dsh-tool-ast   # or pnpm add / yarn add
```

### From the git checkout (pre-publish / development)

```bash
git clone git@github.com:hy-sde/dsh-tool-ast.git
cd dsh-tool-ast
pnpm install
pnpm run build

AST_TGZ="$(cd packages/tool-ast && pnpm pack --silent --pack-destination /tmp)"
dsh plugin --profile web add "$AST_TGZ"
```

### Verify

```bash
dsh web --dump-config   # look for the hy-sde-ast-fs group rows
```

### Uninstall

```bash
dsh plugin --profile web remove @hy-sde-org/dsh-tool-ast
```

> **Already shipped?** If a future DeepSeek Harness release adopts structural
> search/rewrite itself, skip installation — adding this bundle on top would
> duplicate the loader row and fail at boot.

> **Prompt-section collisions on raw profiles:** the shipped `web` profile
> (the GUI) disables the base host-plane tool rows, so the plugin's
> `read`/`write` register cleanly there. On a raw `dsh-base` profile the host
> still mounts its own `tool-fs`, whose `tool:read` sections collide with the
> plugin's — boot fails with "prompt section … is already registered". Apply
> the same disables a web deployment has, or mount the plugin rows inside a
> preset realm (the harness preset pattern).

## What the bundle does

The plugin's `cordis.patch.yml` is **self-contained**: `ctx.fs` is not
mounted host-wide in Harness (presets own local filesystem discovery), so the
bundle brings its own isolated fs realm with fresh row ids (`hy-sde-*`) that
cannot collide with shipped rows; the `subprocess` seam stays on the host
plane, exactly as in the shipped presets:

- `hy-sde-ast-fs` — a `cordis:group` isolated on `fs`
  - `hy-sde-ast-fs-local` — `@deepseek-ai/dsh-fs-local` (cwd: `DSH_CWD` or the
    harness process cwd; override by patching this row)
  - `hy-sde-ast-tool-fs` — `@deepseek-ai/dsh-tool-fs` with `enableEdit: false`
    (read/write only — the `edit` name belongs to dsh-tool-edit or the host).
    **On harness releases whose `tool-fs` predates the `enableEdit` option**
    (the `dsh-v0.1.0-rc.7` tag and the published
    `@deepseek-ai/dsh-tool-fs@0.1.0-rc.7`) the key is silently ignored and the
    bundle also registers `edit` — harmless when installed standalone, fatal
    only if a second `edit`-owner (dsh-tool-edit) lands in the same realm.
  - `hy-sde-ast-tool-ast` — `@hy-sde-org/dsh-tool-ast`

Configure per deployment by patching the rows by id:

```yaml
- id: hy-sde-ast-fs-local
  config:
    cwd: /path/to/workspace
- id: hy-sde-ast-tool-ast
  config:
    astGrepMaxMatches: 200
    astEditMaxHunkBytes: 8000
    timeoutMs: 45000
```

### Pairing with the rich `edit` plugin

Install `@hy-sde-org/dsh-tool-edit` and this package's tools **inside one
filesystem realm** rather than both bundles standalone (each bundle owns a
`tool-fs` and the second would collide). Follow the harness `code-edit`
preset shape: mount both tool rows under a single fs-isolated group.

> **Requires a modern harness `tool-fs`.** This pairing depends on
> `@deepseek-ai/dsh-tool-fs` supporting `enableEdit: false` so the plain
> `edit`-owner steps aside for the rich editor — a feature that landed after
> the `dsh-v0.1.0-rc.7` tag (current dev tree / next release). On the released
> `rc.7` harness there is no `code-edit` preset and a `tool-fs` row always
> registers `edit`, so the pairing cannot be assembled there. On such a
> release, install **this bundle standalone** and keep the shipped `edit` — 
> `ast_grep`/`ast_edit` complement the stock editor with no conflicts
> (verified). The `dsh-tool-edit` bundle, by contrast, fails to boot on `rc.7`
> because its own `tool-fs` + `tool-edit` rows both claim `edit`.

## Engine behavior

- **Spawn.** The engine is a subpath of the `@ast-grep/cli` package, resolved
  by `createRequire` from the plugin — the platform binary arrives with
  `pnpm install`, identical to how ripgrep tooling ships.
- **No shell layer.** Every model input (pattern, rewrite, lang, strictness,
  globs, paths) is its own argv element — a hostile pattern stays inert.
- **Strict parse.** `--json=stream` output is parsed line-by-line; a
  malformed line fails the run (`AST_FAILED`) rather than being dropped.
- **Exit semantics.** exit 0 = matches; exit 1 + clean stderr = zero matches;
  exit 1 + stderr = `AST_FIND_ERROR` (bad path); exit 2 = `AST_USAGE_ERROR`;
  else `AST_FAILED`. Cooperative timeout / cancellation → `AST_ABORTED`.
- **Apply is preview-first and version-guarded.** `ast_edit` defaults to
  preview; `apply: true` writes only through the fs edit-intent waterfall
  with the observation/version guard, never a raw engine write.
- **Caps.** All result caps are configurable (see `Config`). The defaults:
  `astGrepMaxMatches` 100, `astGrepMaxNodeBytes` 2000, `astEditMaxHunkBytes`
  4000, `astEditMaxFiles` 200, `searchMetaMaxBytes` 65536, `rawOutputMaxBytes`
  8 MiB, `graceMs` 3000, `stderrMaxBytes` 64 KiB, `timeoutMs` 30000.

## Development

```bash
pnpm install
pnpm -r check      # strict typecheck (tsc --noEmit)
pnpm -r test       # engine unit tests + real-engine integration suite
pnpm -r build      # tsc -> dist
bash scripts/release-public.sh --check      # pre-publish validation
bash scripts/release-public.sh --publish    # publish to npm
```

The test suite runs the REAL packaged ast-grep against real temp files,
exercising `ast_grep`, `ast_edit` preview and apply (observation/version
guard), sandbox denial mapping, and abort classification — no mocks of the
engine.

## Layout

```
packages/tool-ast/   @hy-sde-org/dsh-tool-ast — the plugin (both tools)
  cordis.patch.yml   the installable harness bundle
  src/core.ts        the engine: binary resolution, spawn, argv, JSON parse,
                     error vocabulary
  src/search.ts      the ast_grep tool body
  src/edit.ts        the ast_edit tool body (preview / apply)
```
