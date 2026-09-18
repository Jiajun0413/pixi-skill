---
name: pixi-skill
description: "Manage Pixi workspaces, manifests, environments, tasks, lockfiles, and global tool environments. Use when working with pixi.toml, pyproject.toml configured for Pixi, pixi.lock, or Pixi global manifests; when adding or removing Conda/PyPI dependencies; when defining tasks, activation, features, or target-specific settings; or when installing and exposing global CLI tools."
---

# Pixi (0.81, 2026)

Verify behavior via `pixi run`/`pixi exec`, never a bare shell — activation is part of the runtime contract. Change manifests declaratively (`pixi add/remove` or TOML edits); never edit `.pixi/` by hand.

## Decision rules

- `pyproject.toml` for Python-first projects; `pixi.toml` for mixed/non-Python.
- Conda deps first (esp. `python`, `numpy`, `pytorch`, CUDA libs); `--pypi` only when a package is unavailable on conda or the project already uses wheels. Default channel is `conda-forge` only — add `pytorch`/`nvidia` explicitly for CUDA.
- Repeatable commands → `[tasks]` (dependency key is `depends-on`, hyphen); one-offs → `pixi exec`.
- `[activation.env]` for stable vars; activation scripts only for dynamic values.
- Features/environments/target tables only when the project truly needs separate stacks.

## update vs upgrade vs lock

- `pixi update [pkg]` — re-resolve within existing manifest constraints.
- `pixi upgrade [pkg]` — LOOSENS specs and rewrites manifest+lock (destructive to pins).
- `--frozen` installs lock as-is; `--locked` fails on stale lock (CI gate). Both also work on standalone scripts with adjacent lock files.

## Global tools (conda MatchSpecs only — no `--pypi` yet)

- `pixi global install <pkg>` new env · `global add --environment E <pkg>` add dep · `global remove --environment E <pkg>` drop dep · `global uninstall E` delete env · `global sync` after editing `$PIXI_HOME/manifests/pixi-global.toml`.

## Network / config traps

- `~/.config/pixi/config.toml` sections are kebab-case: `[pypi-config] index-url`, `[mirrors]` (maps `conda.anaconda.org/<ch>` → mirror URL list), `[repodata-config] disable-sharded`/`disable-zstd`.
- ⚠ Writing `[repodata]` (old name) is SILENTLY IGNORED — no error, no effect.
- Repodata availability cache is keyed by canonical URL, valid 14 days → clear the repodata cache dir (`~/.cache/rattler/cache/repodata`, or its NFS-redirected `/tmp/pixi-cache-*/repodata`) after changing mirrors.
- Campus-network mirrors (CERNET/NJU): zst yes, sharded repodata no → `disable-sharded = true`. `HTTPS_PROXY` is respected (`pixi self-update` needs it behind firewalls).

## Newer surface

- `pixi import <file>` — import environment.yml / requirements.txt into a workspace env (`--format`, `-e`, `-p`).
- `pixi run script.py` — PEP 723 `/// script` header (Python, stable); `/// conda-script` (any language, experimental) embeds deps + `entrypoint` in-file — enable via `pixi config set experimental.conda-script true --global`; script-aware commands take `--script`; remote URLs/Gists run directly (0.81).
- `pixi exec [COMMAND]...` — one-off temp env; `pixi exec -- <cmd>` guesses the package from the command, else specs via `-s`/`--with` (conda only). Purge: `pixi clean cache --exec`.

## References (read on demand)

- `references/manifest.md` — full TOML syntax: workspace, features/environments, tasks, pypi-dependencies, CUDA rich-platform requirements, target tables, activation, pre-2025 migration. Read when hand-editing or migrating a manifest.
- `references/operations.md` — lockfile flags (v7), global-tools manifest/CLI detail, exec, auth, troubleshooting. Read for env/CLI/lock problems.
- `references/config-network.md` — config.toml anatomy, mirror/cache mechanics, proxy, `pixi import`, single-file scripts. Read for mirror/network/setup issues or script workflows.
