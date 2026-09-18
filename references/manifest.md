# Pixi manifest reference

Read when hand-editing or migrating pixi.toml / pyproject.toml.

## Contents

- Workspace
- Version specifiers
- Environments & features
- Tasks
- PyPI dependencies
- System requirements / CUDA
- Target tables
- Channels & activation
- Conda + PyPI resolution
- Pre-2025 → current migration

Canonical docs root: `https://pixi.prefix.dev/latest/` (manifest: `/reference/pixi_manifest/`, CLI: `/reference/cli/pixi/`).

## Workspace

Top-level table `[workspace]` (`[tool.pixi.workspace]` in pyproject). Required: `name`, `channels`, `platforms`. Optional: `version`, `authors`, `description`, `license`, `channel-priority` (strict default; flexible options since 0.76), `solve-strategy`, `requires-pixi`, `exclude-newer`, `conda-pypi-map`.

```toml
[workspace]
name = "my-project"
channels = ["conda-forge"]
platforms = ["linux-64", "osx-arm64", "win-64"]
```

## Version specifiers

- Conda: MatchSpec — `"2.0.*"`, `">=1.2,<=1.4"`, `{ version=">=1", channel="pytorch" }`. `pixi add` auto-pins per `--pinning-strategy` (default semver). Extras/flags/`when` and conditional deps via `if(...)` are supported on conda deps.
- PyPI: PEP 440 — `"~=3.5.0"`, `"==3.1.0"`, `"*"`.

## Environments & features

```toml
[feature.test.dependencies]
pytest = "*"

[environments]
test = ["test"]                                   # shorthand
prod = { features = ["prod"], solve-group = "g" } # envs sharing a solve
lint = { features = ["lint"], no-default-feature = true }

[environments.gpu.dependencies]   # inline envs (0.74+)
torch = "*"
```

`default` feature is implicit unless `no-default-feature = true`. `solve-group` shares solved versions across envs. `{ workspace = true }` reuses a `[workspace.dependencies]` entry.

## Tasks

```toml
[tasks]
simple = "echo hi"
build = { cmd = "npm run build", cwd = "frontend", inputs = ["package.json"], outputs = ["dist"] }
downstream = { cmd = "pytest", depends-on = "build" }   # hyphen, not underscore; string or list
env-task = { cmd = "python run.py $ARG", env = { ARG = "v" }, args = [{ arg = "ARG", default = "v" }] }
```

Runner is deno_task_shell (cross-platform). Prefix a name with `_` to hide it from `pixi task list`.

## PyPI dependencies

```toml
[pypi-dependencies]
fastapi = "*"
pandas = { version = ">=1", extras = ["dataframe"] }
pkg = { git = "https://github.com/org/pkg.git", rev = "abc123", subdirectory = "pkg" }  # --subdirectory since 0.74
local = { path = "./local", editable = true }     # path relative to workspace root
torch = { version = "*", index = "https://download.pytorch.org/whl/cu118" }
```

Path deps also via `pixi add --path <path>` (conda/PyPI/ROS).

## System requirements / CUDA

Inline rich entries on `workspace.platforms` (0.71+; replaces the deprecated `[system-requirements]` table):

```toml
[workspace]
platforms = [
  "osx-arm64",
  { platform = "linux-64", cuda = "12.0", glibc = "2.28" },
  { name = "gpu", platform = "linux-64", cuda = { driver = "12.0", arch = "8.6" } },
]
```

Keys: `platform` (required), `name`, `cuda`, `glibc`, `macos`/`osx`, `linux`, `windows`, `archspec`. Detection can be overridden with `CONDA_OVERRIDE_*` env vars. CLI: `pixi workspace platform add linux-64 --cuda 12` (`--auto-detected` also available). Bind features to rich platforms via `feature.<name>.platforms`.

## Target tables

```toml
[target.win-64.dependencies]
python = "3.7"
[target.osx.dependencies]      # 'osx' matches osx-64 and osx-arm64
python = "3.11"
[target.unix.activation.env]
LD_LIBRARY_PATH = "$CONDA_PREFIX/lib"
```

Selectors: exact subdir, family (`win`/`osx`/`linux`/`unix`), or rich-platform `name`. Target platform must be a subset of `workspace.platforms`. (Inline `[package.target.*.host/build/run-dependencies]` is deprecated in favor of `if(...)` tables.)

## Channels & activation

```toml
[workspace]
channels = ["conda-forge", "pytorch", "nvidia"]
channel-priority = "strict"

[activation]
scripts = ["env_setup.sh"]                         # .sh/.bash Unix, .bat Windows (called, not sourced)
env = { LD_LIBRARY_PATH = "$CONDA_PREFIX/lib" }
```

Env-var priority: `task.env` > `activation.env` > `activation.scripts` > dependency activation > outside env.

## Conda + PyPI resolution

Conda resolves first; Pixi maps names and resolves PyPI for the rest. When both provide a package, conda wins. Prefer conda for native foundations (`python`, `numpy`, `scipy`, `pytorch`, toolchains, CUDA). ⚠ `conda-pypi-map`: bare mapping files **overlay** other mappings by default (they don't replace them).

## Pre-2025 → current migration

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
| `--subdir` (git deps) | `--subdirectory` (0.74) |
