# Pixi official notes

Detailed reference for the pixi-skill. Read when editing manifests by hand, debugging activation/native errors, or migrating an older manifest.

## Contents

- Docs links
- Manifest structure
- Version specifiers
- Environments & features
- Tasks
- PyPI dependencies
- System requirements / CUDA
- Target tables
- Channels & activation
- Conda + PyPI resolution
- Lockfile & install flags
- Global tools
- Common commands
- Authentication
- Troubleshooting
- Pre-2025 → current migration

## Docs links

Canonical: `https://pixi.prefix.dev/latest/`

- Manifest: `.../reference/pixi_manifest/`
- pyproject: `.../python/pyproject_toml/`
- CLI: `.../reference/cli/pixi/`
- Multi-environment: `.../workspace/multi_environment/`
- Multi-platform: `.../workspace/multi_platform_configuration/`
- Lock file: `.../workspace/lock_file/`
- Authentication: `.../deployment/authentication/`
- Global tools: `.../global_tools/introduction/`

## Manifest structure

Required top-level table: `[workspace]` (`[tool.pixi.workspace]` in pyproject).

```toml
[workspace]
name = "my-project"
channels = ["conda-forge"]
platforms = ["linux-64", "osx-arm64", "win-64"]
```

Fields: `name`, `channels`, `platforms` (required); `version`, `authors`, `description`, `license`, `channel-priority` (`"strict"` default, `"disabled"` discouraged), `solve-strategy`, `requires-pixi`, `exclude-newer`, `conda-pypi-map`.

## Version specifiers

- Conda: MatchSpec — `"2.0.*"`, `">=1.2,<=1.4"`, `{ version=">=1", channel="pytorch" }`. `pixi add` auto-pins per `--pinning-strategy` (default `semver`).
- PyPI: PEP 440 — `"~=3.5.0"`, `"==3.1.0"`, `*` = any.

## Environments & features

```toml
[feature.test.dependencies]
pytest = "*"

[environments]
test = ["test"]                                   # shorthand
prod = { features = ["prod"], solve-group = "g" } # share solve with another env
lint = { features = ["lint"], no-default-feature = true }
```

`default` feature is implicit, included unless `no-default-feature = true`. `solve-group` makes envs share solved versions for common deps.

## Tasks

```toml
[tasks]
simple = "echo hi"
build = { cmd = "npm run build", cwd = "frontend", inputs = ["package.json"], outputs = ["dist"] }
downstream = { cmd = "pytest", depends-on = "build" }   # hyphen, not underscore
env-task = { cmd = "python run.py $ARG", env = { ARG = "v" }, args = [{ arg = "ARG", default = "v" }] }
```

Runner: `deno_task_shell` (cross-platform). `depends-on` accepts a string or list; cross-environment works. Prefix name with `_` to hide from `pixi task list`.

## PyPI dependencies

```toml
[pypi-dependencies]
fastapi = "*"
pandas = { version = ">=1", extras = ["dataframe"] }
pkg = { git = "https://github.com/org/pkg.git", rev = "abc123" }
local = { path = "./local", editable = true }     # path relative to workspace root
torch = { version = "*", index = "https://download.pytorch.org/whl/cu118" }
```

Fields: `version`, `extras`, `git`, `rev`, `branch`, `tag`, `subdirectory`, `path`, `editable`, `url`, `index`.

## System requirements / CUDA

Declare inline on `workspace.platforms`:

```toml
[workspace]
platforms = [
  "osx-arm64",
  { platform = "linux-64", cuda = "12.0", glibc = "2.28" },
  { name = "gpu", platform = "linux-64", cuda = { driver = "12.0", arch = "8.6" } },
]
```

Friendly keys: `platform` (required), `name`, `cuda` (`__cuda`), `glibc` (`__glibc`), `macos`/`osx` (`__osx`), `linux` (`__linux`), `windows` (`__win`), `archspec` (`__archspec`). Bind a feature to a rich platform via `feature.<name>.platforms`. CLI: `pixi workspace platform add linux-64 --cuda 12`.

## Target tables

```toml
[target.win-64.dependencies]
python = "3.7"
[target.osx.dependencies]      # 'osx' matches both osx-64 and osx-arm64
python = "3.11"
[target.unix.activation.env]
LD_LIBRARY_PATH = "$CONDA_PREFIX/lib"
```

Selectors: exact subdir (`linux-64`), family (`win`, `osx`, `linux`, `unix`), or rich-platform `name`. Target platform must be a subset of `workspace.platforms`.

## Channels & activation

```toml
[workspace]
channels = ["conda-forge", "pytorch", "nvidia"]   # pytorch CUDA needs these
channel-priority = "strict"

[activation]
scripts = ["env_setup.sh"]                         # .sh/.bash Unix, .bat Windows (called, not sourced)
env = { LD_LIBRARY_PATH = "$CONDA_PREFIX/lib" }    # $VAR Unix, %VAR% Windows
```

Env var priority: `task.env` > `activation.env` > `activation.scripts` > dependency activation scripts > outside env. Common native vars: `LD_LIBRARY_PATH`, `DYLD_FALLBACK_LIBRARY_PATH`, `PKG_CONFIG_PATH`, `CMAKE_PREFIX_PATH`, `CC`, `CXX`, `CUDA_HOME`.

## Conda + PyPI resolution

Pixi resolves Conda first, maps to PyPI names, then resolves PyPI. When both provide a package, Conda wins. Prefer Conda for foundational native packages (`python`, `numpy`, `scipy`, `pytorch`, toolchains, CUDA libs). Prefer PyPI for packages unavailable on Conda. Beware PyPI wheels expecting runtime libs that differ from the env's.

## Lockfile & install flags

`pixi.lock` (format v6), backward-compatible only — don't hand-edit.

| Flag | Behavior |
|---|---|
| (none) | Update lock if manifest changed, then install |
| `--frozen` | Install from lock as-is; do NOT update lock even on mismatch |
| `--locked` | Refuse to run if lock is out of date (CI gate); conflicts with `--frozen` |
| `--no-install` | Modify lock only, no env install |
| `--as-is` (run/shell) | `--no-install --frozen` |

`pixi lock` — solve + write lock, no install; `--check` exits non-zero on drift (CI). `pixi update [pkg]` — re-resolve within manifest constraints. `pixi upgrade [pkg]` — loosen manifest requirements and rewrite manifest+lock.

## Global tools

Manifest-based (stable since v0.33). Manifest at `$PIXI_HOME/manifests/pixi-global.toml` (`PIXI_HOME` defaults to `~/.pixi`); env prefixes under `~/.pixi/global`.

```toml
version = 1
[envs.ipython]
channels = ["conda-forge"]
dependencies = { ipython = "*" }
exposed = { ipython = "ipython", ipython3 = "ipython3" }
```

- `pixi global install <pkg>` — install a **new** global tool (isolated env per package by default; `--environment <name>` groups packages, `--with <dep>` adds non-exposed deps, `--expose name=exe` aliases).
- `pixi global add <pkg> --environment <env>` — add a dependency to an **existing** global env (requires `--environment`).
- `pixi global uninstall <env>` — remove a whole env.
- `pixi global remove <pkg>` — remove a package from an env (opposite of `add`).
- `pixi global update` / `pixi global update <env>` — update global envs.
- `pixi global sync` — rebuild envs from manifest (after manual edits / VCS pull).
- `pixi global list`, `pixi global edit`, `pixi global tree [<env>]`, `pixi global expose add/remove <name>=<exe>`.

## Common commands

- Inspect: `pixi info`, `pixi list` (`--explicit`), `pixi tree`, `pixi task list`, `pixi search <pkg>`.
- Deps: `pixi add <pkg>` (`--pypi`, `--host`, `--build`, `--feature`, `--editable`), `pixi remove <pkg>`.
- Env: `pixi install` (`-e <env>`), `pixi reinstall`, `pixi clean`, `pixi lock` (`--check`), `pixi update [pkg]`, `pixi upgrade [pkg]`.
- Run: `pixi run <task>`, `pixi exec -- <cmd>`, `pixi shell`, `pixi shell-hook` (`--json`).
- Manifest: `pixi workspace channel|platform|environment|feature …`.
- Self: `pixi self-update`.

## Authentication

`pixi auth login <host>` (`--token`, `--conda-token`, `--username`/`--password`, `--s3-*`, `--oauth*`), `pixi auth logout <host>`, `pixi auth status`.

## Troubleshooting

- Works in `pixi run` but not external shell/IDE → `pixi shell-hook --json`; check if vars belong in activation tables.
- Native package loader/ABI errors → confirm runs inside activation; `pixi list --explicit` for dep source; check runtime library lookup needs activation vars.
- Stale env → `pixi install`, `pixi reinstall`, `pixi clean`.
- CI lock drift → `pixi lock --check` or `pixi install --locked`.

## Pre-2025 → current migration

Only relevant when editing an older manifest.

| Old | Current |
|---|---|
| `[project]` table | `[workspace]` (`[tool.pixi.workspace]`) |
| `pixi project channel/environment …` | `pixi workspace channel/environment …` |
| `[system-requirements]` table | inline `workspace.platforms` entries |
| `pixi global upgrade` / `upgrade-all` | `pixi global update` |
| `pixi global remove <env>` (whole env) | `pixi global uninstall <env>` |
| `depends_on` (underscore) | `depends-on` (hyphen) |
| `pixi update` (loosen + bump all) | `pixi update` keeps constraints; `pixi upgrade` loosens |
| `pixi login` / `pixi logout` standalone | `pixi auth login` / `pixi auth logout` |
