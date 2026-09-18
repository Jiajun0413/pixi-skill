# Pixi config & network reference

Read for mirror/network/setup issues, cache weirdness, or single-file script workflows.

## Config file anatomy

Global: `~/.config/pixi/config.toml`; per-project: `.pixi/config.toml` (higher priority). CLI: `--config-file <path>`, `--no-config`. All section names are **kebab-case**. Config is shared with other rattler-based tools (rattler-build, rattler-index) since 0.78.

```toml
[pypi-config]
index-url = "https://mirrors.cernet.edu.cn/pypi/web/simple"

[repodata-config]
disable-sharded = true     # mirror has no repodata_shards.msgpack.zst
disable-zstd = true        # only if mirror lacks .zst

[mirrors]
"https://conda.anaconda.org/conda-forge" = ["https://mirrors.cernet.edu.cn/anaconda/cloud/conda-forge/"]
"https://conda.anaconda.org/bioconda" = ["https://mirrors.cernet.edu.cn/anaconda/cloud/bioconda/"]
```

⚠ Pitfall: `[repodata]` (old name) is **silently ignored** — use `[repodata-config]`.

## Repodata cache mechanics

- Availability flags (`has_zst` etc.) are keyed by **canonical URL** (`conda.anaconda.org/...`) and cached for **14 days** → after switching mirrors, delete the repodata cache: `rm -rf ~/.cache/rattler/cache/repodata` (or the redirected dir below), else stale flags cause wrong-format fetches/403 loops.
- On NFS/SMB/CephFS homes the cache auto-redirects to `/tmp/pixi-cache-<user>/` (see run warnings); override via `[cache.repodata]`/`[cache.pypi-mapping]` or `[cache.netfs-redirect] = "never"`.

## Mirrors (campus network)

- CERNET: `mirrors.cernet.edu.cn/anaconda/cloud/<channel>` (302 → USTC chinanet node) and `/pypi/web/simple`. Serves `.zst`; sharded repodata 404 → keep `disable-sharded = true`.
- NJU: `mirrors.nju.edu.cn/anaconda/cloud/<channel>`, pypi `/pypi/web/simple`. Same zst/sharded profile.
- Before switching mirrors, probe health AND format availability (zst/sharded); clear the repodata cache afterwards (see above).
- Pixi respects `HTTPS_PROXY` (needed for `pixi self-update` / GitHub behind firewalls).

## Offline mode

`--offline` runs from local cache only, with strict solve semantics (locally available packages only).

## pixi import

`pixi import <file>` imports into an existing workspace env: `--format conda-env` (environment.yml) or `--format pypi-txt` (requirements.txt); auto-detects if omitted. `-e <env>` names the target env (without `--feature`, content is written inline on the env); `-p` adds platforms as needed.

## Single-file scripts

- `pixi run script.py` — PEP 723 `# /// script` header (stable, Python): `dependencies` plus `[tool.pixi.dependencies]`/`[tool.pixi.workspace]` for conda deps/channels.
- `/// conda-script` (any language, experimental): enable once via `pixi config set experimental.conda-script true --global`. Block carries `channels`, `entrypoint` (e.g. `entrypoint = "Rscript ${SCRIPT}"`), `[dependencies]`, ends with `# /// end-conda-script`; `${SCRIPT}` and `${CACHE}` are usable in entrypoint.
- Script-aware commands: `run/install/add/remove/lock --script`. Remote scripts (HTTP(S), GitHub Gists) download, reuse a cached env, and run (0.81+). Adjacent lock files honored by `update`/`upgrade`.
- uv's cache lives inside the pixi cache dir.

## Troubleshooting network

- 403 on repodata → mirror missing that format: check zst/sharded; set `[repodata-config]` accordingly; clear cache.
- Solves hang → mirror down or proxy missing; `pixi info` shows resolved config; `--offline` to test cache-only.
