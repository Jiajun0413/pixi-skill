# Pixi official notes

Primary source when the workflow is unclear — official docs (canonical at `pixi.prefix.dev/latest/`):

- Overview: `https://pixi.prefix.dev/latest/`
- Getting started: `https://pixi.prefix.dev/latest/getting_started/`
- Manifest reference: `https://pixi.prefix.dev/latest/reference/pixi_manifest/`
- pyproject.toml: `https://pixi.prefix.dev/latest/python/pyproject_toml/`
- CLI index: `https://pixi.prefix.dev/latest/reference/cli/pixi/`
- Multi-environment: `https://pixi.prefix.dev/latest/workspace/multi_environment/`
- Multi-platform: `https://pixi.prefix.dev/latest/workspace/multi_platform_configuration/`
- Environment variables: `https://pixi.prefix.dev/latest/reference/environment_variables/`
- `pixi shell-hook`: `https://pixi.prefix.dev/latest/reference/cli/pixi/shell-hook/`
- Conda and PyPI: `https://pixi.prefix.dev/latest/concepts/conda_pypi/`
- Lock file: `https://pixi.prefix.dev/latest/workspace/lock_file/`
- Authentication: `https://pixi.prefix.dev/latest/deployment/authentication/`
- Global tools: `https://pixi.prefix.dev/latest/global_tools/introduction/`
- Global manifest: `https://pixi.prefix.dev/latest/global_tools/global_manifest/`
- Dependency types: `https://pixi.prefix.dev/latest/build/dependency_types/`
- Changelog: `https://pixi.prefix.dev/latest/CHANGELOG/`

## Official concepts to preserve

### Manifest choice

- Pixi supports both `pixi.toml` and `pyproject.toml`.
- Official guidance: `pyproject.toml` for Python projects; `pixi.toml` for non-Python or mixed-language.
- The required top-level table is `[workspace]` (or `[tool.pixi.workspace]` in pyproject). `name`, `channels`, `platforms` are required. The default channel list is `conda-forge` only.
- Preserve the current manifest format unless migration is part of the task.

### Dependency resolution

- Pixi supports Conda and PyPI in one project, resolved Conda-first:
  1. Resolve Conda dependencies.
  2. Map Conda packages to PyPI names.
  3. Resolve remaining PyPI dependencies.
- When both ecosystems provide a package, the Conda package wins.
- PyPI resolution may fail because Conda already fixed an incompatible transitive version.

### Activation behavior

- Pixi activation is more than prepending `bin` to `PATH`; dependency activation scripts also run.
- Behavior can differ between `pixi run`, `pixi exec`, `pixi shell`, and a shell/IDE that only points at the env binary.
- Customize in the manifest with `[activation]`, `[activation.env]`, `[target.unix.activation.env]`, `[target.win.activation.env]`.
- Scripts are `.sh`/`.bash` on Unix and `.bat` on Windows; they are *called*, not sourced — only env-var mutations persist into `pixi run`/`pixi shell`.

```toml
[target.unix.activation.env]
ENV_VAR = "$OTHER_ENV_VAR/unix-value"

[target.win.activation.env]
ENV_VAR = "%OTHER_ENV_VAR%\\windows-value"
```

### Environment variable priority

`task.env` > `activation.env` > `activation.scripts` > dependency activation scripts > outside environment variables.

### Inspecting activation

- `pixi shell-hook` prints the activation script for the current shell.
- `pixi shell-hook --json` shows the env vars activation would set.
- Use when diagnosing differences between CLI, IDE, task runner, and login shell.

### Environment layout and repair

- Environments live under `.pixi/envs/<name>` by default.
- State is derived from the manifest and lockfile.
- Prefer Pixi repair commands over manual edits in `.pixi/`:
  - `pixi install` — install/refresh from lock.
  - `pixi reinstall` — resync the installed prefix to the lock.
  - `pixi clean` — remove envs/cache.

### System requirements — declared inline on `workspace.platforms`

- The old `[system-requirements]` table is deprecated; declare virtual packages inline:

```toml
[workspace]
platforms = [
  "osx-arm64",
  { platform = "linux-64", cuda = "12.0", glibc = "2.28" },
  { name = "gpu", platform = "linux-64", cuda = { driver = "12.0", arch = "8.6" } },
]
```

- Friendly keys: `platform` (required), `name`, `cuda` (`__cuda`), `glibc` (`__glibc`), `macos`/`osx` (`__osx`), `linux` (`__linux`), `windows` (`__win`), `archspec` (`__archspec`).
- CLI: `pixi workspace platform add linux-64 --cuda 12`, `pixi workspace platform edit`, `pixi workspace platform list`.
- Per-env `CONDA_OVERRIDE_*` env vars still override detected values.

### Lockfile flags

- `pixi.lock` (format v6) is backward-compatible only — older lock + newer pixi works; newer lock + older pixi fails. Don't hand-edit.
- `--frozen` installs from lock as-is (no update even on mismatch); `--locked` refuses to run if lock is out of date (CI gate); the two conflict.
- `pixi lock --check` exits non-zero on drift; `pixi update` re-resolves within manifest constraints; `pixi upgrade` loosens the manifest and updates manifest+lock.

### Global tools

- Managed with `pixi global ...`; declarative manifest at `$PIXI_HOME/manifests/pixi-global.toml` (`PIXI_HOME` defaults to `~/.pixi`); env prefixes under `~/.pixi/global`.
- Each install lives in an isolated conda env (pipx-style); only exposed executables go on `PATH`.
- After manual global-manifest edits (or pulling one via VCS), run `pixi global sync` to rebuild envs from the manifest.

## General guidance derived from official behavior

### Native runtime issues

- If a package fails with shared-library / ABI / compiler-runtime errors, check activation and dependency source before changing package versions.
- Common variables for native builds/runtime lookup: `LD_LIBRARY_PATH`, `DYLD_FALLBACK_LIBRARY_PATH`, `PKG_CONFIG_PATH`, `CMAKE_PREFIX_PATH`, `CC`, `CXX`, `CUDA_HOME`.
- Put stable project-level values in activation or task config rather than personal shell startup files.

### Conda/PyPI mixed environments

- Prefer Conda for foundational native dependencies and toolchains.
- Prefer PyPI for leaf packages or packages unavailable on Conda.
- Beware PyPI wheels that expect runtime libraries differing from those the env provides.

## Useful commands

### Workspace

- `pixi info`
- `pixi list`
- `pixi list --explicit`
- `pixi tree`
- `pixi task list`
- `pixi add <pkg>` / `pixi add --pypi <pkg>` / `pixi add --host|--build|--editable <pkg>`
- `pixi remove <pkg>`
- `pixi install`
- `pixi reinstall`
- `pixi clean`
- `pixi run <task>`
- `pixi exec -- <cmd>`
- `pixi shell`
- `pixi shell-hook` / `pixi shell-hook --json`
- `pixi lock` / `pixi lock --check`
- `pixi update [pkg]` / `pixi upgrade [pkg]`
- `pixi workspace channel|platform|environment|feature …`
- `pixi self-update`

### Global tools

- `pixi global install <pkg>` — install a **new** global tool (isolated env per package by default; `--environment <name>` groups packages, `--with <dep>` adds non-exposed deps, `--expose name=exe` aliases, supports `--platform` and git/path sources)
- `pixi global add <pkg> --environment <env>` — add a dependency to an **existing** global env (requires `--environment`; use `--expose` to expose executables)
- `pixi global list`
- `pixi global uninstall <env>` (remove a whole env)
- `pixi global remove <pkg>` (remove a package from an env — opposite of `add`)
- `pixi global update` / `pixi global update <env>`
- `pixi global sync` (rebuild from manifest after manual edits / VCS pull)
- `pixi global expose add <name>=<exe>` / `pixi global expose remove <name>`
- `pixi global edit`
- `pixi global tree [<env>]`

### Authentication

- `pixi auth login <host>` (`--token`, `--conda-token`, `--username`/`--password`, `--s3-*`, `--oauth*`)
- `pixi auth logout <host>`
- `pixi auth status`
