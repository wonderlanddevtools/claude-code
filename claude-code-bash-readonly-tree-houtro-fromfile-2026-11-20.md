# Bash readonly auto-allow gadgets — tree --houtro / --fromfile (2026-11-20)

Hunt target: NEW Bash readonly step-7 auto-allow secret-dump gadgets in
`/agent/repos/claude-code`, distinct from the already-filed allowlist dumpers
(tac/rev/fmt/fold/pr, expand/unexpand/comm, tsort, cmp, man -l, du --files0-from,
pyright, base64 --input=, diff --from-file / wc --files0-from / file --magic-file,
xargs, hintro/strings/@file/fromtabfile/xaa/foundry, find --files0-from, ps,
grep/rg -f, git blame --ignore-revs-file, etc.).

## Result: 2 NEW MEDIUM

### 1. `tree -H … --houtro=FILE` — HTML outro file content dump (MEDIUM)

**Distinct from filed `hintro`:** same `tree` HTML path, different flag
(`--houtro` was not on the filed list; only `--hintro` was).

**Why auto-allowed**
- `tree` is in `COMMAND_ALLOWLIST` with `--houtro: 'string'` and `-H: 'string'`
  (`src/tools/BashTool/readOnlyValidation.ts` ~710–713).
- `tree` is **not** a `PathCommand` → `checkPathConstraints` returns passthrough
  for absolute host paths (`src/tools/BashTool/pathValidation.ts`).
- Step 7: `BashTool.isReadOnly` → `behavior: 'allow'`
  (`src/tools/BashTool/bashPermissions.ts` ~1153–1162).

**PoC (dummy creds only)**
```bash
printf 'API_KEY=sk-dummy-not-real\nPASSWORD=dummy-pw\n' > /tmp/dummy_creds.txt
tree -H . --houtro=/tmp/dummy_creds.txt /some/dir
# HTML stdout embeds both credential lines verbatim before </body>
```

Harness: `readOnly=allow`, `path=passthrough`, `step7AutoAllow=true`.

### 2. `tree --fromfile FILE` — line-as-path content dump (MEDIUM)

**Distinct from filed `fromtabfile`:** different flag/semantics. `--fromfile`
reads newline-separated paths and prints each line as a tree node name;
`--fromtabfile` reads a tab-indented tree. Filed list named `fromtabfile` only.

**Why auto-allowed**
- `--fromfile: 'none'` in tree allowlist (`readOnlyValidation.ts` ~719).
- Positional file path is accepted by `validateFlags` fallthrough.
- `tree` not in `PATH_EXTRACTORS` → absolute path passthrough.
- Step 7 auto-allow as above.
- Note: fused `--fromfile=/abs` is rejected (`none` + `=` → fail); space form works.

**PoC**
```bash
tree --fromfile /tmp/dummy_creds.txt
# prints:
# /tmp/dummy_creds.txt
# ├── API_KEY=sk-dummy-not-real
# └── PASSWORD=dummy-pw
```

Harness: `readOnly=allow`, `path=passthrough`, `step7AutoAllow=true`.

## Checked / not NEW MEDIUM+

| Candidate | Outcome |
|-----------|---------|
| less/more/openssl enc -in/iconv/recode/sqlite3 .read/look/join/tee/xxd | **Not** in READONLY/COMMAND_ALLOWLIST → step7 false |
| info -f/--file | Auto-allow, but useful dump needs `-o/--output` which is **blocked** |
| nl/od/hexdump/column/cut/paste/sort + abs path | PathCommand → path **ask** (not step7) |
| awk/sed/uniq abs | Not readonly and/or path ask |
| jq --argfile | Path ask on abs file; `--slurpfile` blocked by jq regex |
| pgrep -F / date -r / fd --ignore-file / numfmt / readlink | Auto-allow but **no content dump** |
| tree --infofile / --gitfile | Auto-allow, no arbitrary content dump |
| man /abs (without -l) | Same family as filed `man -l` — not re-reported |
| Remaining fused PathCommand dumps | Already filed (base64 --input, wc --files0-from, file --magic-file, grep -f, …) |

## Suggested fix

Remove `--houtro` / `--hintro` / `--fromfile` / `--fromtabfile` (and other
file-bodied tree flags) from `tree.safeFlags`, or add `tree` to `PATH_EXTRACTORS`
with extractors for those flag values and require paths inside the working tree.
