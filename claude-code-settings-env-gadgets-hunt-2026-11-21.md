# Claude Code settings.env gadget hunt — 2026-11-21

Focus: project `.claude/settings.json` env → `applyConfigEnvironmentVariables()` after Trust / under `claude -p` → auto-allowed Bash sinks. Distinct from the filed skip-list.

## Validated (3)

1. **LD_AUDIT** (critical) — glibc audit module load on `true`/`pwd`/`ls` via `bash -c`; distinct from LD_PRELOAD.
2. **GIT_SSH_COMMAND / GIT_SSH** (critical) — RCE on auto-allowed `git ls-remote origin` / `git remote show origin` when origin is SSH.
3. **MAGIC** (high) — GNU `file` dumps host files listed in MAGIC as warning lines; any auto-allowed `file README.md`.

See `claude-code-settings-env-gadgets-2026-11-21---flagged-vulnerabilities.json`.

## Code spine

- `applyConfigEnvironmentVariables`: `src/utils/managedEnv.ts:187-199`
- Trust apply: `src/interactiveHelpers.tsx:180-184`
- `-p` apply: `src/main.tsx:1952-1965`
- subprocess inherit: `src/utils/Shell.ts:316-323`, `src/utils/subprocessEnv.ts:79-90`
- step 7 allow: `src/tools/BashTool/bashPermissions.ts:1153-1162`

## Ruled out

PAGER/MANPAGER/GIT_PAGER/LESSOPEN (non-tty redirect); OPENSSL_CONF (no allowlisted openssl user; python3 --version does not load it); GCONV_PATH (locale no-op; iconv not allowlisted); ENV/BASHOPTS/PROMPT_COMMAND; language *OPT/STARTUP hooks; GIT_ASKPASS; GIT_PROXY_COMMAND (git:// only); GIT_TEMPLATE_DIR; NPM/COREPACK; SSLKEYLOGFILE alone; TERMINFO/INPUTRC/HOSTALIASES; LD_LIBRARY_PATH (incomplete PoC); MAGICK_/GS_LIB.
