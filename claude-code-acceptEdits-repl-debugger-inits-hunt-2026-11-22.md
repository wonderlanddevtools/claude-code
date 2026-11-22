# Claude Code acceptEdits delayed-RCE hunt — REPL/debugger cwd inits (2026-11-22)

Repo: wonderlanddevtools/claude-code @ `main`
Workspace: `/agent/repos/claude-code`

## Focus

Remaining acceptEdits FileWrite auto-allow gadgets that a normal later victim action executes as code — preferring local package-manager / hook / REPL inits over CI laundry lists. Explicitly excluding already-reported paths (package.json, JS tool configs, husky/lefthook/pre-commit, Makefile/justfile/Taskfile/flake/nix, .cargo, Gemfile/Podfile/Fastfile, CMake/meson, JVM, mix/sbt, pnp/mise/lint-staged/zig, Vagrant/TF/Ansible, netlify/vercel/wrangler, gitpod/continue/idx, Dockerfile/compose, .npmrc, shell rcs, .mcp.json, etc.).

## Result: **1 NEW HIGH**

### HIGH — Language REPL / debugger cwd init files (shell-rc parallels)

`DANGEROUS_FILES` intentionally blocks `.bashrc` / `.zshrc` / `.profile` / `.bash_profile` / `.zprofile` (shell startup code). It does **not** block the same threat class for language REPLs and debuggers that auto-source cwd init files:

| File | Victim action | Execution |
|------|---------------|-----------|
| `.pdbrc` | `python -m pdb …` / `pdb.set_trace()` | Python statements (stdlib reads `./.pdbrc`) |
| `.Rprofile` / `.Renviron` | `R` / `Rscript` (without `--vanilla` / `--no-init-file`) | R code / env |
| `.irbrc` / `.pryrc` | `irb` / `pry` | Ruby |
| `.ghci` | `ghci` | GHCi commands (`:! …`) |
| `.ocamlinit` | `ocaml` toplevel | OCaml |
| `.gdbinit` / `.lldbinit` | `gdb` / `lldb` in cwd | debugger commands (`shell …`) — gdb may also require auto-load safe-path |

Live bun `checkPathSafetyForAutoEdit`: all of the above **AUTO**; controls `.bashrc` / `.mcp.json` / `.git/config` **ASK**.

Live PoC: planted `.pdbrc` with `os.system("touch …")` fired under `python3 -m pdb t.py` (cwd init). Stdlib `pdb.Pdb.__init__` comments `# Read ~/.pdbrc and ./.pdbrc` and opens `".pdbrc"`.

Distinct from filed Gemfile/setup.py/JS-config/Makefile classes: trigger is REPL/debugger **session start**, not install/build — and these are the missing siblings of the already-blocked shell rc list.

## Ruled out this pass (AUTO but not NEW / not strong enough)

- justfile / Taskfile / Earthfile / flake.nix / shell.nix / .cargo / build.rs / CMake / meson / composer.json / Podfile / Fastfile / Package.swift — already filed or prior-rejected siblings
- .pre-commit-config.yaml / non-GH CI / Jenkinsfile / etc. — already filed
- netlify/vercel/wrangler — already filed; fly.toml/render.yaml/railway — deferred deploy siblings
- playwright/cypress/vitest/webpack/rollup/astro/svelte/nuxt/gatsby — covered by filed vite/jest/eslint JS-config class
- .simple-git-hooks.js — needs reinstall (prior reject)
- .tool-versions / .nvmrc / .python-version / rust-toolchain — version pins, not code exec
- .inputrc / .tmux.conf / .screenrc / .mailcap / .wgetrc / .curlrc — weak or non-cwd auto-load
- .vimrc / .exrc — needs `exrc` (off by default on modern vim)
- .env / credentials.json — secret overwrite, not RCE
- pixi/conda/tox/nox — setup.py-class prior reject

## Shared mechanism

1. `DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES` — `filesystem.ts:57-79`
2. `isDangerousFilePathToAutoEdit` — `:435-488`
3. `checkPathSafetyForAutoEdit` — `:620-665`
4. acceptEdits in-cwd auto-allow — `:1361-1376`
5. `FileWriteTool` / `FileEditTool` → `checkWritePermissionForTool`
6. `permissionSetup` honors project `defaultMode: acceptEdits` — `:743-772`

## Flagged JSON

`claude-code-acceptEdits-repl-debugger-inits-2026-11-22---flagged-vulnerabilities.json`
