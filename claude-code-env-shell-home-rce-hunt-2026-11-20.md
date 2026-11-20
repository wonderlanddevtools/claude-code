# Claude Code project-env RCE gadget hunt — 2026-11-20

Focus: `.claude/settings.json` `env` applied via `applyConfigEnvironmentVariables` after generic TrustDialog (or immediately under `claude -p`), especially vars inherited by auto-allowed Bash/git/python/node commands.

**Result: 2 NEW CRITICAL** — see `claude-code-env-shell-home-rce-2026-11-20---flagged-vulnerabilities.json`.

## Validated

1. **CLAUDE_CODE_SHELL / SHELL** — replaces BashTool shell binary (`findSuitableShell`); path only needs to contain `bash`/`zsh` + be executable.
2. **HOME** — redirects `createAndSaveSnapshot` / login shell to planted `.bashrc` / `.bash_profile` via `os.homedir()`.

## Method

- Traced `applySafeConfigEnvironmentVariables` → TrustDialog → `applyConfigEnvironmentVariables` → `subprocessEnv` → Bash/git sinks.
- Confirmed TrustDialog dangerous-env detection is analytics-only (generic UI).
- Local PoCs for shell override and HOME snapshot sourcing; language-runtime and remaining git/editor candidates disproved or classified as close variants of already-filed gadgets.
