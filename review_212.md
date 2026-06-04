Thanks — auto-detecting the platform and delegating to `install_aris_codex.sh` is a nice UX win. Two blockers before merge though (both independently confirmed by a Claude + GPT-5.5 xhigh cross-model review), plus one design check.

## 🔴 P1 — bash 3.2 + `set -u`: empty-array expansion aborts the delegation
The script runs `set -euo pipefail` under `#!/usr/bin/env bash`, and macOS ships /bin/bash 3.2.57. The new block does `for arg in "${ORIGINAL_ARGS[@]}"` and ends with `exec bash "$CODEX_INSTALLER" "${FILTERED_ARGS[@]}"`. Under bash 3.2 + `set -u`, expanding an **empty** array throws `unbound variable` and aborts — so running the installer with no extra args from a Codex-marked project dies *before* it can delegate (the exact feature this PR adds). The file already uses the safe pattern elsewhere (`[[ ${#arr[@]} -gt 0 ]]` guard before iterating).
→ Guard both expansions, e.g. `"${ORIGINAL_ARGS[@]+"${ORIGINAL_ARGS[@]}"}"`, or gate the loop/exec on a non-empty count.

## 🔴 P1 — detection isn't fail-closed: a Claude user can be silently mis-routed
`AGENTS.md` alone is treated as a Codex marker. But a fresh Claude project usually has no `.claude/` yet (the installer creates it), so a repo carrying `AGENTS.md` but no `CLAUDE.md`/`.claude/` yields has_codex=true, has_claude=false and is silently `exec`'d into the Codex installer — wrong skill set, no confirmation prompt, only a `log`-level message (suppressed under `--quiet`).
→ Require explicit `--platform` (or a confirmation) when no Claude marker exists, or only auto-delegate on stronger Codex markers (`.agents/` / `.codex/config.toml`), not `AGENTS.md` alone.

## 🟡 P2 — `--platform` stripping is token-based
A forwarded argument whose *value* is literally `--platform` would be removed along with the following token. Low likelihood, but avoidable by rebuilding the forwarded args during the real option parse.

## Design check
`--platform` was deliberately removed earlier (with an explicit error pointing Codex users to manual setup), so re-adding it reverses that decision. That's a legitimate choice — just please confirm it's intended, and double-check the inline usage/help text still renders coherently with the re-added flag.

The diff applies cleanly to current `main` and needs no inventory integration (no skills added). Fix the two P1s and this is good to go. Thanks for the contribution!
