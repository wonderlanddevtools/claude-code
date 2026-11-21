# Claude Code acceptEdits delayed-RCE hunt — 2026-11-21

Repo: wonderlanddevtools/claude-code @ `main`
Workspace: `/agent/repos/claude-code`

## Focus

- **A**: acceptEdits FileWrite auto-allow of files a typical victim later executes (beyond already-filed DANGEROUS_* gaps)
- **B**: other authZ/logic — only if a NEW distinct complete chain exists

## Result: **3 NEW HIGH** (Focus A); Focus B **NONE**

Shared mechanism (all three):

1. `DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES` — `filesystem.ts:57-79` (shell/git/mcp basenames + `.git/.vscode/.idea/.claude` only)
2. `checkPathSafetyForAutoEdit` — `:620-665`
3. acceptEdits in-cwd auto-allow after safety — `:1361-1376`
4. `FileWriteTool` / `FileEditTool` → `checkWritePermissionForTool`

Live basename/dir check: all reported paths **AUTO**; controls `.mcp.json`, `.bashrc`, `.git/hooks/*` **ASK**.

### 1. HIGH — `mix.exs` / Elixir Mix config

Silent acceptEdits write of `mix.exs` or `config/runtime.exs|config.exs|dev.exs` → victim `mix deps.get|compile|test` (or release start) evaluates Elixir → RCE.

Distinct from Gemfile / setup.py / Gradle+Maven filings.

### 2. HIGH — `build.sbt`

Silent write → victim `sbt compile|test|run` evaluates Scala build DSL → RCE.

Distinct from filed Gradle/Maven JVM entrypoints.

### 3. HIGH — `netlify.toml` / `vercel.json` / `wrangler.toml`

Silent write of hosting build commands → victim `netlify`/`vercel`/`wrangler` CLI or platform deploy → RCE.

Distinct from filed `.github/workflows` + non-GH CI (Jenkins/GitLab/Circle/…).

## Focus B

No NEW distinct complete chain on hooks/skills/slash/MCP/LSP/SessionStart/cron/WebFetch/OAuth/updater/include/CLAUDE.md/plugins beyond already-filed set.

## Deferred (AUTO but not filed this pass)

`.overcommit.yml`, `.simple-git-hooks.js`, `Dangerfile`, `mise.toml`, `deno.json`, `build.zig`, `fly.toml`, `render.yaml` — valid gaps; kept out to prefer strongest 1–3 / avoid hook-manager and deploy laundry lists.

## Flagged JSON

`claude-code-acceptEdits-mix-sbt-deploy-2026-11-21---flagged-vulnerabilities.json`
