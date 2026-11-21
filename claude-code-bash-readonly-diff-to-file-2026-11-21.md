# Bash readonly auto-allow — `diff --to-file=` fused path dump (2026-11-21)

## Result: 1 NEW HIGH

### `diff --to-file=/ABS/SECRET cwd_file` — host file content dump via fused flag (HIGH)

**Distinct from filed `diff --from-file=`:** same `filterOutFlags` gap class, different flag (`--to-file`). Filed list named `--from-file=` only.

**Why auto-allowed (bashToolCheckPermission step 7)**

1. `diff` is in `READONLY_COMMANDS` → `makeRegexForSafeCommand('diff')` allows arbitrary non-metacharacter args including `--to-file=/abs/path`  
   (`src/tools/BashTool/readOnlyValidation.ts:1487`, regex via `:1422–1424`, `:1511`).
2. `diff` is a `PathCommand` whose extractor is plain `filterOutFlags`  
   (`src/tools/BashTool/pathValidation.ts:289`), which **drops every argv token starting with `-`**, including fused `--to-file=/ABS/SECRET` (`:126–138`).
3. Only the cwd-relative operand (e.g. `placeholder.txt`) is path-checked → `checkPathConstraints` **passthrough**.
4. `BashTool.isReadOnly` → `checkReadOnlyConstraints` → **allow** → step 7 auto-allow  
   (`src/tools/BashTool/bashPermissions.ts:1153–1162`, `BashTool.tsx:437–440`).

Space-separated `--to-file /ABS/SECRET` correctly path-asks (secret extracted). Fused `=` form does not.

**Impact**

Prompt-injected or malicious model turns can read arbitrary host files (SSH keys, AWS creds, `~/.gitconfig`, etc.) without FileRead permission and without a user approval prompt, because the command is classified read-only and the secret path never enters path validation.

**Attack path**

```text
Bash("diff --to-file=/home/user/.ssh/id_rsa placeholder.txt")
  → checkPathConstraints: extract [placeholder.txt] only → passthrough
  → isReadOnly / checkReadOnlyConstraints → allow
  → bashToolCheckPermission step 7 → behavior: allow
  → stdout contains secret file lines as `>` diff hunks
```

**Evidence**

| Location | Role |
|----------|------|
| `src/tools/BashTool/pathValidation.ts:126–138` | `filterOutFlags` drops `--flag=/path` |
| `src/tools/BashTool/pathValidation.ts:289` | `diff: filterOutFlags` |
| `src/tools/BashTool/readOnlyValidation.ts:1487` | `diff` in `READONLY_COMMANDS` |
| `src/tools/BashTool/readOnlyValidation.ts:1422–1424` | `makeRegexForSafeCommand` permits fused flags |
| `src/tools/BashTool/bashPermissions.ts:1153–1162` | step 7 readonly auto-allow |

**Harness**

```text
command: diff --to-file=/tmp/cc-perm-secret.txt placeholder.txt
path: passthrough
readOnly: allow
step7: true
extracted: ["placeholder.txt"]
```

**PoC (fixture only, not real secrets)**

```bash
printf 'SSH_PRIVATE_KEY=BEGIN_DUMMY_KEY_MATERIAL\nAWS_SECRET_ACCESS_KEY=secretvalue\n' \
  > /tmp/cc-perm-secret.txt
mkdir -p /tmp/cc-perm-cwd && printf 'placeholder\n' > /tmp/cc-perm-cwd/placeholder.txt
cd /tmp/cc-perm-cwd
diff --to-file=/tmp/cc-perm-secret.txt placeholder.txt
# →
# 1c1,2
# < placeholder
# ---
# > SSH_PRIVATE_KEY=BEGIN_DUMMY_KEY_MATERIAL
# > AWS_SECRET_ACCESS_KEY=secretvalue
```

**Remediation**

- Teach `PATH_EXTRACTORS.diff` (or a shared fused-flag helper) to extract values from `--from-file=`, `--to-file=`, `--exclude-from=`, and space-separated forms; **or**
- Remove `diff` from bare `READONLY_COMMANDS` / move it to `COMMAND_ALLOWLIST` that denies `--from-file` / `--to-file` / `--exclude-from`; **or**
- Reject any `diff` argv token matching `--(from|to|exclude-from)-file(=|$)` unless the path passes `validatePath`.

---

## Checked / not NEW validated dump gadgets

| Candidate | Outcome |
|-----------|---------|
| `jq -n --rawfile/--slurpfile/--rawfile=` | PATH_EXTRACTORS skips filename (treats as filter) **but** blocked by jq regex + `validateJqCommand` + AST → `readOnly=passthrough`, not step7 |
| `jq --argfile` | Path-asks on abs FILE (extractor includes it); already filed family |
| `sort --files0-from=` | Dumps content, but **not** in `safeFlags` → readonly reject |
| `sha256sum/md5sum -c/--check` | Auto path-checks checksum file; warn mode does **not** echo line bodies |
| `awk -f=` / `sed --file=` | Extractor drops fused script path, but commands **not** readonly-allowlisted |
| `uniq` with file args | Flags-only regex → reject; flags-only forms don't dump files |
| `cut`/`paste`/`column` + abs path | PathCommand → path **ask** |
| `realpath`/`readlink`/`basename`/`dirname`/`numfmt`/`expr`/`test`/`getconf`/`seq`/`which`/`type` | READONLY + not PathCommand → step7, but **no content dump** |
| `file -f=` | step7 true via allowlist, but GNU short `-f=` treats arg as `=/path` → no read of secret |
| `file --files-from=` | Dumps lines via "cannot open" errors, but `--files-from` **not** in `safeFlags` → not step7 |
| `info -f` / `--file=` | step7 true (not PathCommand); no local `info` binary to prove content dump; prior note that useful dump needs blocked `-o` |
| `stat --reference=` | step7 true on paper; this coreutils 9.4 build has **no** `--reference`; not a content dump |
| `grep -f` / `--file=` | Already filed |
| `diff --from-file=` / `wc --files0-from=` / `base64 --input=` / `file --magic-file=` | Already filed |
