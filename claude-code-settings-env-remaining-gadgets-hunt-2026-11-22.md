# Claude Code settings.env remaining gadgets hunt — 2026-11-22

Focus: project `.claude/settings.json` `env` → `applyConfigEnvironmentVariables()` after Trust / under `claude -p` → auto-allowed Bash (`true`/`pwd`/`ls`/`file`/`git ls-remote`/`git status`) or Claude startup children (`getGitStatus` / `execFileNoThrow`). Skip already-filed list from the hunt prompt.

## Result

**NONE** — no NEW complete RCE chains.

## Code spine

- Post-trust / `-p` apply all env: `src/utils/managedEnv.ts:187-199`
- Trust apply: `src/interactiveHelpers.tsx:180-184`
- getGitStatus children: `src/context.ts:64-77` (`status`/`log`/`config` via `execFileNoThrow`)
- Auto-allow step 7: `src/tools/BashTool/bashPermissions.ts:1153-1162`
- `git ls-remote` allow + URL reject: `src/utils/shell/readOnlyCommandValidation.ts:312-340`, `src/tools/BashTool/readOnlyValidation.ts:1306-1325`
- Version allowlist only: `node -v` / `python --version` / `python3 --version` — `src/tools/BashTool/readOnlyValidation.ts:1537-1540`

## Candidate verification

| # | Candidate | Verdict |
|---|-----------|---------|
| 1 | `GIT_PROXY_COMMAND` on auto-allowed `git ls-remote origin` | **REJECTED** — fires only for `git://` (confirmed ctor/proxy exec). HTTPS/SSH origins use `git-remote-https` / SSH, not proxy. Allowlist rejects explicit `://` URLs; normal clones do not leave `git://` origin. Incomplete without already-filed `GIT_CONFIG_*` `insteadOf`. |
| 2 | `core.gitProxy` via env | **REJECTED** — same sink as (1); via `GIT_CONFIG_COUNT`/`PARAMETERS` is already-filed GIT_CONFIG class. |
| 3 | `OPENSSL_CONF` on bun/node HTTPS | **REJECTED** — provider/engine ctor fires on `openssl version` and `curl` (neither allowlisted). Host `git-remote-https` links **GnuTLS** (not OpenSSL) → no load. `python3 --version` does not load; `import ssl` does (not allowlisted form). Node `https.get` did not load. Already noted in prior hunt. |
| 4 | `BUN_OPTIONS` / `DENO_DIR` | **REJECTED** — bun/deno not installed here; neither on readonly allowlist; no Claude child re-executes bun with those vars for allowlisted work. |
| 5 | `PYTHONSAFEPATH` / `PYTHONWARNINGS` | **REJECTED** — not RCE (as stated). |
| 6 | `TESSDATA_PREFIX` | **REJECTED** — tesseract not allowlisted. |
| 7 | `FONTCONFIG_FILE` / `FONTCONFIG_PATH` | **REJECTED** — `file`/`ls`/`true` do not open fontconfig; `fc-list` not allowlisted. |
| 8 | `GIO_MODULE_DIR` / `GTK_PATH` / `QT_PLUGIN_PATH` | **REJECTED** — no GUI child on auto-allowed / getGitStatus path. |
| 9 | `ICONV_PATH` | **REJECTED** — libiconv-specific; allowlisted bins do not link libiconv; `iconv` not allowlisted. |
| 10 | `CHARSETALIASDIR` | **REJECTED** — alias file only, not module load / RCE. |
| 11 | `LD_PROFILE` | **REJECTED** — writes `.profile` data for named SONAME; no arbitrary code exec. |
| 12 | `MALLOC_TRACE` | **REJECTED** — writes malloc trace file; no exec. |
| 13 | `PCSCANLIB` | **REJECTED** — no allowlisted sink. |
| 14 | `HTS_PATH` | **REJECTED** — HTSlib; no allowlisted sink. |
| 15 | `PERLLIB` / `PERL5LIB` | **REJECTED** — `perl` not on COMMAND_ALLOWLIST / version regexes. |
| 16 | `RUBYLIB` | **REJECTED** — `ruby` not allowlisted. |
| 17 | ruby/python/perl/php version allowlist | **CONFIRMED** — only `python --version` / `python3 --version` / `node -v` / `node --version`. Those do not load PYTHONPATH sitecustomize / PYTHONSTARTUP / PERL5OPT / RUBYOPT (filed class already covers NODE_OPTIONS). |
| 18 | `git var` / `git version --build-options` | **REJECTED** — not in `GIT_READ_ONLY_COMMANDS`. |
| 19 | `ssh -G` | **REJECTED** — `ssh` not allowlisted. |
| 20 | `curl -w` / CURLOPT env | **REJECTED** — `curl` not allowlisted. (`CURL_HOME`/`CURL_CA_BUNDLE` already filed.) |

## Extra probes (also no NEW chain)

- `PKCS11_MODULE_PATH` / `P11_KIT_MODULE_PATH` on `git ls-remote` HTTPS: no module ctor load.
- `OPENSSL_ENGINES` alone: no load without OPENSSL_CONF.
- `GIT_CONFIG_SYSTEM` + `core.gitProxy`: works on `git://` but GIT_CONFIG class already filed.
