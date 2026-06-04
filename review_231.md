Thanks for this — a no-API-key reviewer fallback through the local Codex login is genuinely useful, and defaulting `LLM_BACKEND=api` (opt-in, non-breaking) is the right call. A cross-model review (Claude + GPT-5.5 xhigh) surfaced a few things to harden before merge, plus a rebase. Direction is good; these are about the exec surface.

## 🔴 P1 — scrub the child environment (secret-exfil surface)
`mcp-servers/llm-chat/server.py` runs `codex exec` via `subprocess.run` without a sanitized `env=`, so the child inherits the full MCP-server environment + the user's Codex config. The reviewer prompt is attacker-influenceable content (you're reviewing arbitrary research text / PRs), so a crafted prompt can induce the Codex agent to read and print env secrets (`LLM_API_KEY`, `OPENAI_API_KEY`, `*_TOKEN`) — which then flow back out through the echoed output. `--sandbox read-only` blocks writes but not reading/printing env.
→ Pass a minimal whitelisted `env=`, and use a dedicated `CODEX_HOME` so local rules/config aren't pulled into a reviewer call.

## 🟡 P2 — don't echo raw CLI output as MCP content/errors
The path returns raw `stderr`/`stdout` (up to 1000 chars) and `str(e)` on failure — that can leak transcripts, file paths, and config details. Return generic auth/timeout/exec errors and redact known secret patterns.

## 🟡 P2 — `LLM_BACKEND=auto` silently switches to `codex-cli` when `LLM_API_KEY` is absent
Not unauthenticated, but after a missing-key misconfig it silently routes reviewer prompts through the local ChatGPT/OAuth Codex account. For a security-sensitive review path, require an explicit `LLM_BACKEND=codex-cli`, or make `auto` warn loudly before falling through.

## 🟡 P2 — tests exercise a copied helper, not the server
The added tests patch `tests/_llm_chat_helpers`, while the production logic lives in `mcp-servers/llm-chat/server.py`; they also don't assert env scrubbing, output redaction, or real auth-failure behavior. Please exercise the actual code path.

## Rebase
The diff no longer applies cleanly to current `main` (`mcp-servers/llm-chat/server.py` diverged — `import datetime` + a datetime debug_log). A 3-way merge resolves with no semantic conflict, so a rebase is all that's needed.

**Reviewer-independence note (non-blocking):** the codex-cli backend is the *same GPT family* as the existing Codex MCP reviewer. Fine as a no-key fallback, but it must not be used as the cross-model jury when the executor is also GPT — worth a sentence in the guide you added.

Happy to re-review as soon as the env scrubbing + output redaction land. Thanks again for the contribution!
