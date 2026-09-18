# Pixi operations reference

Read for lockfile, global tools, exec, auth, or troubleshooting questions.

## Lockfile & install flags

`pixi.lock` is format **v7** (0.71+; backward-compatible only — don't hand-edit).

| Flag | Behavior |
|---|---|
| (none) | Update lock if manifest changed, then install |
| `--frozen` | Install from lock as-is; never update lock |
| `--locked` | Refuse to run if lock is stale (CI gate); conflicts with `--frozen` |
| `--no-install` | Modify lock only |
| `--as-is` (run/shell) | `--no-install --frozen` |

`pixi lock --check` exits non-zero on lock drift (CI). `pixi update [pkg]` re-resolves within constraints; `pixi upgrade [pkg]` loosens specs and rewrites manifest+lock. Both apply to standalone scripts with adjacent lock files.

## Global tools

Manifest: `$PIXI_HOME/manifests/pixi-global.toml` (`PIXI_HOME` defaults to `~/.pixi`); env prefixes under `~/.pixi/global`.

```toml
version = 1
[envs.ipython]
channels = ["conda-forge"]
dependencies = { ipython = "*" }
exposed = { ipython = "ipython", ipython3 = "ipython3" }
```

- `pixi global install <pkg>` — new env per package by default; `--environment <name>` groups, `--with <dep>` adds non-exposed deps, `--expose name=exe` aliases, `--path`/`--build-backend` build from source. Specs are conda MatchSpecs only — **no `--pypi` yet**.
- `pixi global add --environment E <pkg>` / `pixi global remove --environment E <pkg>` — add/remove deps in an existing env.
- `pixi global uninstall E` — delete whole env. `pixi global update [env]` — update envs.
- `pixi global sync` — rebuild envs from the manifest (after manual edits/VCS pull).
- `pixi global list` / `edit` / `tree [env]` / `expose add|remove name=exe`.

## pixi exec

`pixi exec [OPTIONS] [COMMAND]...` runs a command in a throwaway environment. `pixi exec -- <cmd>` guesses the package from the command; give explicit specs via `-s/--spec` or extra deps via `--with` (conda MatchSpecs only). `-c` selects channels. Purge temp envs: `pixi clean cache --exec`.

## Authentication

`pixi auth login <host>` (`--token`, `--conda-token`, `--username`/`--password`, `--s3-*`, `--oauth*`), `pixi auth logout <host>`, `pixi auth status`.

## Command groups

- Inspect: `pixi info`, `pixi list [--explicit]`, `pixi tree`, `pixi task list`, `pixi search` (wildcard default).
- Deps: `pixi add` (`--pypi/--host/--build/--feature/--editable/--path`), `pixi remove`.
- Env: `pixi install [-e] [--script]`, `pixi reinstall`, `pixi clean`, `pixi lock --check`.
- Run: `pixi run <task|script>`, `pixi exec`, `pixi shell`, `pixi shell-hook --json`.
- Workspace/self: `pixi workspace channel|platform|environment|feature|activation|preview|dependencies`, `pixi self-update`.

## Troubleshooting

- Works in `pixi run` but not in external shell/IDE → `pixi shell-hook --json`; check if vars belong in activation tables.
- Native loader/ABI errors → confirm it runs inside activation; `pixi list --explicit` to see dep source.
- Stale env → `pixi install`, `pixi reinstall`, `pixi clean`.
- CI lock drift → `pixi lock --check` or `pixi install --locked`.
- `requires-pixi` mismatch → `pixi self-update --version <ver>`.
