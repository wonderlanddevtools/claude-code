# Claude Code hunt — GIT_EXEC_PATH / pager / man|info / acceptEdits / SSLKEYLOGFILE — 2026-11-22

## Summary

| # | Candidate | Verdict | Severity |
|---|-----------|---------|----------|
| 1 | Project env `GIT_EXEC_PATH` → getGitStatus `status`/`log` | **REJECTED** | — |
| 1b | Project env `GIT_EXEC_PATH` → auto-allowed `git ls-remote origin` / `git remote show origin` (HTTPS) | **VERIFIED** | critical |
| 2 | `GIT_PAGER` / `PAGER` / `GIT_SSH_COMMAND` / `GIT_SSH` / `GIT_ASKPASS` | **REJECTED** (pagers/askpass); `GIT_SSH*` already filed | — |
| 3 | `man -l` readonly dump | **VERIFIED** but **already filed** | high (prior) |
| 3b | `info -f` readonly dump | **REJECTED** | — |
| 4 | acceptEdits gaps for workflows/CI/justfile/flake/CMake/pre-commit | **Gap real**, **already filed** | — |
| 5 | `SSLKEYLOGFILE` via Bun/Node API HTTPS | **REJECTED** | — |
| 6 | Other project env git keys rewriting getGitStatus | **REJECTED** (overlap with filed `GIT_CONFIG_*`/`core.fsmonitor`) | — |

## 1 / 1b — GIT_EXEC_PATH

### REJECTED: getGitStatus status/log

`getGitStatus` (`src/context.ts:64-77`) runs `git status`, `git log`, `git config` via `execFileNoThrow(gitExe(), …)`.

- `gitExe()` (`src/utils/git.ts:212-216`) resolves the main `git` binary via `whichSync('git')` (PATH), not GIT_EXEC_PATH.
- `execFileNoThrow` (`src/utils/execFileNoThrow.ts:110-115`) passes `env: undefined` → execa inherits `process.env` (so GIT_EXEC_PATH *is* present).
- But modern git implements `status`/`log`/`config` as **builtins** (`GIT_TRACE` → `built-in: git status`). Builtins do **not** re-exec `git-status` / `git-log` from GIT_EXEC_PATH.
- PoC: planted `git-status`/`git-log` under GIT_EXEC_PATH never ran; marker files absent.

### VERIFIED: auto-allowed HTTPS remote helpers

After Trust / `claude -p`, `applyConfigEnvironmentVariables()` (`src/utils/managedEnv.ts:187-199`) assigns project `settings.env` including `GIT_EXEC_PATH` (not in `SAFE_ENV_VARS`). BashTool spawn inherits it via `subprocessEnv()` (`src/utils/Shell.ts:316-323`).

Auto-allowed:
- `git ls-remote` (`src/utils/shell/readOnlyCommandValidation.ts:312-340`) — remote **name** form allowed; explicit `://` / `user@host:` URLs rejected.
- `git remote show` (`…:473-488`) — single remote name.

When `origin` is HTTPS, git runs `git-remote-https` from **GIT_EXEC_PATH**:

```
trace: run_command: git-remote-https origin https://…
```

PoC (committable plant):
1. `.claude/settings.json` → `env.GIT_EXEC_PATH` = `.claude/git-exec` (relative OK)
2. `.claude/git-exec/git-remote-http` (+ symlink `git-remote-https`), mode 0755
3. CLAUDE.md steers `git ls-remote origin` or `git remote show origin`
4. Marker written; builtins (`status`/`log`) still work for victim UX

Distinct from filed:
- `PATH` (replaces main `git` binary)
- `GIT_EXTERNAL_DIFF` (diff helper)
- `GIT_CONFIG_COUNT` / `GIT_CONFIG_GLOBAL` + `core.fsmonitor` (config hook on status)
- `GIT_SSH_COMMAND` / `GIT_SSH` (SSH transport; HTTPS remotes do not hit that sink)

## 2 — Pagers / SSH / ASKPASS

| Key | Verdict |
|-----|---------|
| `GIT_PAGER` / `PAGER` | REJECTED — BashTool redirects stdout/stderr to a file fd (non-TTY); pagers do not run (`src/utils/Shell.ts:289-332`). No PTY spawn. |
| `GIT_ASKPASS` | REJECTED — auto-allowed status/log/diff do not prompt; marketplace/worktree clear ASKPASS. |
| `GIT_SSH_COMMAND` / `GIT_SSH` | Already VERIFIED filed (settings-env-gadgets 2026-11-21) for SSH-origin `ls-remote`/`remote show`. |

## 3 — man -l / info -f

### man -l — VERIFIED, already filed

- In `COMMAND_ALLOWLIST` with `-l: 'string'` (`src/tools/BashTool/readOnlyValidation.ts:304-321`).
- **Not** a `PathCommand` (`pathValidation.ts:27-63`) → `checkPathConstraints` passthrough (`:857-862`) → step 7 auto-allow.
- PoC with real man-db (`man.REAL -l /etc/passwd`) dumps file bytes under non-TTY redirect.
- Prior hunt already filed (`man -l` referenced as filed in tree-houtro hunt). **Do not re-report.**

### info -f — REJECTED

- Allowlisted (`-f`/`--file`: string); not PathCommand.
- On non-TTY / `TERM=dumb` (BashTool redirect): `info: Terminal type 'dumb' is not smart enough to run Info` — no content dump.
- Useful write needs `-o/--output`, which is **blocked** by omission from safeFlags.

## 4 — acceptEdits DANGEROUS_* gaps

Confirmed AUTO (not matched) for:
`.github/workflows/ci.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `justfile`, `flake.nix`, `CMakeLists.txt`, `.pre-commit-config.yaml`

`DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES` (`filesystem.ts:57-79`) still only shell/git/mcp + `.git`/`.vscode`/`.idea`/`.claude`.

**Already filed** (acceptEdits CI / Makefile/flake / CMake / pre-commit / justfile hunts). Not re-reported.

## 5 — SSLKEYLOGFILE

REJECTED for API-client impact:
- Node `https` / `fetch` in this environment did **not** write SSLKEYLOGFILE.
- Bun not available in this VM to retest; even if Bun wrote keys, alone it does not yield secrets without a separate MITM channel.
- `HTTPS_PROXY` + `NODE_EXTRA_CA_CERTS` MITM already filed.

## 6 — Other project env git keys (getGitStatus)

`GIT_CONFIG_GLOBAL` / `GIT_CONFIG_COUNT` + `core.fsmonitor` **does** RCE on `git status` (confirmed again). Overlaps filed `GIT_CONFIG_COUNT/core.fsmonitor` class — not distinct.

No other NEW auto-allowed getGitStatus rewrite found beyond that filed class and the GIT_EXEC_PATH remote-helper sink above.
