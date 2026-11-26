# acceptEdits remaining auto-exec gaps — `.yarnrc` / GN / DEPS / RoboFile — 2026-11-26

Repo: wonderlanddevtools/claude-code @ main  
Run: https://cursor.com/agents/bc-a263f48e-f240-5608-95e2-9e5150b475a9

## Mechanism (live)
- `DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES`: `src/utils/permissions/filesystem.ts:57-79`
- `isDangerousFilePathToAutoEdit`: `:435-488` (basename + dir segment only)
- `checkPathSafetyForAutoEdit`: `:620-665`
- acceptEdits + in-cwd allow: `:1361-1376`
- FileWriteTool → `checkWritePermissionForTool`: `FileWriteTool.ts:135-141`
- project `permissions.defaultMode` accepted: `permissionSetup.ts:743-772`
- TrustDialog generic: `TrustDialog.tsx:206-209`

Live bun (`NODE_ENV=test`, `mode: acceptEdits`, cwd set):

| path | safety | permission |
|------|--------|------------|
| `.yarnrc` | SAFE | allow (acceptEdits) |
| `scripts/yarn.js` | SAFE | allow |
| `DEPS` | SAFE | allow |
| `BUILD.gn` / `.gn` | SAFE | allow |
| `RoboFile.php` | SAFE | allow |
| `.bashrc` | UNSAFE | ask (safetyCheck) |

## NEW validated HIGH (not in kickoff already-reported list)

### 1. `.yarnrc` (Yarn classic `yarn-path`) — live PoC
Filed list has `.yarnrc.yml` (Berry `yarnPath`) and `.pnp.cjs`, not classic `.yarnrc`.  
Yarn v1 reads `yarn-path` from `.yarnrc` and execs that file as the yarn binary on every `yarn` invocation.

Live PoC (yarn 1.22.22):
```
.yarnrc: yarn-path "./scripts/yarn.js"
yarn install → scripts/yarn.js ran → /tmp/yarnrc_classic_accepteedits_pwned
```

### 2. `DEPS` (gclient hooks) — Chromium/Flutter depot_tools
`gclient sync` / `gclient runhooks` runs `hooks[].action` command lists from DEPS (hooks on by default).  
Modern `gclient_eval` is a restricted AST parse (not full `exec`), but planted `hooks` still spawn arbitrary argv.  
DISTINCT from filed SConstruct/Makefile/Bazel.

### 3. `BUILD.gn` / `.gn` (GN `exec_script` / `action`)
`gn gen` synchronously runs `exec_script()` Python during buildfile evaluation; `action()` runs at ninja build.  
`.gn` can set `script_executable` / `exec_script_allowlist`.  
DISTINCT from filed Bazel/Buck Starlark (`BUILD.bazel` / `BUCK`).

### 4. `RoboFile.php` (consolidation/robo)
Robo `Runner::loadRoboFile` `require_once`s `RoboFile.php`; top-level PHP + task methods run on `robo` / `vendor/bin/robo`.  
DISTINCT from filed `composer.json` scripts (JSON lifecycle; explicitly ruled out as package.json sibling) and Gemfile/Guardfile Ruby.

## FOCUS B
**NONE** new MEDIUM+ complete authz/config-trust chains beyond already-reported bypassPermissions/-p, additionalDirectories home, skipDangerousModePermissionPrompt, settings.local.json, extraKnownMarketplaces + GIT_TEMPLATE_DIR, MCP enableAll, plugin postinstall, SessionStart HTTP, agent triggers cron, WebFetch SSRF, OAuth listeners, bridge, chrome, proxy, CA certs, ANTHROPIC_BASE_URL, skill symlink, config include, LSP command, statusline, swarm, deeplink, pluginConfigs, etc.

## Ruled-out closest (with why)
| candidate | why |
|-----------|-----|
| `.gitattributes` | filter= needs gitconfig `filter.*` (blocked `.gitconfig`); alone not RCE |
| `.mailmap` | no exec |
| Earthfile / Taskfile.yml | Makefile/justfile/Dockerfile delayed siblings (prior deferred) |
| `.buckconfig` / Pants.toml / pleasew / BUILD.plz | config or Bazel/Buck Starlark siblings (BUILD/BUCK filed) |
| tox.ini / Pipfile / poetry.toml / hatch.toml / pdm.toml / pixi.toml / environment.yml | package.json/setup.py install-config class |
| sitecustomize.py / usercustomize.py / `.pythonrc` | needs PYTHONPATH / site-packages / PYTHONSTARTUP (incomplete alone) |
| Capfile / Dangerfile / Thorfile / Berksfile / Cheffile / Kitchen.yml / Policyfile.rb | Fastfile/CI/IaC Ruby siblings (deferred or weaker) |
| netlify/vercel/wrangler already; fly/render/railway/heroku/serverless/cdk/pulumi/sam | deploy siblings |
| skaffold/garden/kustomize/helmfile/CronJob | Tilt/TF/k8s deploy siblings |
| appveyor/drone/cloudbuild/buildkite | non-GH CI class filed |
| `.goreleaser` / electron-builder / tauri.conf.json | release/deploy config; weaker than JS-module loaders |
| forge.config.js / app.config.js / capacitor.config.ts / metro / playwright / cypress / karma | JS config-as-code class (vite/next/jest/eslint filed) |
| composer.json / deno.json | package.json lifecycle / task siblings (composer explicitly prior-ruled) |
| premake5.lua / Build.PL | siblings of just-filed xmake.lua / Makefile.PL |
| `.vimrc` / init.lua / `.emacs` / `.tmux.conf` / config.fish | cwd not auto-loaded without exrc / home path (home addDir already filed) |
| `.ssh/config` / authorized_keys / `.aws/config` / `.docker/config.json` / `.netrc` | cwd writes inert unless env retargets HOME/AWS_CONFIG_FILE/DOCKER_CONFIG (env gadgets filed) |
| `.tool-versions` / volta / `.nvmrc` | version pins; asdf plugins not auto-exec on write |
| simple-git-hooks / yorkie / pinst / commitlint | husky/lefthook/lint-staged class |
| noxfile / SConstruct / Tiltfile / mage / sst / Directory.Build.* / ecosystem.config.js / … | already filed 2026-11-25 hunts |
