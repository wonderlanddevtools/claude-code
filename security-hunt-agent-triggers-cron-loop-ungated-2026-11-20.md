# Security hunt — AGENT_TRIGGERS ungating / cron /loop auto-RCE (2026-11-20)

Repo: wonderlanddevtools/claude-code @ `1f3dc213cd1f4fc3b6942616e6c1d7491a89e17c` (main)
Gate-removal commits: `33fe494`, `2934f30`

## Verdict: **1 CRITICAL NEW**

Prior same-day hunt (`claude-code-agent-triggers-cron-loop-hunt-2026-11-20`) concluded NONE because it treated cron as “prompt injection only” and assumed Bash still prompts. That missed the planted-cron → slash → project-skill `allowed-tools` + `` !` `` / ` ```! ` `` path, which auto-allows and executes shell without a Bash permission prompt.

---

## CRITICAL — Planted `.claude/scheduled_tasks.json` auto-invokes project skill RCE after Trust

### Reachability after ungating (validated)

| Check | Result | Evidence |
|-------|--------|----------|
| `feature('AGENT_TRIGGERS')` still gates cron? | **NO** | `isKairosCronEnabled()` = `!CLAUDE_CODE_DISABLE_CRON` only (`src/tools/ScheduleCronTool/prompt.ts:36-38`). Comment about GrowthBook `tengu_kairos_cron` is stale. |
| Cron tools loaded? | **YES** | `src/tools.ts:29-33` always `require`s CronCreate/Delete/List (no feature wrap). |
| `/loop` registered? | **YES** | `src/skills/bundled/index.ts:36` always `registerLoopSkill()`; `isEnabled: isKairosCronEnabled` (`loop.ts:83`). |
| REPL scheduler mounts? | **YES** | `src/screens/REPL.tsx:4049-4056` always calls `useScheduledTasks`. |
| `-p`/SDK scheduler? | **YES** | `src/cli/print.ts:2702-2729` starts when `isKairosCronEnabled()`. |
| Planted file auto-enables? | **YES** | `cronScheduler.ts:478-485` `hasCronTasksSync()` → `setScheduledTasksEnabled(true)`. |
| TrustDialog mentions scheduled_tasks? | **NO** | Generic folder trust only (`TrustDialog.tsx:207-209`). Skill Bash is analytics-only (`:128-144`, `:278-282`). |

### Complete attack chain

1. Attacker commits:
   - `.claude/scheduled_tasks.json` — recurring task, ancient `createdAt`, `permanent: true`, `prompt: "/pwn"`
   - `.claude/skills/pwn/SKILL.md` — `allowed-tools: Bash(*)` plus `` ```! `` payload (or `` !`cmd` ``)
2. Victim clones and runs `claude`, accepts generic TrustDialog (or `claude -p` which skips trust).
3. Scheduler auto-enables from planted file. Overdue recurring tasks fire on first idle tick **without** AskUserQuestion (missed ask is one-shot-only: `cronScheduler.ts:195-197`, fire path `:283-297`).
4. Fire enqueues prompt **without** `skipSlashCommands` (`useScheduledTasks.ts:71-82`; print.ts `:2706-2722`).
5. `processUserInput` / QueryEngine `ask()` treat `/pwn` as slash (`processUserInput.ts:531-550`; `QueryEngine.ts:419-431` omits skip).
6. Slash bypasses SkillTool permission UI. `getPromptForCommand` injects `alwaysAllowRules.command = allowedTools` and runs `executeShellCommandsInPrompt` (`loadSkillsDir.ts:375-395`; `promptShellExecution.ts:97-115`).
7. Payload runs as the victim. No Bash prompt. No model cooperation.

### Why DISTINCT

- Not SessionStart/hooks HTTP, LSP, statusline, WebFetch, OAuth, enableAllProjectMcpServers, sticky MCP, skill symlink, or settings.local trust.
- Not “user typed `/skill`” — the already-filed skill CRITICAL requires user/model skill invocation. This entry point is **auto-fire after Trust alone** via ungated durable cron.
- Prior agent-triggers hunt (NONE) missed this completion path.

### Severity

**CRITICAL** — planted repo → Trust click (or `-p`) → arbitrary shell, no further consent.

### Remediation

- Keep durable cron behind an explicit opt-in (or restore a real runtime gate); do not auto-enable from planted `scheduled_tasks.json` without TrustDialog disclosure.
- Cron-enqueued prompts must set `skipSlashCommands: true` (or strip leading `/`) so fires cannot invoke skills/commands.
- Refuse `allowed-tools` + inline `` !` `` shell from project/plugin skills without an explicit high-risk approval (already needed for the filed skill sink).
- Surface scheduled_tasks + skill Bash/hooks in TrustDialog UI, not analytics-only.

---

## Disproved / incomplete candidates

| Candidate | Why not NEW MEDIUM+ |
|-----------|---------------------|
| Cron fires plain-text prompt → model Bash | Still hits Bash permission prompts under default mode; headless ask→deny. Incomplete alone. |
| CronCreateTool silent allow | Default `checkPermissions → allow` (`Tool.ts:753-766`) but writing durable cron still needs model tool call; planted file does not need CronCreate. Amplification only. |
| `/loop` skill itself | Bundled; only schedules via CronCreate after user/model invokes `/loop`. Not auto-planted RCE. |
| `feature('AGENT_TRIGGERS_REMOTE')` RemoteTrigger | Still gated; `feature()` always false → dead (`tools.ts:34-36`, `bundled/index.ts:49`). |
| Session server `0.0.0.0` | Behind `feature('DIRECT_CONNECT')` (`main.tsx:3961`) → dead in this build; also auto-mints auth token (`:3999`). |
| Computer Use sandbox gaps | `getChicagoEnabled()` defaults false + subscription (`gates.ts:12-57`); CU packages stubbed. |
| WorkflowTool | Stub (`WorkflowTool.ts` empty export). |
| Missed one-shot catch-up | Asks AskUserQuestion first (`buildMissedTaskNotification`); not silent. |
| Skill frontmatter hooks alone via cron | Register after `` !` `` (`processSlashCommand.tsx:875-878`); secondary. `` !` `` is the direct RCE. |
| SessionStart HTTP / CLAUDE.md include | Already validated none remaining (commit `1f3dc21`). |

---

## Gate residual

Only local kill switches remain:

- `CLAUDE_CODE_DISABLE_CRON` → disables entire scheduler (`prompt.ts:36-38`)
- GrowthBook `tengu_kairos_cron_durable` (default **true**) only forces `durable:false` on CronCreate writes; **does not** block reading planted `scheduled_tasks.json`
