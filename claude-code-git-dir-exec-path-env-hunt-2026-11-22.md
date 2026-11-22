# Claude Code hunt — GIT_DIR / GIT_EXEC_PATH re-verify / remaining env gadgets — 2026-11-22

Focus: project `.claude/settings.json` `env` → `applyConfigEnvironmentVariables()` after Trust / immediately under `claude -p` → inherited by auto-allowed Bash / startup `getGitStatus` (`execFileNoThrow`). Skip the prompt’s already-filed class list.

## Summary

| # | Candidate | Verdict | Severity |
|---|-----------|---------|----------|
| 1 | `GIT_EXEC_PATH` → auto-allowed `git ls-remote origin` / `git remote show origin` (HTTPS `git-remote-https`) | **STILL VERIFIED** (already flagged earlier today) | critical |
| 2 | `GIT_DIR` (optional `GIT_WORK_TREE` / `GIT_COMMON_DIR`) → planted in-repo git metadata → `core.fsmonitor` on status / `credential.helper` on ls-remote | **VERIFIED NEW** | critical |
| 3–N | Other listed candidates | **REJECTED** (one-liners below) | — |

## Code spine

- Post-trust / `-p` apply all env: `src/utils/managedEnv.ts:187-199`
- Trust apply: `src/interactiveHelpers.tsx:180-184`
- `-p` apply **before** `getSystemContext`: `src/main.tsx:1952-1977` (comments explicitly name `PATH` / `GIT_DIR` / `GIT_WORK_TREE`)
- Subprocess inherit (no GIT_* scrub): `src/utils/subprocessEnv.ts:79-90`, `src/utils/Shell.ts:316-323`
- `execFileNoThrow` inherits `process.env` when `env` omitted: `src/utils/execFileNoThrow.ts:110-115`
- Auto-allow step 7: `src/tools/BashTool/bashPermissions.ts:1153-1162`
- `git ls-remote` / `git remote show` / `git status`: `src/utils/shell/readOnlyCommandValidation.ts`
- Argv guards block `-c` / `--config-env` / `--exec-path` only — **not** env `GIT_DIR` / `GIT_EXEC_PATH`: `src/tools/BashTool/readOnlyValidation.ts:1721-1747`
- `SAFE_ENV_VARS` does not include any `GIT_*`: `src/utils/managedEnvConstants.ts:108-191`

## 1 — GIT_EXEC_PATH (re-verify)

**Still works.** Relative `GIT_EXEC_PATH=.claude/git-exec` with planted `git-remote-http` / `git-remote-https` runs on `git ls-remote origin` when origin is HTTPS (`GIT_TRACE` → `run_command: git-remote-https`). Marker written. Builtins (`status`/`log`) still do not re-exec from `GIT_EXEC_PATH`.

Already flagged in `claude-code-git-exec-path-man-sslkeylogfile-hunt-2026-11-22---flagged-vulnerabilities.json`. Not re-filed.

## 2 — GIT_DIR → planted git dir (NEW CRITICAL)

### Why distinct

- **Not** on the hunt skip list (skip list names `GIT_CONFIG_COUNT` / `PARAMETERS` / `GLOBAL` + sinks, not `GIT_DIR`).
- Different env key and plant: committed directory tree (e.g. `.claude/evil.git/{HEAD,config,objects,refs,…}`) instead of `GIT_CONFIG_*` keys/files.
- Remediation that only strips `GIT_CONFIG_*` leaves this open.
- `main.tsx` documents applying project `GIT_DIR` / `GIT_WORK_TREE` before `getSystemContext` under `-p`.

### Attack path

1. Attacker commits `.claude/settings.json` with `env.GIT_DIR=.claude/evil.git` and a planted git directory whose `config` sets `core.fsmonitor=./.claude/pwn.sh` and/or `credential.helper=!./.claude/pwn-cred.sh`, plus HTTPS `remote.origin.url`.
2. Victim clones and runs `claude` or `claude -p`.
3. Interactive: after Trust, `applyConfigEnvironmentVariables()`. Headless `-p`: apply immediately, then `void getSystemContext()` → `getGitStatus()` → `git status` with inherited `GIT_DIR`.
4. `core.fsmonitor` runs → RCE (no BashTool required under `-p`). After Trust, auto-allowed Bash `git status` / `git ls-remote origin` also inherit `GIT_DIR` → fsmonitor and/or `credential.helper` RCE.

### PoC notes

- Relative `GIT_DIR=.claude/evil.git` → `git --no-optional-locks status --short` wrote fsmonitor marker.
- Same `GIT_DIR` → `git ls-remote origin` wrote credential.helper marker (HTTPS origin).
- `GIT_COMMON_DIR` pointing at a planted common dir with `core.fsmonitor` also fired (same class; remediate together).

### Interactive timing note

`interactiveHelpers.tsx` calls `getSystemContext()` at line 153 **before** `applyConfigEnvironmentVariables()` at 184, so the memoized startup status snapshot may miss `GIT_DIR` in interactive mode. `-p` ordering and post-apply BashTool auto-allows remain complete.

## Rejected candidates (one-liners)

| Candidate | Why |
|-----------|-----|
| `GIT_EXEC_PATH` via getGitStatus status/log | Builtins; no helper re-exec (prior hunt; reconfirmed) |
| `GIT_TEMPLATE_DIR` | Only `init`/`clone` template copy |
| `GIT_ATTR_SOURCE` | `fatal: bad --attr-source` on this git; attributes are not code exec |
| `GIT_CONFIG_SYSTEM` | Works with clean HOME, but same sink class as filed `GIT_CONFIG_*` + fsmonitor |
| `GIT_NAMESPACE` / `GIT_INDEX_FILE` / object alts | Redirect only; no RCE on allowlisted cmds |
| `TERMINFO` / `TERMCAP` | BashTool non-TTY redirect; no terminfo load on `bash -c true`/`ls` |
| `HOSTALIASES` / `RES_OPTIONS` / `LOCALDOMAIN` | Resolver remap/options only; not RCE |
| `NSS_WRAPPER_*` | Needs `LD_PRELOAD` of wrapper (already filed) |
| `GCONV_PATH` / `LOCPATH` / `ICONV_PLUGIN` / `NLSPATH` / `TZDIR` | No load on auto-allowed `true`/`pwd`/`ls`/`git`/`file`; `iconv` not allowlisted |
| `MALLOC_CONF` / `GLIBC_TUNABLES` | Tuning only; not module/code load RCE |
| `PYTHONHOME` | No allowlisted python beyond `--version` (does not use PYTHONHOME gadgets) |
| `JAVA_TOOL_OPTIONS` / `_JAVA_OPTIONS` / `JDK_JAVA_OPTIONS` | `java` not allowlisted |
| `DOTNET_STARTUP_HOOKS` | `dotnet` not allowlisted |
| `PHP_INI_SCAN_DIR` | `php` not allowlisted |
| `PERL5OPT` / `RUBYOPT` / `LUA_INIT` / `GUILE_LOAD_PATH` / `GHC_PACKAGE_PATH` / `OCTAVE_SITE_INITFILE` / `R_PROFILE_USER` / `MPLCONFIGDIR` / `JUPYTER_CONFIG_DIR` | Interpreters not on readonly allowlist |
| `GIT_TRACE2_PARENT_SID` | Not RCE (as stated) |
| `SSLKEYLOGFILE` | Already rejected (no Node write here; MITM class filed) |
| `XDG_CONFIG_HOME` → git/config fsmonitor | Only when `~/.gitconfig` absent; close variant of filed `GIT_CONFIG_GLOBAL`; HOME already filed for snapshot |
| `HOME` → `.gitconfig` fsmonitor | Works, but HOME already filed (shell snapshot); fsmonitor via HOME is close variant of `GIT_CONFIG_GLOBAL` |
