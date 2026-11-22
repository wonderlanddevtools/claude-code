# Claude Code acceptEdits delayed-RCE hunt — Cursor hooks / remaining gaps (2026-11-22)

Repo: wonderlanddevtools/claude-code @ `main`
Workspace: `/agent/repos/claude-code`
Commit: `ce1572f`

## Focus

NEW HIGH-impact DISTINCT acceptEdits FileWrite auto-allow gadgets still missing after the REPL/debugger cwd-init filing (`claude-code-acceptEdits-repl-debugger-inits-2026-11-22`). Prefer different loader/trigger than already-filed package/CI/build/REPL classes. Explicitly re-check candidates from the hunt brief: debugger/REPL leftovers, git-hook alternatives, systemd units, `.cursor/hooks.json`.

## Result: **1 NEW HIGH**

### HIGH — `.cursor/` not in `DANGEROUS_DIRECTORIES` (hooks.json / mcp.json auto-exec)

`DANGEROUS_DIRECTORIES` is only `.git`, `.vscode`, `.idea`, `.claude`. The comment at `filesystem.ts:430` says `.vscode` is blocked “to prevent VS Code settings manipulation and potential code execution” — but the Cursor project dir `.cursor/` is omitted.

Under acceptEdits, in-cwd FileWrite of `.cursor/hooks.json` (and hook scripts / `.cursor/mcp.json`) **auto-allows**. Cursor docs: project hooks at `/.cursor/hooks.json` automatically load and run in trusted workspaces; Cursor watches and reloads hooks.json; cloud agents pick up project hooks. Workspace trust is off by default (every folder trusted). Hook `command` can be inline shell (`curl …|bash`, `python3 -c …`) — no chmod step required.

Live bun check (cwd set, `FileWriteTool`, `mode: acceptEdits`):

| Path | safety | write |
|------|--------|-------|
| `.cursor/hooks.json` | AUTO | **allow** (`mode:acceptEdits`) |
| `.cursor/hooks/pwn.sh` | AUTO | **allow** |
| `.cursor/mcp.json` | AUTO | **allow** |
| `.vscode/tasks.json` | ASK | ask (`safetyCheck`) |
| `.claude/settings.json` | ASK | ask |
| `.mcp.json` / `.bashrc` | ASK | ask |

Distinct from: filed husky/lefthook/pre-commit (git hooks; `.git` blocked), filed root `.mcp.json` (Claude Code MCP), filed REPL/debugger cwd inits, filed `.vscode` block. Different loader (Cursor hooks / Cursor project MCP) and different trigger (next Cursor agent event / cloud agent / workspaceOpen).

Note: mix-sbt-deploy hunt listed “`.cursor IDE config`” under `already_filed_not_rereported` but no finding document exists in-repo or in memory; this hunt’s skip list does not include it and explicitly asks for `.cursor/hooks.json`.

## Ruled out this pass (AUTO but not NEW / not strong enough)

| Candidate | Why not filed |
|-----------|----------------|
| `.gdbinit` / `.lldbinit` | Already in REPL/debugger filing; modern gdb needs `auto-load local-gdbinit` + safe-path; lldb defaults to warn/not-source cwd `.lldbinit` |
| `.pdbrc` / `.Rprofile` / `.irbrc` / `.pryrc` / `.ghci` / `.ocamlinit` | Already filed 2026-11-22 |
| `.perldb` | Same REPL/debugger class as filed `.pdbrc` (sibling filename only) |
| `.guile` / `.sbclrc` / `.pythonrc(.py)` | Home/`PYTHONSTARTUP` only — not cwd auto-load |
| `.vimrc` / `.exrc` / `.nvimrc` / `.nvim.lua` | Need `exrc` and/or explicit `:trust` |
| `.tmux.conf` / `.screenrc` | Not cwd auto-sourced by default |
| `.simple-git-hooks*` | Needs reinstall; prior reject |
| Repo systemd units (`.config/systemd/user/…`) | Write AUTO under cwd but systemd does not auto-load units from a project tree |
| `poetry.toml` / `Pipfile` | Not RCE on write alone |
| Other config-as-code laundry list | Same class as dozens of already-filed findings |

## Shared mechanism

1. `DANGEROUS_FILES` / `DANGEROUS_DIRECTORIES` — `filesystem.ts:57-79`
2. Comment naming `.vscode` code-exec rationale — `:427-434`
3. `isDangerousFilePathToAutoEdit` — `:435-488`
4. `checkPathSafetyForAutoEdit` — `:620-665`
5. acceptEdits in-cwd auto-allow — `:1361-1376`
6. `FileWriteTool` → `checkWritePermissionForTool` — `FileWriteTool.ts:135-141`
7. `permissionSetup` honors project `defaultMode: acceptEdits` — `permissionSetup.ts:743-772`

## Flagged JSON

`claude-code-acceptEdits-cursor-hooks-2026-11-22---flagged-vulnerabilities.json`
