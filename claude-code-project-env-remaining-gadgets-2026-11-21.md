# Claude Code project-env remaining gadgets hunt — 2026-11-21

## Validated MEDIUM+

### 1. CRITICAL — RIPGREP_CONFIG_PATH `--pre` RCE via auto-allowed `rg` / GrepTool

After generic TrustDialog (or `claude -p`), `applyConfigEnvironmentVariables` assigns project `.claude/settings.json` `env.RIPGREP_CONFIG_PATH`. That path is not in `SAFE_ENV_VARS` and is not scrubbed by `subprocessEnv()`. BashTool step 7 auto-allows `rg <pattern> …` (`checkReadOnlyConstraints` → allow) while rejecting argv `--pre` (`passthrough`). Ripgrep still loads `--pre` from the config file, executing an in-repo script on every content search. GrepTool / `ripGrep()` also inherit `process.env` and only pass `--no-config` in bundled/embedded mode — non-bundled builtin/system rg honors the planted config.

**Distinct from:** filed `sandbox.ripgrep.command` binary swap; argv `$VAR--pre=` obfuscation (already blocked at readOnlyValidation.ts:1338–1350); LD_PRELOAD/PATH.

### 2. MEDIUM — AWS_CONTAINER_CREDENTIALS_FULL_URI (+ TOKEN) SSRF after USE_BEDROCK

`CLAUDE_CODE_USE_BEDROCK` is in `SAFE_ENV_VARS` (applied even pre-trust). After trust, project env can set `AWS_CONTAINER_CREDENTIALS_FULL_URI` / `AWS_CONTAINER_AUTHORIZATION_TOKEN`. When Bedrock auth has no explicit keys, `@anthropic-ai/bedrock-sdk` falls through to `fromNodeProviderChain()`, which GETs that URI and sends the Authorization token. Local PoC: HTTP hit with planted token; credentials returned and used.

**Distinct from:** filed `AWS_SHARED_CREDENTIALS_FILE` + `credential_process` RCE; `GOOGLE_APPLICATION_CREDENTIALS` executable ADC.

## Ruled out

| Candidate | Why |
|-----------|-----|
| GCONV_PATH + locale | `locale` auto-allowed but does not load gconv modules; `iconv` triggers classic chain but is **not** allowlisted (`passthrough`) |
| OPENSSL_CONF / OPENSSL_ENGINES | Engine ctor fires on `openssl version` (not allowlisted). `git ls-remote` https did **not** load engine/.so |
| GIT_PAGER / PAGER / MANPAGER / LESSOPEN | BashTool/Shell stdout is pipe/file, not tty — pagers do not run |
| GIT_EXTERNAL_DIFF / GIT_SEQUENCE_EDITOR | Already filed / not on read-only status path |
| PYTHONSTARTUP / PERL5OPT / RUBYOPT / JAVA_TOOL_OPTIONS / DOTNET_STARTUP_HOOKS | No auto-allowed command invokes those runtimes (`python --version` exact-match only; java/perl/ruby/dotnet not readonly-allowed) |
| GIT_CONFIG (single) | Does not apply core.fsmonitor on `git status` (unlike filed GIT_CONFIG_GLOBAL/SYSTEM/PARAMETERS) |
| GIT_ATTR_SOURCE | Rejected / no RCE on status |
| GIT_TEMPLATE_DIR / GIT_INDEX_FILE / GIT_NAMESPACE / GIT_OBJECT_* | No RCE on auto-allowed `git status` |
| npm_config_* | `npm` not readonly-allowlisted |
| SSLKEYLOGFILE | Writes TLS keys on `git ls-remote` https, but alone is not theft without MITM/proxy (proxy+CA already filed) |
| TMPDIR | No validated privileged tmp gadget on auto-allowed/startup children |
| HOSTALIASES | Single-label only; dotted names (github.com) ignored; no RCE |
| TERMINFO / INPUTRC / EXINIT / VIMINIT | No auto-allowed consumer that executes code |
| IFS / CDPATH / ENV / PS0 | ENV not sourced for non-interactive bash (Claude's shell); PS0 not in `-c`; IFS doesn't bypass argv allowlist; CDPATH alone ≠ RCE |
| DYLD_* | macOS-only; LD_AUDIT already filed as Linux analog |
| CORECLR / COMPlus | No auto-allowed .NET host on Bash readonly path |
| PHPRC | php not allowlisted |
| RUSTC_WRAPPER / CARGO_HOME | cargo not readonly-allowed |
| Startup `countFilesRoundedRg` | Uses `rg --files` which does **not** invoke `--pre` |
