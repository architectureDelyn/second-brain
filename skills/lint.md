# Skill: `/second-brain lint`

This skill implements the `lint` health-check for the second-brain plugin. When a user runs `/second-brain lint` or `/second-brain lint --fix`, follow every step in this document precisely and in order. Execute all checks silently, collect all findings, then output the report. Do not interrupt with questions.

---

## Overview

The lint pipeline audits the brain for structural issues by:
1. Loading the brain config and resolving paths
2. Running all health checks (collect findings — do not stop on first issue)
3. Generating the health report
4. Optionally applying auto-fixes (only when `--fix` is passed)

---

## Step 1 — Load Config

Read `second-brain-config.json` from the brain storage path.

Try both locations using the Read tool:
- `.second-brain/second-brain-config.json` (relative to current working directory)
- `~/.second-brain/second-brain-config.json` (user home directory)

**Config resolution rule**: If both files exist, prefer the project-local one (`.second-brain/`). Use the first one that exists when only one is present. If neither exists, stop immediately and output:

```
Error: No second-brain found. Run `/second-brain init` first.
```

From the config, extract and store:
- `storage_path` — the root directory of the brain
- `name` — the brain name (for report header)
- `schema.stale_threshold_days` — integer, days before a page is stale (default: `90` if missing)
- `schema.max_page_words` — integer, word count limit per page (default: `2000` if missing)

All file paths in subsequent steps are relative to `storage_path` (expand `~` to the actual home directory when constructing shell paths).

Detect the `--fix` flag: if the user's command was `/second-brain lint --fix`, store `FIX_MODE = true`. Otherwise `FIX_MODE = false`.

---

## Step 2 — Inventory Brain Files

Before running checks, collect the full file inventory. This data is reused across multiple checks.

### 2a — List page files

Use the Bash tool:

```bash
ls "<STORAGE_PATH>/pages/" 2>/dev/null | grep '\.md$' | sort
```

Store the result as `PAGE_FILES` — a list of `.md` filenames (basename only, e.g., `transformer-architecture.md`). Exclude `.gitkeep` and any non-`.md` files.

If the `pages/` directory does not exist or is empty (no `.md` files), store `PAGE_FILES = []`.

### 2b — Read index.md

Read `<STORAGE_PATH>/index.md` using the Read tool. If the file does not exist, treat `INDEX_ENTRIES = []` and record a WARNING: `index-missing: index.md not found`.

Parse the catalog table rows. Each data row has the format:
```
| [[page-filename]] | summary text | tag1, tag2 | YYYY-MM-DD |
```

Extract the page filename from inside `[[...]]` in column 1 (strip `[[` and `]]`, do not add `.md`). Store the list of referenced page names (without `.md`) as `INDEX_ENTRIES`.

Skip the header row (`| Page | Summary | Tags | Updated |`) and the separator row (`|------|...|`).

### 2c — Read log.md

Read `<STORAGE_PATH>/log.md` using the Read tool. If the file does not exist, store `LOG_LINES = []` and `LAST_INGEST = "never"`.

Extract:
- `LOG_LINES` — all lines starting with `## [` (the log entry headers), in file order
- `LAST_INGEST` — the date from the most recent `## [YYYY-MM-DD] ingest |` entry. If no ingest entry exists, use `"never"`.

### 2d — Get current date

```bash
date -u +"%Y-%m-%d"
```

Store as `TODAY` (YYYY-MM-DD). Use this for stale threshold calculations.

---

## Step 3 — Run All Health Checks

Run every check below. For each issue found, append a finding object to `FINDINGS`:

```
{ severity: "ERROR"|"WARNING"|"INFO", code: "<check-code>", message: "<human message>" }
```

Collect ALL findings across ALL checks before proceeding to Step 4. Do not stop early.

### Check A — Orphan Pages (WARNING)

**Definition**: A page file in `pages/` that is not referenced in `index.md`.

For each filename in `PAGE_FILES`:
- Derive the stem (filename without `.md`, e.g., `transformer-architecture`)
- Check whether the stem appears in `INDEX_ENTRIES`
- If NOT found: add finding:
  ```
  { severity: "WARNING", code: "orphan-page", message: "pages/<filename> not in index.md" }
  ```

### Check B — Broken Wiki-Links (ERROR)

**Definition**: A `[[wiki-link]]` inside a page file that refers to a page that does not exist in `pages/`.

For each file in `PAGE_FILES`:
1. Read the full file content using the Read tool
2. Extract all `[[...]]` occurrences from the content (anywhere in the file, including inside the frontmatter block's content area and the body). Use this Bash command to extract them:
   ```bash
   grep -n '\[\[' "<STORAGE_PATH>/pages/<filename>" | grep -oP '\[\[\K[^\]]+(?=\]\])'
   ```
   Alternatively, parse the file content directly: find all substrings matching `\[\[` ... `\]\]`.
3. For each extracted link target:
   - Normalize: trim whitespace, strip any `|display-text` alias suffix (e.g., `[[page|alias]]` → `page`)
   - Derive the expected filename: `<link-target>.md`
   - Check whether `<link-target>.md` exists in `PAGE_FILES`
   - If NOT found: add finding:
     ```
     { severity: "ERROR", code: "broken-link", message: "[[<link-target>]] referenced in pages/<filename> (line <N>)" }
     ```
   - Include the line number `<N>` in the message. If line number cannot be determined, use `line unknown`.

To extract wiki-links with line numbers efficiently, use:
```bash
grep -n '\[\[' "<STORAGE_PATH>/pages/<filename>"
```
Then parse the matching lines to extract link targets and their line numbers.

### Check C — Stale Pages (INFO)

**Definition**: A page whose `updated` frontmatter date is more than `stale_threshold_days` old relative to `TODAY`.

For each file in `PAGE_FILES` (using content already read in Check B where available — do not re-read):
1. Parse the YAML frontmatter block (between the opening `---` and closing `---`)
2. Extract the `updated` field value (format: `YYYY-MM-DD`)
3. If the `updated` field is present and parseable:
   - Compute `days_since_update` = (`TODAY` - `updated`) in days
   - If `days_since_update` > `stale_threshold_days`: add finding:
     ```
     { severity: "INFO", code: "stale", message: "pages/<filename> not updated in <days_since_update> days (threshold: <stale_threshold_days> days)" }
     ```
4. If the `updated` field is missing or unparseable, skip the stale check for this page (missing-frontmatter is reported in Check E).

To compute days between two YYYY-MM-DD dates, use the Bash tool:
```bash
echo $(( ( $(date -d "<TODAY>" +%s 2>/dev/null || date -j -f "%Y-%m-%d" "<TODAY>" +%s) - $(date -d "<updated>" +%s 2>/dev/null || date -j -f "%Y-%m-%d" "<updated>" +%s) ) / 86400 ))
```

On macOS, prefer `date -j -f "%Y-%m-%d"`. On Linux, prefer `date -d`. To detect the platform, check `uname -s` once and reuse.

Alternatively, compute the day difference using arithmetic on the date strings directly (year * 365 + month * 30 + day approximation) — this avoids shell date portability issues and is accurate enough for threshold comparisons.

### Check D — Oversized Pages (WARNING)

**Definition**: A page whose word count exceeds `max_page_words`.

For each file in `PAGE_FILES`:
1. Count the words in the full file (including frontmatter, as frontmatter words are negligible):
   ```bash
   wc -w < "<STORAGE_PATH>/pages/<filename>"
   ```
2. If `word_count` > `max_page_words`: add finding:
   ```
   { severity: "WARNING", code: "oversized", message: "pages/<filename> has <word_count> words (limit: <max_page_words>)" }
   ```

To batch word counts efficiently for multiple files:
```bash
wc -w "<STORAGE_PATH>/pages/"*.md 2>/dev/null
```
This outputs one line per file plus a total; parse the per-file lines.

### Check E — Missing Frontmatter (ERROR)

**Definition**: A page that lacks one or more required YAML frontmatter fields.

Required fields: `title`, `tags`, `created`, `updated`, `sources`

For each file in `PAGE_FILES` (using content already read where available):
1. Check whether the file starts with `---` (opening frontmatter delimiter)
2. If NO opening `---`: add finding:
   ```
   { severity: "ERROR", code: "missing-frontmatter", message: "pages/<filename> has no YAML frontmatter block" }
   ```
   Skip further frontmatter checks for this file.
3. If frontmatter block is present, extract the content between the first `---` and the next `---`
4. For each required field (`title`, `tags`, `created`, `updated`, `sources`):
   - Check whether the field key is present in the frontmatter block (simple line-prefix match: look for a line starting with `<field>:`)
   - If a required field is missing: add finding:
     ```
     { severity: "ERROR", code: "missing-frontmatter", message: "pages/<filename> lacks `<field>` field" }
     ```

Note: a line like `title:` (key present but value empty) passes Check E (field is present) but will be caught by Check F (empty required field value).

### Check F — Invalid Frontmatter (ERROR)

**Definition**: A page whose YAML frontmatter is syntactically invalid or has empty required field values.

For each file in `PAGE_FILES` (using content already read where available):
1. If the file has no frontmatter (caught by Check E), skip this check.
2. Check for YAML structural issues:
   - Unclosed frontmatter: file starts with `---` but has no second `---` delimiter → add finding:
     ```
     { severity: "ERROR", code: "invalid-frontmatter", message: "pages/<filename> has unclosed frontmatter block (missing closing ---)" }
     ```
   - Empty required field values:
     - `title:` is present but value is empty or whitespace-only → add finding:
       ```
       { severity: "ERROR", code: "invalid-frontmatter", message: "pages/<filename> has empty `title` field" }
       ```
     - `tags:` is present but value is `[]` or empty → add finding:
       ```
       { severity: "ERROR", code: "invalid-frontmatter", message: "pages/<filename> has empty `tags` list" }
       ```
     - `sources:` is present but value is `[]` or empty → this is acceptable (a page may have no external sources); do NOT flag.

### Check G — Duplicate Titles (WARNING)

**Definition**: Two or more pages share the same `title` frontmatter value.

Collect all `title` values from frontmatter across all `PAGE_FILES` (using content already read where available).

Build a map: `title_value → [list of filenames with that title]`

For each title value that maps to 2+ filenames: add one finding per duplicate group:
```
{ severity: "WARNING", code: "duplicate-title", message: "Title \"<title>\" used in: pages/<file1>, pages/<file2>" }
```

### Check H — Empty Pages (WARNING)

**Definition**: A page with fewer than 50 words of content (excluding the frontmatter block).

For each file in `PAGE_FILES` (using content already read where available):
1. Strip the frontmatter block: remove everything from the opening `---` to the closing `---` (inclusive)
2. Count words in the remaining content
3. If `content_word_count` < 50: add finding:
   ```
   { severity: "WARNING", code: "empty-page", message: "pages/<filename> has only <content_word_count> words of content (minimum: 50)" }
   ```

### Check I — Index Drift (WARNING)

**Definition**: An entry in `index.md` that references a page file that does not exist in `pages/`.

For each entry in `INDEX_ENTRIES`:
- Derive the expected filename: `<entry>.md`
- Check whether `<entry>.md` exists in `PAGE_FILES`
- If NOT found: add finding:
  ```
  { severity: "WARNING", code: "index-drift", message: "index.md references [[<entry>]] but pages/<entry>.md does not exist" }
  ```

### Check J — Log Integrity (INFO)

**Definition**: Log entries in `log.md` that are not in chronological order (dates going backwards).

For each consecutive pair of log entry lines in `LOG_LINES`:
- Extract the date from each: `## [YYYY-MM-DD] ...` → `YYYY-MM-DD`
- If `date[i]` > `date[i+1]` (string comparison works for ISO dates): add finding:
  ```
  { severity: "INFO", code: "log-order", message: "log.md has out-of-order entries: <date[i]> appears before <date[i+1]>" }
  ```

Only report the first out-of-order pair found (do not flood the report with every inversion).

---

## Step 4 — Generate Health Report

After all checks complete, output the health report.

### 4a — Compute summary counts

```
ERROR_COUNT   = count of findings with severity "ERROR"
WARNING_COUNT = count of findings with severity "WARNING"
INFO_COUNT    = count of findings with severity "INFO"
TOTAL_PAGES   = count of PAGE_FILES
```

### 4b — Determine health rating

- `GOOD` — `ERROR_COUNT == 0` AND `WARNING_COUNT == 0`
- `FAIR` — `ERROR_COUNT == 0` AND `WARNING_COUNT > 0`
- `POOR` — `ERROR_COUNT > 0`

### 4c — Output the report

Output using this exact structure:

```markdown
## Second Brain Health Report

**Brain**: {name} | **Pages**: {TOTAL_PAGES} | **Last ingest**: {LAST_INGEST}

### Errors (must fix)
- [ ] broken-link: [[nonexistent-page]] referenced in pages/topic-a.md (line 15)
- [ ] missing-frontmatter: pages/quick-note.md lacks `tags` field

### Warnings (should fix)
- [ ] orphan-page: pages/old-draft.md not in index.md
- [ ] oversized: pages/big-topic.md has 3,200 words (limit: 2,000)

### Info
- [ ] stale: pages/setup-guide.md not updated in 120 days (threshold: 90 days)

### Summary
- Errors: {ERROR_COUNT} | Warnings: {WARNING_COUNT} | Info: {INFO_COUNT}
- Overall health: {GOOD|FAIR|POOR} ...
```

**Formatting rules:**

- Each finding line: `- [ ] <code>: <message>` (checkbox format for easy tracking)
- Under `### Errors`: list all ERROR findings. If none, write `_None_`.
- Under `### Warnings`: list all WARNING findings. If none, write `_None_`.
- Under `### Info`: list all INFO findings. If none, write `_None_`.
- Under `### Summary`:
  - Line 1: `- Errors: {N} | Warnings: {M} | Info: {P}`
  - Line 2: `- Overall health: GOOD (0 errors, 0 warnings)` or `FAIR (0 errors, {M} warnings)` or `POOR ({N} errors)`

**If `FIX_MODE = true`**: append the fixes section after the summary (see Step 5). Do not append it when `FIX_MODE = false`.

---

## Step 5 — Apply Auto-Fixes (only when `--fix` is passed)

This step runs only when `FIX_MODE = true`.

Auto-fix only the following issue types. Do not auto-fix any other issue type.

### Fix 1 — Orphan Pages → Add to index.md

For each `orphan-page` finding:
1. Read the page file to extract its frontmatter `title` and `tags`
2. Get the current date from `TODAY` (already computed in Step 2d)
3. Derive a one-line summary: use the first sentence of the page body (after frontmatter), truncated to 100 characters. If no body content, use `"(no summary)"`.
4. Read `<STORAGE_PATH>/index.md`
5. Append a new row to the catalog table:
   ```
   | [[<page-stem>]] | <summary> | <tags-as-comma-list> | <TODAY> |
   ```
   Where `<tags-as-comma-list>` is the tags joined with `, ` (e.g., `ml, transformers`).
6. Write the updated `index.md` back.

### Fix 2 — Missing Frontmatter Fields → Add with defaults

For each `missing-frontmatter` finding where a specific field (not the entire block) is missing:
1. Read the page file
2. Locate the frontmatter block
3. For each missing field, add the following default values at the end of the frontmatter block (before the closing `---`):
   - `tags`: → `tags: [uncategorized]`
   - `created`: → `created: <TODAY>`
   - `updated`: → `updated: <TODAY>`
   - `sources`: → `sources: []`
4. Do NOT auto-add `title` — a missing title key inside an existing frontmatter block requires human judgment. This rule applies only when a frontmatter block already exists but is missing the `title` key.
5. Write the updated file back.

If the page has no frontmatter block at all (the `missing-frontmatter: has no YAML frontmatter block` finding), add a minimal frontmatter block at the top of the file:
```yaml
---
title: (untitled)
tags: [uncategorized]
created: <TODAY>
updated: <TODAY>
sources: []
---
```
Then write the file back. Note: the `title: (untitled)` here is a bootstrap placeholder inserted because no frontmatter existed at all — the user must manually update it to a meaningful title after the fix is applied.

### Fix 3 — Index Drift → Remove stale index.md entries

For each `index-drift` finding:
1. Read `<STORAGE_PATH>/index.md`
2. Remove the table row that references the non-existent page (the row containing `[[<entry>]]`)
3. Write the updated `index.md` back.

### Fix output

After applying all fixes, output:

```
Fixed {N} issues. Run `/second-brain lint` again to verify.
```

Where `{N}` is the total number of individual fix operations applied (one per finding resolved, not per finding type).

**Do NOT fix and do NOT modify any files for:**
- `broken-link` (requires human judgment — rename or remove?)
- `oversized` (splitting content requires judgment)
- `duplicate-title` (which page to rename requires judgment)
- `empty-page` (may be intentional stubs)
- `stale` (updating content requires judgment)
- `log-order` (log is append-only; reordering would alter history)

---

## Decision Tree Summary

```
/second-brain lint [--fix]
│
├─ Config exists?
│   └─ NO → error: run init first
│
├─ Detect --fix flag → FIX_MODE = true|false
│
├─ Inventory files:
│   ├─ List pages/*.md → PAGE_FILES
│   ├─ Parse index.md → INDEX_ENTRIES
│   ├─ Parse log.md → LOG_LINES, LAST_INGEST
│   └─ Get current date → TODAY
│
├─ Run all checks (collect ALL findings before reporting):
│   ├─ A: Orphan pages     (WARNING) — pages/ not in index.md
│   ├─ B: Broken wiki-links (ERROR)  — [[link]] targets that don't exist
│   ├─ C: Stale pages       (INFO)   — updated date > threshold
│   ├─ D: Oversized pages   (WARNING)— word count > max_page_words
│   ├─ E: Missing frontmatter (ERROR)— required fields absent
│   ├─ F: Invalid frontmatter (ERROR)— unclosed block or empty required fields
│   ├─ G: Duplicate titles  (WARNING)— same title in 2+ pages
│   ├─ H: Empty pages       (WARNING)— <50 words of content
│   ├─ I: Index drift       (WARNING)— index.md entries with no matching file
│   └─ J: Log integrity     (INFO)   — log entries out of chronological order
│
├─ Generate health report (always):
│   ├─ Header: brain name, page count, last ingest date
│   ├─ ### Errors (must fix)
│   ├─ ### Warnings (should fix)
│   ├─ ### Info
│   └─ ### Summary + health rating (GOOD | FAIR | POOR)
│
└─ FIX_MODE = true?
    ├─ YES → apply safe fixes:
    │         ├─ Fix 1: orphan pages → add to index.md
    │         ├─ Fix 2: missing frontmatter fields → add defaults
    │         └─ Fix 3: index drift → remove stale rows from index.md
    │         └─ Output: "Fixed N issues. Run /second-brain lint again to verify."
    └─ NO  → output report only (no file changes)
```

---

## Error Handling

- **Config not found**: stop immediately with "No second-brain found. Run `/second-brain init` first."
- **pages/ directory missing**: treat `PAGE_FILES = []`; proceed with checks (index drift, log integrity still run).
- **index.md missing**: treat `INDEX_ENTRIES = []`; record a WARNING and proceed with remaining checks.
- **log.md missing**: treat `LOG_LINES = []`, `LAST_INGEST = "never"`; proceed with remaining checks.
- **Page file unreadable** (permission denied): skip that file in all checks and add a WARNING:
  ```
  { severity: "WARNING", code: "unreadable", message: "pages/<filename> could not be read (permission denied)" }
  ```
- **Fix write failure** (permission denied, disk full): report the error and which fix failed. Continue applying remaining fixes rather than stopping.
- **index.md write failure during fix**: report the error. Do not leave index.md in a partially written state — read → modify in memory → write atomically.

---

## Performance Notes

For brains with up to 200 pages, lint must complete in under 5 seconds. To stay within this budget:

- **Batch file reads**: use `wc -w "<STORAGE_PATH>/pages/"*.md` once to get all word counts instead of one `wc -w` call per file.
- **Batch link extraction**: use a single `grep -n '\[\[' "<STORAGE_PATH>/pages/"*.md` call to extract all wiki-links across all files in one pass, then parse the output.
- **Reuse file content**: once a page is read in Check B (broken links), reuse the same content for Checks C, E, F, G, and H. Do not read the same file multiple times.
- **Single date call**: call `date` once in Step 2d and reuse `TODAY` for all stale calculations.

---

## Obsidian Compatibility Notes

- The health report is output as markdown — it renders cleanly in Obsidian if the user pastes it or saves it.
- `[[wiki-links]]` in findings use the page filename without `.md`, matching Obsidian conventions.
- Frontmatter fixes preserve the existing `---` delimiters and field order — only the missing fields are appended inside the block.
- Auto-fixed `index.md` rows follow the same `| [[page]] | summary | tags | date |` format used by ingest.
