---
name: pixi
description: "Manage Pixi workspaces, manifests, environments, tasks, lockfiles, and global tool environments. Use when working with pixi.toml, pyproject.toml configured for Pixi, pixi.lock, or Pixi global manifests; when adding or removing Conda/PyPI dependencies; when defining tasks, activation, features, or target-specific settings; or when installing and exposing global CLI tools."
---

# Pixi

Use this skill when the task involves Pixi manifests, environments, tasks, lockfiles, Conda/PyPI dependencies, activation behavior, or Pixi global tools.

Read `references/official-notes.md` first when the task depends on Pixi behavior rather than local project conventions.

## Current state (2026, pixi v0.7x)

The top-level manifest table is **`[workspace]`** (in `pyproject.toml`: `[tool.pixi.workspace]`). `name`, `channels`, `platforms` are required. Default channel is `conda-forge` only — re-add `pytorch`/`nvidia` explicitly when needed.

```toml
[workspace]
name = "my-project"
channels = ["conda-forge"]
platforms = ["linux-64", "osx-arm64", "win-64"]
```

## Working style

- Prefer official Pixi concepts and command flows over ad hoc shell habits.
- Preserve the project's existing manifest style unless migration is part of the task.
- Prefer declarative manifest changes and Pixi commands over editing `.pixi/` by hand.
- Use `pixi run`, `pixi exec`, or `pixi shell` when checking behavior — activation is part of the runtime contract.

## Core decisions

- **Manifest choice**: `pyproject.toml` for Python-first projects; `pixi.toml` for mixed-language or non-Python.
- **Dependency source**: Conda first, PyPI only when needed or when the project already uses PyPI wheels.
- **Tasks**: repeatable commands in `[tasks]`; `pixi exec -- <cmd>` for one-offs.
- **Activation**: `[activation.env]` for stable vars; activation scripts only for dynamic values.
- **Multiple environments**: features + environments + target tables only when the project truly needs separate stacks (`cpu`, `cuda`, `test`, `docs`).

## Standard workflow

1. Inspect: `pixi info`, `pixi list`, `pixi task list`.
2. Identify manifest: `pixi.toml` or `pyproject.toml`.
3. Make the smallest declarative change: `pixi add <pkg>`, `pixi add --pypi <pkg>`, `pixi remove <pkg>`, or direct manifest edit for restructures.
4. Validate: `pixi install`, `pixi run <task>`.
5. If behavior differs across shells/tools: `pixi shell-hook --json`.

## Manifest essentials

### Environments & features

```toml
[feature.test.dependencies]
pytest = "*"

[environments]
test = ["test"]                                   # shorthand
prod = { features = ["prod"], solve-group = "g" } # share solve with another env
lint = { features = ["lint"], no-default-feature = true }
```

The `default` feature is implicit and included unless `no-default-feature = true`. `solve-group` makes envs share solved versions for common deps.

### Tasks

```toml
[tasks]
simple = "echo hi"
build = { cmd = "npm run build", cwd = "frontend", inputs = ["package.json"], outputs = ["dist"] }
downstream = { cmd = "pytest", depends-on = "build" }   # hyphen, not underscore
env-task = { cmd = "python run.py $ARG", env = { ARG = "v" }, args = [{ arg = "ARG", default = "v" }] }
hidden = "..."      # prefix name with _ to hide from pixi task list
```

Runner is `deno_task_shell` (cross-platform). `depends-on` accepts a string or list. Cross-environment depends-on works.

### Version specifiers

Conda deps use MatchSpec (`"2.0.*"`, `">=1.2,<=1.4"`, `{ version=">=1", channel="pytorch" }`). PyPI deps use PEP 440 (`"~=3.5.0"`, `"==3.1.0"`, `*` = any). `pixi add` auto-pins per `--pinning-strategy` (default `semver`).

### PyPI deps — extras, git, path, index

```toml
[pypi-dependencies]
fastapi = "*"
pandas = { version = ">=1", extras = ["dataframe"] }
pkg = { git = "https://github.com/org/pkg.git", rev = "abc123" }
local = { path = "./local", editable = true }     # path relative to workspace root
torch = { version = "*", index = "https://download.pytorch.org/whl/cu118" }
```

### System requirements / CUDA (inline, not `[system-requirements]`)

```toml
[workspace]
platforms = [
  "osx-arm64",
  { platform = "linux-64", cuda = "12.0", glibc = "2.28" },           # friendly keys
  { name = "gpu", platform = "linux-64", cuda = { driver = "12.0", arch = "8.6" } },
]
```

Friendly keys: `platform` (required), `name`, `cuda` (`__cuda`), `glibc` (`__glibc`), `macos`/`osx` (`__osx`), `linux` (`__linux`), `windows` (`__win`), `archspec` (`__archspec`). Bind a feature to a rich platform via `feature.<name>.platforms`. CLI: `pixi workspace platform add linux-64 --cuda 12`.

### Target tables (platform overrides)

```toml
[target.win-64.dependencies]
python = "3.7"          # override for win-64
[target.osx.dependencies]   # 'osx' matches both osx-64 and osx-arm64
python = "3.11"
[target.unix.activation.env]
LD_LIBRARY_PATH = "$CONDA_PREFIX/lib"
```

Selectors: exact subdir (`linux-64`), family (`win`, `osx`, `linux`, `unix`), or rich-platform `name`. Target platform must be a subset of `workspace.platforms`.

### Channels & activation

```toml
[workspace]
channels = ["conda-forge", "pytorch", "nvidia"]   # pytorch CUDA needs these
channel-priority = "strict"                        # default; "disabled" discouraged

[activation]
scripts = ["env_setup.sh"]                         # .sh/.bash on Unix, .bat on Windows (called, not sourced)
env = { LD_LIBRARY_PATH = "$CONDA_PREFIX/lib" }    # $VAR on Unix, %VAR% on Windows
```

## Activation and runtime

Pixi activation is more than `PATH` — dependency activation scripts also run, so behavior can differ between `pixi run`, `pixi shell`, and an IDE that only points at the env binary. Use `[activation.env]` / `[target.<os>.activation.env]` for stable vars (`LD_LIBRARY_PATH`, `DYLD_FALLBACK_LIBRARY_PATH`, `PKG_CONFIG_PATH`, `CMAKE_PREFIX_PATH`, `CC`, `CXX`, `CUDA_HOME`). Priority: `task.env` > `activation.env` > `activation.scripts` > dependency activation scripts > outside env.

## Conda + PyPI guidance

Pixi resolves Conda first, maps to PyPI names, then resolves PyPI. When both provide a package, **Conda wins**. Prefer Conda for foundational native packages (`python`, `numpy`, `scipy`, `pytorch`, compiler toolchains, CUDA libs). Prefer PyPI for packages unavailable on Conda or when the project requires a PyPI distribution. Beware PyPI wheels expecting runtime libs that differ from the env's.

## Lockfile & install flags

`pixi.lock` (format v6) is regenerated from manifest+lock. Don't hand-edit.

| Flag | Behavior |
|---|---|
| (none) | Update lock if manifest changed, then install |
| `--frozen` | Install from lock as-is; do NOT update lock even on mismatch |
| `--locked` | Refuse to run if lock is out of date (CI gate); conflicts with `--frozen` |
| `--no-install` | Modify lock only, no env install |
| `--as-is` (run/shell) | `--no-install --frozen` |

- `pixi lock` — solve + write lock, no install; `--check` exits non-zero on drift (CI).
- `pixi update [pkg]...` — re-resolve within manifest constraints (no manifest edits).
- `pixi upgrade [pkg]...` — **loosen** manifest requirements and update manifest+lock.

## Global tools

`pixi global` is manifest-based (stable since v0.33). Manifest at `$PIXI_HOME/manifests/pixi-global.toml` (`PIXI_HOME` defaults to `~/.pixi`); env prefixes under `~/.pixi/global`. Two distinct verbs:

- `pixi global install <pkg>` — install a **new** global tool; by default creates an isolated env per package (pipx-style) and exposes its executables onto `PATH`. Use `--environment <name>` to group several packages into one shared env, `--with <dep>` to add non-exposed deps, `--expose name=exe` to alias.
- `pixi global add <pkg> --environment <env>` — add a dependency to an **existing** global env (requires `--environment`; use `--expose` to expose its executables).

```toml
version = 1
[envs.ipython]
channels = ["conda-forge"]
dependencies = { ipython = "*" }
exposed = { ipython = "ipython", ipython3 = "ipython3" }
```

```bash
pixi global install ruff                              # new tool, isolated env
pixi global install "python=3.12" --expose py3=python # alias + pin
pixi global install --environment ds --expose jupyter --expose ipython jupyter numpy pandas
pixi global add scipy --environment ds                # add dep to existing env 'ds'
pixi global list                                      # list envs + exposed cmds
pixi global uninstall <env>                           # remove a whole env
pixi global remove <pkg>                              # remove a package from an env (opposite of add)
pixi global update                                     # update all global envs
pixi global update <env>                               # update one env
pixi global sync                                      # rebuild envs from manifest (after manual edits / VCS pull)
pixi global edit                                      # open manifest in $EDITOR
pixi global expose add <name>=<exe>                   # expose an executable
pixi global tree [<env>]
```

## Common commands

Workspace: `pixi info`, `pixi list` (`--explicit`), `pixi tree`, `pixi task list`, `pixi add <pkg>` (`--pypi`, `--host`, `--build`, `--feature`, `--editable`), `pixi remove <pkg>`, `pixi install` (`-e <env>`), `pixi run <task>`, `pixi exec -- <cmd>`, `pixi shell`, `pixi shell-hook` (`--json`), `pixi lock` (`--check`), `pixi update [pkg]`, `pixi upgrade [pkg]`, `pixi reinstall`, `pixi clean`, `pixi search <pkg>`, `pixi workspace channel|platform|environment|feature …`, `pixi self-update`.

Auth: `pixi auth login <host>` (`--token`, `--conda-token`, `--username`/`--password`, `--s3-*`, `--oauth*`), `pixi auth logout`, `pixi auth status`.

## Troubleshooting entry points

- Command works in `pixi run` but not external shell/IDE → `pixi shell-hook --json`; check if vars belong in activation tables.
- Native package fails with loader/ABI errors → confirm it runs inside activation; `pixi list --explicit` for dep source; check runtime library lookup needs activation vars.
- Stale env → `pixi install`, `pixi reinstall`, `pixi clean`.
- CI lock drift → `pixi lock --check` or `pixi install --locked`.

## References

- Read `references/official-notes.md` for the official Pixi concepts, links, and behavior notes behind this skill.
