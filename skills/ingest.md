# Skill: `/second-brain ingest`

This skill implements the `ingest` pipeline for the second-brain plugin. When a user runs `/second-brain ingest <source>`, follow every step in this document precisely and in order. Execute the pipeline silently after validation — do not interrupt with questions unless a duplicate is detected.

---

## Overview

The ingest pipeline converts raw source files (`.md`, `.txt`, `.pdf`) into structured wiki pages by:
1. Loading the brain config and validating the source
2. Checking for duplicates
3. Reading and chunking large content
4. Extracting key concepts via LLM analysis
5. Creating or updating wiki pages from `templates/page.md`
6. Cross-referencing pages with `[[wiki-links]]`
7. Updating `index.md`, `log.md`, and `sources/manifest.json`

---

## Step 1 — Load Config

Read `second-brain-config.json` from the brain storage path.

Try both locations using the Read tool:
- `.second-brain/second-brain-config.json` (relative to current working directory)
- `~/.second-brain/second-brain-config.json` (user home directory)

Use the first one that exists. If neither exists, stop immediately and output:

```
Error: No second-brain found. Run `/second-brain init` first.
```

From the config, extract and store:
- `storage_path` — the root directory of the brain
- `schema.naming_convention` — `"kebab-case"`, `"camelCase"`, or `"free-form"`
- `ingest.auto_cross_reference` — boolean, controls wiki-link insertion

All file paths in subsequent steps are relative to `storage_path` (expand `~` to the actual home directory when constructing shell paths).

---

## Step 2 — Validate Source

The `<source>` argument is the file path or directory path provided by the user.

**If source is a directory:**
- Walk the directory recursively using the Bash tool:
  ```bash
  find "<source>" -type f \( -name "*.md" -o -name "*.txt" -o -name "*.pdf" \)
  ```
- Collect all matching files into a list. If no supported files are found, stop and output:
  ```
  Error: No supported files found in "<source>". Supported formats: .md, .txt, .pdf
  ```
- Process each file sequentially through Steps 3–10, then produce a combined summary in Step 11.

**If source is a single file:**
- Verify the file exists. If not, stop and output:
  ```
  Error: File not found: "<source>"
  ```
- Verify the extension is `.md`, `.txt`, or `.pdf`. If not, stop and output:
  ```
  Error: Unsupported file type "<extension>". Supported formats: .md, .txt, .pdf
  ```
- Continue with Step 3.

---

## Step 3 — Check for Duplicate

Read `<STORAGE_PATH>/sources/manifest.json`.

Compute the SHA-256 hash of the source file using the Bash tool:

```bash
shasum -a 256 "<source-path>"
```

Extract the hash value (first field of output). Search the `sources` array in `manifest.json` for an entry where `"hash"` matches this value.

**If a match is found:**
- Extract the `ingested_at` date from the matching entry (format as `YYYY-MM-DD`).
- Ask the user exactly:
  ```
  This source was already ingested on <YYYY-MM-DD>. Re-ingest to update existing pages? (yes/no)
  ```
- Wait for the user's response.
  - `yes`, `y`, `re-ingest`, `update` → continue with Step 4 (update mode: existing pages will be updated rather than duplicated)
  - `no`, `n`, `skip`, `cancel` → stop and output: `Skipped. No changes made.`
  - Any other response → re-ask once with: `Please answer yes or no.` If still unrecognized, stop with: `Skipped. No changes made.`

**If no match is found:** continue directly to Step 4 (create mode).

Store a flag `INGEST_MODE` = `"update"` or `"create"` based on the above.

---

## Step 4 — Read Source Content

Read the full content of the source file based on its extension.

**For `.md` and `.txt` files:**
- Use the Read tool to read the full file content.

**For `.pdf` files:**
- Use the Read tool to read the file. Claude Code supports PDF reading natively.
- If the PDF appears to contain only images (no extractable text — e.g., the content is empty or contains only whitespace), output a warning and skip this file:
  ```
  Warning: "<source>" appears to be an image-only PDF. Text extraction failed — skipping.
  ```
  Do not stop the entire pipeline for a directory ingest; continue with the next file.

**Chunking large files (>3000 words):**

Count the approximate word count of the extracted text:
```bash
echo "<content>" | wc -w
```

Or estimate from character count (divide by 5 as approximation).

If word count exceeds 3000:
- Split the content into sequential 3000-word segments (segment 1: words 1–3000, segment 2: words 3001–6000, etc.).
- Process each segment independently through Step 5 (concept extraction).
- Merge all extracted concepts into a single deduplicated list before proceeding to Step 6.
- Prefer natural split points: paragraph breaks, section headings. Do not split mid-sentence.

Store the full extracted text (or merged concept list after chunking) for use in Step 5.

---

## Step 5 — Extract Concepts

This is an LLM reasoning task. Analyze the source content and identify **10–15 key concepts, topics, entities, or ideas** that are worth preserving as standalone wiki pages.

For each concept, determine:

1. **Concept name** — a clear, descriptive noun phrase (e.g., "Transformer Architecture", "Gradient Descent", "Attention Mechanism")
2. **Key information** — the essential facts, definitions, processes, or insights about this concept found in the source
3. **Related concepts** — other concepts from this same source that are closely connected

Apply the naming convention from config when deriving filenames:
- `kebab-case`: `transformer-architecture.md`
- `camelCase`: `transformerArchitecture.md`
- `free-form`: use the concept name as-is, replacing spaces with hyphens

**Quality criteria for concept selection:**
- Concepts should be self-contained (understandable as a standalone page)
- Avoid overly broad concepts (e.g., "Introduction") and overly narrow ones (e.g., "footnote 3")
- Prefer concepts that would be useful to look up independently
- For technical documents: include key algorithms, data structures, patterns, and terminology
- For narrative/prose documents: include key ideas, arguments, people, places, and themes

Store the extracted concept list for use in Step 6.

---

## Step 6 — File Into Pages

For each extracted concept, create or update a wiki page.

### 6a — Resolve Timestamps

Get the current date using the Bash tool:

```bash
date -u +"%Y-%m-%d"
```

Store as `TODAY` (format: `YYYY-MM-DD`). Use this value for all `created` and `updated` frontmatter fields in this ingest run.

### 6b — Check for Existing Page

Read `<STORAGE_PATH>/index.md` to get the list of existing pages.

For each concept, check whether an existing page already covers this topic. Match on:
- Page title similarity (same or closely related name)
- Shared tags
- Aliases listed in existing page frontmatter

Use fuzzy matching: "Machine Learning" and "ML" should match; "Neural Networks" and "Deep Learning" should not automatically match.

**If an existing page is found (update path):**
1. Read the existing page file from `<STORAGE_PATH>/pages/<existing-page-name>.md`
2. Parse the YAML frontmatter
3. Merge the new information:
   - Update the `updated` field to `TODAY`
   - Append the source path to the `sources` list (no duplicates)
   - Append new content below the existing content, preceded by a source attribution comment:
     ```markdown
     <!-- ingested from: <source-filename>, <TODAY> -->
     ```
   - Do not delete or overwrite existing content
4. Write the updated file back

**If no existing page is found (create path):**
1. Read the template from `templates/page.md` (path relative to the plugin root — use the directory where CLAUDE.md lives)
2. Replace every token:

   | Token | Replace with |
   |-------|-------------|
   | `{{TITLE}}` | Human-readable concept name (e.g., "Transformer Architecture") |
   | `{{TAGS}}` | Comma-separated tags derived from content and inferred category (minimum 1 tag). Do not quote tags. Example: `machine-learning, neural-networks, transformers` |
   | `{{CREATED_DATE}}` | `TODAY` (YYYY-MM-DD) |
   | `{{UPDATED_DATE}}` | `TODAY` (YYYY-MM-DD) |
   | `{{SOURCES}}` | The source file path or name (e.g., `notes/project-alpha.md`) |
   | `{{CONTENT}}` | The extracted knowledge for this concept (plain markdown, no YAML) |
   | `{{RELATED_LINKS}}` | Leave as empty string for now — filled in Step 7 |

3. Write the completed file to `<STORAGE_PATH>/pages/<concept-filename>.md`

### 6c — Enforce Page Size Limit

After writing each page, count its approximate word count. If a page exceeds **2000 words**:
- Create a `<concept>-overview.md` page with a brief summary and links to sub-pages
- Split the full content into `<concept>-part-1.md`, `<concept>-part-2.md`, etc. (each under 2000 words)
- Delete the oversized `<concept>.md` (replace it with the overview)
- Track all sub-page names as created pages for index and manifest updates

### 6d — Track Created and Updated Pages

Maintain two lists throughout Step 6:
- `PAGES_CREATED` — filenames (without `.md`) of newly created pages
- `PAGES_UPDATED` — filenames (without `.md`) of pages that were updated

---

## Step 7 — Cross-Reference

After all pages for this ingest run are created or updated, insert `[[wiki-links]]` into the "Related" section of each new/updated page.

For each page in `PAGES_CREATED` + `PAGES_UPDATED`:
1. Read the page
2. Collect all page titles and filenames from `PAGES_CREATED` + `PAGES_UPDATED` (excluding the current page)
3. Also scan `index.md` for the titles/filenames of all other existing pages in the brain
4. For each related concept mentioned in the page content, check if a wiki page exists for it
5. In the `## Related` section, add `[[wiki-link]]` entries for each related page found. Use the filename without `.md` as the link target. Example:
   ```markdown
   ## Related

   - [[transformer-architecture]]
   - [[attention-mechanism]]
   - [[gradient-descent]]
   ```
6. Write the updated page back

**Only insert links to pages that actually exist** (in `PAGES_CREATED`, `PAGES_UPDATED`, or already in `index.md`). Do not create links to hypothetical pages.

If `auto_cross_reference` is `false` in the config, skip this step entirely.

---

## Step 8 — Update index.md

Read `<STORAGE_PATH>/index.md`.

**For each page in `PAGES_CREATED`:** append a new row to the catalog table:

```
| [[<page-filename>]] | <one-line summary of the page content> | <tag1>, <tag2> | <TODAY> |
```

- `<page-filename>` is the filename without `.md` (e.g., `transformer-architecture`)
- The one-line summary is a concise description (max ~100 characters) of the page's subject
- Tags are the first 1–3 tags from the page's frontmatter

**For each page in `PAGES_UPDATED`:** find the existing row for that page in the table and update the `Updated` column value to `TODAY`.

Write the updated `index.md` back.

---

## Step 9 — Update log.md

Read `<STORAGE_PATH>/log.md`.

Append a new entry at the end of the file:

```
## [<TODAY>] ingest | <source-filename> → <N> pages created, <M> pages updated
```

Where:
- `<TODAY>` is the current date (YYYY-MM-DD)
- `<source-filename>` is the basename of the source file (e.g., `project-alpha.md`)
- `<N>` is the count of `PAGES_CREATED`
- `<M>` is the count of `PAGES_UPDATED`

For a directory ingest, append one log entry per source file processed (not one combined entry).

Write the updated `log.md` back.

---

## Step 10 — Update sources/manifest.json

Read `<STORAGE_PATH>/sources/manifest.json`.

**If `INGEST_MODE` is `"create"`** (no prior entry for this hash): append a new object to the `sources` array:

```json
{
  "path": "<source-path>",
  "hash": "<sha256>",
  "ingested_at": "<ISO8601>",
  "pages_created": ["<page-name-1>", "<page-name-2>"],
  "pages_updated": []
}
```

**If `INGEST_MODE` is `"update"`** (existing entry found in Step 3): find and replace the matching entry with:

```json
{
  "path": "<source-path>",
  "hash": "<sha256>",
  "ingested_at": "<ISO8601-now>",
  "pages_created": ["<page-name-1>"],
  "pages_updated": ["<existing-page>"]
}
```

Where:
- `<source-path>` is the full path as provided by the user
- `<sha256>` is the hash computed in Step 3
- `<ISO8601>` is the current datetime from the Bash tool: `date -u +"%Y-%m-%dT%H:%M:%SZ"`
- `pages_created` lists filenames without `.md` from `PAGES_CREATED`
- `pages_updated` lists filenames without `.md` from `PAGES_UPDATED`

Write the updated JSON back to `sources/manifest.json`. Validate it is parseable JSON before writing.

---

## Step 11 — Output Summary

After all steps complete, output a structured summary:

```
✅ Ingest complete!

  Source:         <source-path>
  Pages created:  <N>
  Pages updated:  <M>

Created pages:
  • <page-name-1>
  • <page-name-2>
  ...

Updated pages:
  • <existing-page>
  ...

Next steps:
  • Search your brain:  /second-brain query <question>
  • Validate pages:     /second-brain lint
```

If no pages were updated, omit the "Updated pages" section. If no pages were created, omit the "Created pages" section.

For a directory ingest, show one summary block per file processed, followed by a combined total:

```
✅ Directory ingest complete!

  Source directory: <directory-path>
  Files processed:  <F>
  Pages created:    <total-N>
  Pages updated:    <total-M>
```

---

## Decision Tree Summary

```
/second-brain ingest <source>
│
├─ Config exists?
│   └─ NO → error: run init first
│
├─ Source is a directory?
│   ├─ YES → collect all .md/.txt/.pdf files → process each sequentially
│   └─ NO  → validate single file (exists? supported extension?)
│
├─ Duplicate check (SHA-256 hash match in manifest?)
│   ├─ MATCH → ask user to re-ingest
│   │    ├─ yes → INGEST_MODE=update, continue
│   │    └─ no  → stop (no changes)
│   └─ NO MATCH → INGEST_MODE=create, continue
│
├─ Read content
│   ├─ .md / .txt → read full text
│   ├─ .pdf → extract text (warn + skip if image-only)
│   └─ >3000 words → chunk → process each chunk → merge concepts
│
├─ Extract 10–15 concepts (LLM task)
│
├─ For each concept:
│   ├─ Existing page in index? (fuzzy match)
│   │   ├─ YES → read, merge, update frontmatter → PAGES_UPDATED
│   │   └─ NO  → create from template → PAGES_CREATED
│   └─ Page > 2000 words? → split into overview + sub-pages
│
├─ Cross-reference: insert [[wiki-links]] in Related sections
│   └─ (skip if auto_cross_reference=false)
│
├─ Update index.md (new rows + updated dates)
├─ Update log.md (append entry)
├─ Update sources/manifest.json (add/update entry)
│
└─ Output summary
```

---

## Error Handling

- **Config not found**: stop immediately with "No second-brain found. Run `/second-brain init` first."
- **Source file not found**: stop with file-not-found error for single file; skip with warning for directory.
- **Unsupported file type**: stop for single file; skip with warning for directory.
- **Image-only PDF**: warn and skip (do not stop a directory ingest).
- **Template not found** (`templates/page.md`): stop and report which template is missing and where it was expected.
- **Manifest JSON invalid**: stop and report the parse error before any writes.
- **Page write fails** (permission denied, disk full): stop immediately and report the full error. Do not leave partial state — report which pages were written before the failure.
- **Directory with no supported files**: stop with a clear message listing what was found.

---

## Obsidian Compatibility Notes

- All created pages must use Obsidian-compatible markdown: YAML frontmatter between `---` delimiters, `[[wiki-link]]` double-bracket syntax, no raw HTML.
- `[[wiki-links]]` in the "Related" section use the page filename **without** the `.md` extension.
- Tags in YAML frontmatter must be a YAML list: `tags: [tag1, tag2]` (not `tags: tag1, tag2`).
- The `sources` field must also be a YAML list: `sources: [path/to/source.md]`.
- `index.md` and `log.md` do NOT have YAML frontmatter — they are system files.
- Page filenames must not contain spaces, uppercase letters (for kebab-case), or special characters other than hyphens.
- Sub-pages from the 2000-word split must be linked from their parent overview page using `[[wiki-links]]` so Obsidian graph view shows the relationship.
