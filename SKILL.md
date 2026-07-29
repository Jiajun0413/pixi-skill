---
name: pixi-skill
description: "Manage Pixi workspaces, manifests, environments, tasks, lockfiles, and global tool environments. Use when working with pixi.toml, pyproject.toml configured for Pixi, pixi.lock, or Pixi global manifests; when adding or removing Conda/PyPI dependencies; when defining tasks, activation, features, or target-specific settings; or when installing and exposing global CLI tools."
---

# Pixi

Manage Pixi projects (pixi v0.7x / 2026). For full manifest syntax, command lists, behavior notes, and pre-2025 migration mappings, read `references/official-notes.md`.

## Working style

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

## Current-state facts

Pixi changed significantly pre-2025 → 2026. These are the current correct forms.

- Top-level manifest table is `[workspace]` (`[tool.pixi.workspace]` in pyproject); required fields `name`, `channels`, `platforms`. Default channel is `conda-forge` only — add `pytorch`/`nvidia` explicitly for CUDA.
- System requirements (CUDA, glibc, etc.) go inline on `workspace.platforms` entries.
- Task dependency field is `depends-on` (hyphen), accepts a string or list.
- `pixi global install <pkg>` creates a new isolated global env; `pixi global add <pkg> --environment <env>` adds a dep to an existing env; `pixi global uninstall <env>` removes a whole env; `pixi global remove <pkg>` removes a package.
- `pixi update [pkg]` re-resolves within manifest constraints; `pixi upgrade [pkg]` loosens the manifest and rewrites manifest+lock.
- Auth is `pixi auth login <host>` / `pixi auth logout` / `pixi auth status`.
- Lockfile: `--frozen` installs from lock as-is; `--locked` fails if lock is stale (CI gate); the two conflict.

## References

- `references/official-notes.md` — full manifest syntax (environments, tasks, PyPI deps, system requirements, target tables, channels, activation), lockfile flags, global tools, command lists, auth, troubleshooting, and pre-2025 → current migration table. Read when editing manifests by hand, debugging activation/native errors, or migrating an older manifest.
