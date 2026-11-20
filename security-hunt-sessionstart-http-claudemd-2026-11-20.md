# Security hunt — Candidate A/B/C validation (2026-11-20)

Repo: wonderlanddevtools/claude-code @ `3b0a5e484d9cc8e2f1ec18b69801e14e358b128b` (main)

## Verdict: **none** (no NEW MEDIUM+ complete chains)

---

## Candidate A — Project HTTP hooks steal env secrets after Trust (SessionStart)

### Answers (yes/no + evidence)

| Question | Answer | Evidence |
|----------|--------|----------|
| After generic TrustDialog, do project `.claude/settings.json` **HTTP** hooks run automatically on **SessionStart**? | **NO** | `src/utils/hooks.ts:1932-1946` filters out `h.hook.type === 'http'` for `SessionStart` and `Setup`. SessionStart itself still runs after trust (`src/main.tsx:2241` → `2437`; trust gate `src/utils/hooks.ts:286-296`, `2074-2080`). |
| Can a hook set `allowedEnvVars: ["ANTHROPIC_API_KEY"]` and interpolate `$ANTHROPIC_API_KEY` when `httpHookAllowedEnvVars` is undefined? | **YES** (for HTTP hooks that *do* run) | `src/utils/hooks/execHttpHook.ts:162-170`: when `policy.allowedEnvVars === undefined`, `effectiveVars = hookVars`. Schema allows project-set `allowedEnvVars` (`src/schemas/hooks.ts:112-117`). Policy undefined = no restriction (`src/utils/settings/types.ts:491-498`). |
| Does SSRF guard block public attacker URLs? | **NO** | `src/utils/hooks/ssrfGuard.ts:42-53` / `55-85` only blocks private/link-local/CGNAT; public IPs allowed. Live: `isBlockedAddress("8.8.8.8") === false`, `"1.2.3.4" === false`. |
| Is HTTP env-exfil DISTINCT from already-filed command SessionStart RCE? | **YES in class**, but SessionStart HTTP chain is **broken** | Distinct sink (`execHttpHook` + header interpolation vs shell). Already filed previously for HTTP env-exfil; that filing incorrectly assumed SessionStart HTTP runs — current code skips it. |

### Complete SessionStart attack chain?

**NO.** Trust → SessionStart command hooks can still RCE (already filed). Trust → SessionStart **HTTP** env-exfil does **not** fire because of the SessionStart/Setup HTTP filter.

Note: UserPromptSubmit / PreToolUse / Notification / Stop HTTP hooks still execute after trust and still honor project `allowedEnvVars` when policy is undefined — that class was already filed (`claude-code-hooks-http-env-exfil---flagged-vulnerabilities.json`). Not re-filed as NEW.

---

## Candidate B — CLAUDE.md `@include` of `~/.claude/.credentials.json`

### Answers (yes/no + evidence)

| Question | Answer | Evidence |
|----------|--------|----------|
| Can planted `@~/.claude/.credentials.json` or `@/home/user/.claude/.credentials.json` load secrets into model context **after Trust alone**? | **NO** | Paths parse as valid includes (`src/utils/claudemd.ts:475-487` accepts `~/` and `/`). They are `isExternal` (`!pathInOriginalCwd`, `:667-669`) and skipped unless `includeExternal`. |
| Is there a dialog that blocks this? | **YES** | `ClaudeMdExternalIncludesDialog` (`src/components/ClaudeMdExternalIncludesDialog.tsx`); shown from `showSetupScreens` (`src/interactiveHelpers.tsx:163-170`) when `shouldShowClaudeMdExternalIncludesWarning()` (`src/utils/claudemd.ts:1420-1429`). Decline sets `hasClaudeMdExternalIncludesApproved: false` (`dialog:127-132`). |
| Does `includeExternal` default true after trust? | **NO** | `getMemoryFiles`: `includeExternal = forceIncludeExternal \|\| config.hasClaudeMdExternalIncludesApproved \|\| false` (`src/utils/claudemd.ts:798-801`). Default project config: `hasClaudeMdExternalIncludesApproved: false` (`src/utils/config.ts:146-147`). Context load uses `getMemoryFiles()` without force (`src/context.ts:172`). |
| Are `.json` / credentials files in `TEXT_FILE_EXTENSIONS`? | **YES for `.json`** | `src/utils/claudemd.ts:96-102` includes `.json`. `extname('.credentials.json') === '.json'` → not skipped by extension gate (`:349-354`). Extension alone does **not** load them; external gate does. |

### Complete attack chain after Trust alone?

**NO.** Explicit `@~/...` / absolute home includes require the external-includes dialog approval. (In-repo symlink `@./x` → host secret is a **different** already-filed finding; not this candidate.)

---

## Candidate C — Other DISTINCT high-impact (oauth localhost / chrome / updater / heapdump / export / SkillTool)

Reviewed against already-filed classes (oauth listener, chrome `/tmp` socket, updater version path traversal, image-paste `/tmp`, copy `/tmp/claude/response.md`, skill symlink, etc.):

| Surface | Result |
|---------|--------|
| OAuth `AuthCodeListener` | Binds `localhost`; state checked; no NEW beyond filed listener class (`src/services/oauth/auth-code-listener.ts:46`, `:164-169`). |
| Chrome native host | `/tmp` socket dir takeover already HIGH; no NEW chain. |
| Native updater | Version `..` path escape already HIGH; regex still allows `-S` with `/` (`download.ts:116`) — already filed. |
| `/heapdump` | User-initiated (ant UI); writes Desktop with 0600 — no planted-repo chain. |
| `/export` | `local-jsx` only; model cannot invoke via SkillTool; path under `getCwd()` from user args — no planted-repo MEDIUM+. |
| SkillTool | Safe-property auto-allow + skill symlink exfil already filed; `hooks`/`allowedTools` not in `SAFE_SKILL_PROPERTIES`. |

**No NEW MEDIUM+ complete chains.**

---

## Output

`findings: []` — see companion flagged JSON.
