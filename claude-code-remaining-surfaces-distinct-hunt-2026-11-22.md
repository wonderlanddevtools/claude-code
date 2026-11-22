# Claude Code remaining-surfaces distinct hunt — 2026-11-22

Repo: wonderlanddevtools/claude-code @ `f39ac02a744aea250078e0f402f41e639b4c7548`
Branch: `cursor/bc-4b45f329-4618-4f93-b1ed-1e01a1f22895-967a` (tracking `main`)

## Result: **NONE**

No NEW MEDIUM+ complete attacker→impact chains distinct from already-reported findings.

## Scope (code-traced)

1. `src/utils/settings/types.ts` — every execute/persist/trust-expand key
2. acceptEdits `DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES` gaps vs kickoff candidate list
3. Cron/loop ungating after `2934f30` (removed `feature('AGENT_TRIGGERS')`)
4. LSP plugin command / include directives / ConfigTool
5. WebFetch SSRF residual
6. Session resume / teleport / chrome / oauth residual

## Settings keys

| Key class | Disposition |
|-----------|-------------|
| hooks / statusLine / fileSuggestion / apiKeyHelper / awsAuthRefresh / awsCredentialExport / gcpAuthRefresh / otelHeadersHelper | Already CRITICAL (helpers + hooks hunts) |
| env / permissions.allow / defaultMode / additionalDirectories / sandbox / enableAllProjectMcpServers / enabledMcpjsonServers / extraKnownMarketplaces / enabledPlugins / agent+initialPrompt / outputStyle / cleanupPeriodDays / skipDangerousModePermissionPrompt / autoMemoryDirectory (localSettings) | Already filed |
| pluginConfigs merge→userSettings promotion + project option substitution | Already CRITICAL (source-build / wonderlandfreecode same code) |
| skipWebFetchPreflight | Part of filed WebFetch allow+SSRF finding |
| plansDirectory | Contained to project root (`plans.ts:89-94`) |
| worktree.symlinkDirectories | Traversal-checked; only links paths under repo root |
| channelsEnabled / allowedChannelPlugins | Policy/managed only; project cannot enable |
| sshConfigs | Schema-only in this tree (no consumer outside `types.ts`) |
| forceLoginMethod / forceLoginOrgUUID | Incomplete phishing; no token theft chain |
| companyAnnouncements / claudeMdExcludes | Already MEDIUM |
| autoMode / skipAutoPermissionPrompt / useAutoModeDuringPlan | Explicitly exclude projectSettings |
| ConfigTool | `USER_TYPE === 'ant'` only → dead on external builds |
| defaultShell / attribution / language / spinner* / UI knobs | No exec/trust expansion |

## acceptEdits gaps

Deny lists still only:

```
DANGEROUS_FILES: .gitconfig .gitmodules .bashrc .bash_profile .zshrc .zprofile .profile .ripgreprc .mcp.json .claude.json
DANGEROUS_DIRECTORIES: .git .vscode .idea .claude
```

All kickoff candidates remain AUTO under `isDangerousFilePathToAutoEdit`, but:

| Candidate | Why not NEW |
|-----------|-------------|
| lefthook.yml / .pre-commit-config.yaml / .circleci / Jenkinsfile / azure-pipelines / .gitlab-ci / .buildkite | Already HIGH (lefthook-ci hunt) |
| .husky / .github / .npmrc / .devcontainer / .cursor / package.json / mix.exs / build.sbt / Gemfile / setup.py / Gradle / Makefile / justfile / flake.nix / .envrc / yarn/pnpm / REPL inits / Netlify/Vercel/Wrangler / Dockerfile | Kickoff + prior HIGH filings |
| vite/webpack/jest/eslint/prettier/babel/playwright/… JS configs | Filed as JS config-as-code class |
| Vagrant / Terraform / Ansible / Podfile / Fastfile / .gitpod / .continue / .idx / .pnp.cjs / mise.toml / .lintstagedrc.js / build.zig | Already HIGH |
| .overcommit.yml / .simple-git-hooks.js / Dangerfile / deno.json / fly.toml / render.yaml / Earthfile / Procfile / serverless / Pulumi / Taskfile / mage / goreleaser / heroku app.json | Same delayed toolchain/hook/deploy class as already-filed siblings; prior hunts deferred and did not promote |
| tox.ini / noxfile / conftest / pytest.ini / poetry / Pipfile / hatch / pdm | Same delayed Python-toolchain class as setup.py |
| sitecustomize / usercustomize / .pythonrc | Incomplete without PYTHONPATH/PYTHONSTARTUP (settings.env already CRITICAL) |
| .gitattributes filter / .git/info/exclude / .mailmap / CODEOWNERS | No exec without `.git/config` write (`.git` blocked) |
| CLAUDE.md / AGENTS.md | Prompt-only; symlink secret include already HIGH |

No single remaining filename both (a) executes via a common victim tool and (b) is distinct enough from the filed hook-manager / CI / package-manifest / deploy / IDE classes to justify a new MEDIUM+.

## Cron ungating (`2934f30`)

- `isKairosCronEnabled` = `!CLAUDE_CODE_DISABLE_CRON` only
- REPL `useScheduledTasks` + print.ts scheduler live
- `onFire` → `enqueuePendingNotification` without `skipSlashCommands` → slash skill → `allowed-tools` + inline shell

**Same CRITICAL chain already filed** (`ungated cron auto-invokes skill inline-shell RCE`). No NEW trampoline (no distinct agent/bypassPermissions-only path beyond that skill chain).

## LSP / include / ConfigTool

- LSP: plugins-only (`services/lsp/config.ts:10-12`); no project `settings.lsp`
- CLAUDE.md `@include`: external dialog + in-cwd/symlink already HIGH
- ConfigTool: ant-only

## WebFetch / resume / teleport / chrome / oauth

- WebFetch private-IP + project `WebFetch(*)` allow + `skipWebFetchPreflight`: already HIGH
- Session resume: JSONL data-only; no tool re-exec
- Chrome `/tmp` socket ownership: already HIGH
- OAuth listeners: localhost + state; no NEW code-theft chain
- Teleport/remote: no NEW complete chain beyond prior filings

## Feature-gated dead code (`feature()` always false)

RemoteTrigger, Monitor, SendUserFile, WebBrowser, Workflow, Coordinator, Voice, etc. remain DCE except cron/loop which were hard-ungated in source.
