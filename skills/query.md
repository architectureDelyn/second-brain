# Skill: `/second-brain query`

This skill implements the `query` pipeline for the second-brain plugin. When a user runs `/second-brain query <question>`, follow every step in this document precisely and in order. Execute the pipeline silently after loading config — do not interrupt with questions until the optional save-back offer at the end.

---

## Overview

The query pipeline searches the wiki and synthesizes a direct answer by:
1. Loading the brain config and resolving the storage path
2. Parsing the question into search terms
3. Searching `index.md` for matching pages (multi-tier ranking)
4. Loading the top matching pages
5. Synthesizing a cited answer from page content
6. Formatting the answer with confidence level and gaps
7. Optionally saving the synthesized answer as a new wiki page

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
- `schema.naming_convention` — for use if saving back a new page

Also resolve the **plugin root path**: the directory containing the CLAUDE.md that registered this skill. Used later to locate `templates/page.md`.

All file paths in subsequent steps are relative to `storage_path` (expand `~` to the actual home directory when constructing shell paths).

---

## Step 2 — Parse Question

Extract search terms from the user's natural language question.

Identify and store as `SEARCH_TERMS`:
- **Nouns and noun phrases** (e.g., "transformer architecture", "gradient descent")
- **Technical terms and acronyms** (e.g., "LSTM", "REST API", "OAuth")
- **Domain keywords** — topic-level words that anchor the question (e.g., "authentication", "caching", "deployment")

Strip common stop words that carry no search signal: *what*, *how*, *why*, *when*, *where*, *who*, *is*, *are*, *the*, *a*, *an*, *it*, *does*, *do*, *can*, *be*, *to*, *of*, *in*, *on*, *for*, *with*, *about*, *explain*, *tell*, *me*, *my*, *your*.

Store the original question text as `QUESTION` for use in output formatting.

Example: question "How does gradient descent work in neural networks?" → `SEARCH_TERMS` = ["gradient descent", "neural networks", "gradient", "descent", "neural"]

---

## Step 3 — Search index.md

Read `<STORAGE_PATH>/index.md`.

Parse the catalog table. Each row has the format:
```
| [[page-filename]] | summary text | tag1, tag2 | YYYY-MM-DD |
```

Extract three values from each row:
- **title**: the page filename inside `[[...]]` (e.g., `gradient-descent`)
- **summary**: the plain text summary (column 2)
- **tags**: the comma-separated tag string (column 3)

Store the total page count as `TOTAL_PAGES` (number of data rows, excluding the header row).

**Tier 1 — Title match (highest rank):**
For each search term in `SEARCH_TERMS`, check if it appears (case-insensitive) in the page title. A title match scores +3.

**Tier 2 — Tag match:**
For each search term, check if it appears (case-insensitive) in the page's tags. A tag match scores +2.

**Tier 3 — Summary match:**
For each search term, check if it appears (case-insensitive) in the page's summary. A summary match scores +1.

After scoring all pages, sort descending by score. Keep only pages with score > 0.

**Tier 4 — Filename grep (if fewer than 3 results so far):**

If the scored list has fewer than 3 pages, use the Bash tool to grep page filenames for the search terms:

```bash
ls "<STORAGE_PATH>/pages/" | grep -i "<term>"
```

Run this for the 2-3 highest-value terms. Collect any matching filenames not already in the scored list and append them at score +0.5 (below summary matches, above nothing).

**Tier 5 — Content grep fallback (if still fewer than 3 results):**

If still fewer than 3 pages after Tier 4, use the Bash tool to grep through page content. Limit to 10 files to avoid excessive I/O:

```bash
grep -ril "<term>" "<STORAGE_PATH>/pages/" | head -10
```

Run for the top 1-2 terms. Collect matching filenames not already in the scored list and append at score +0.25.

**Final result set:**
Take the top 10 pages by score. Store as `CANDIDATE_PAGES` (list of page filenames without `.md`, ordered by score descending).

If `CANDIDATE_PAGES` is empty (zero matches across all tiers), proceed directly to Step 7 (empty brain / no match output).

---

## Step 4 — Load Relevant Pages

For each page filename in `CANDIDATE_PAGES` (top 10), read the file using the Read tool:

```
<STORAGE_PATH>/pages/<page-filename>.md
```

If a file listed in the index cannot be read (e.g., deleted or missing), log a silent warning and skip that page. Do not stop the pipeline.

Store the loaded pages as `LOADED_PAGES` — a list of objects, each containing:
- `filename`: the page filename (without `.md`)
- `title`: the `title` field from YAML frontmatter
- `tags`: the `tags` list from YAML frontmatter
- `content`: the full raw file content (including frontmatter)
- `score`: the relevance score from Step 3

---

## Step 5 — Synthesize Answer

This is an LLM reasoning task. Using only the content of `LOADED_PAGES`, generate a comprehensive answer to `QUESTION`.

**Rules for synthesis:**
- Base the answer strictly on content found in the loaded pages. Do not add facts, claims, or explanations from general knowledge that are not present in the loaded pages.
- Every factual claim in the answer must be traceable to a specific loaded page.
- Use inline `[[page-filename]]` citations immediately after each claim sourced from a page. Example: "Gradient descent updates weights by computing the loss gradient [[gradient-descent]]."
- If multiple pages contribute to one claim, cite all of them: "... [[gradient-descent]] [[backpropagation]]."
- Do not cite a page you did not read. Do not fabricate page names.
- If loaded pages partially answer the question (some but not all aspects covered), answer what is covered and identify what is missing in the Gaps section.

**Determine confidence level** based on `LOADED_PAGES` count and answer coverage:
- `HIGH` — 5 or more relevant pages loaded AND the answer directly addresses the question
- `MEDIUM` — 2–4 relevant pages loaded, OR the answer only partially addresses the question
- `LOW` — 0–1 relevant pages loaded, OR no direct answer possible from the content

Store the synthesized answer text, the confidence level, and any identified gaps for Step 6.

---

## Step 6 — Format and Output Answer

Output the full answer using this exact structure:

```markdown
## Answer

<synthesized answer with inline [[page-filename]] citations>

### Sources Referenced
- [[page-name-1]] — "<relevant excerpt of ~10-20 words from that page>"
- [[page-name-2]] — "<relevant excerpt of ~10-20 words from that page>"

### Confidence
[HIGH | MEDIUM | LOW]

### Gaps Identified
- <specific topic missing from the brain that would help answer this question>
- <another missing topic, if any>
```

**Sources Referenced rules:**
- List only pages that were actually cited in the answer body (subset of `LOADED_PAGES`)
- The excerpt must be a verbatim quote from the page content — copy it exactly, do not paraphrase
- Keep excerpts to ~10–20 words (one sentence or part of a sentence)
- Do not list a source page that was not cited inline in the answer

**Confidence level rules:**
- Show only one level: `HIGH`, `MEDIUM`, or `LOW`
- Follow the thresholds from Step 5
- Do not add explanatory text beyond the level word on the confidence line

**Gaps Identified rules:**
- Name specific topics (e.g., "Momentum-based optimization variants", "Learning rate scheduling") — not vague gaps (e.g., "more information")
- If the answer is complete and no gaps are apparent, write: `None identified`
- Aim for 1–3 gap items; more than 3 suggests the question is too broad

---

## Step 7 — Handle No Results

If `CANDIDATE_PAGES` is empty after all tiers in Step 3, output exactly this message (substituting values):

```
No relevant pages found in your second-brain for: "<QUESTION>"

Your brain contains <TOTAL_PAGES> pages. Try:
  - Run `/second-brain ingest <file>` to add relevant knowledge
  - Use different keywords: <suggest 2-3 alternative phrasings of the question>
```

For the keyword suggestions, propose alternative phrasings based on synonyms or related terms from `SEARCH_TERMS`. Example: if the question was about "gradient descent", suggest terms like "optimization", "weight update", "backpropagation".

After outputting this message, stop. Do not proceed to Step 8.

---

## Step 8 — Optional Save-Back

After displaying the answer (Step 6), determine whether the synthesized answer contains **new insight** — i.e., a synthesis or connection across pages that is not already captured verbatim in any single existing page.

Heuristic: if the answer draws on 3+ pages and combines their information in a non-trivial way, it likely contains new insight.

If new insight is detected, offer save-back:

```
Save this answer as a new wiki page? (yes/no)
```

Wait for the user's response:
- `yes`, `y`, `save` → proceed with save-back below
- `no`, `n`, `skip`, `cancel`, or no response after the answer is acknowledged → stop. Do not modify any files.
- Any other response → treat as `no`.

**If user confirms save-back:**

1. Use the Bash tool to get the current date:
   ```bash
   date -u +"%Y-%m-%d"
   ```
   Store as `TODAY`.

2. Derive a page filename from the question text using the naming convention from config:
   - Take the first 4-6 significant words from `QUESTION` (stripped of stop words)
   - Apply the naming convention (kebab-case by default)
   - Example: question "How does gradient descent work?" → filename `gradient-descent-explained`

3. Read the template from `<plugin-root>/templates/page.md`.

4. Replace every token:

   | Token | Replace with |
   |-------|-------------|
   | `{{TITLE}}` | A human-readable title derived from the question (e.g., "Gradient Descent Explained") |
   | `{{TAGS}}` | Tags derived from `SEARCH_TERMS` (comma-separated, no quotes, inside `[...]`) |
   | `{{CREATED_DATE}}` | `TODAY` (YYYY-MM-DD) |
   | `{{UPDATED_DATE}}` | `TODAY` (YYYY-MM-DD) |
   | `{{SOURCES}}` | `query-synthesis` (indicates this page was synthesized, not ingested from a file) |
   | `{{CONTENT}}` | The full synthesized answer text from Step 5 (without the markdown headers — just the answer body and sources list) |
   | `{{RELATED_LINKS}}` | `[[wiki-links]]` for each page cited in the answer |

5. Write the completed file to `<STORAGE_PATH>/pages/<derived-filename>.md`.

6. Append a new row to `<STORAGE_PATH>/index.md`:
   ```
   | [[<derived-filename>]] | <one-line summary of the synthesized answer> | <tags> | <TODAY> |
   ```

7. Append a log entry to `<STORAGE_PATH>/log.md`:
   ```
   ## [<TODAY>] query | "<QUESTION>" → saved as [[<derived-filename>]]
   ```

8. Output confirmation:
   ```
   ✅ Saved as [[<derived-filename>]] in your second-brain.
   ```

If save-back is not offered (answer drew on fewer than 3 pages) or the user declines, make no changes to any files.

---

## Decision Tree Summary

```
/second-brain query <question>
│
├─ Config exists?
│   └─ NO → error: run init first
│
├─ Parse question → extract SEARCH_TERMS
│
├─ Search index.md (multi-tier scoring):
│   ├─ Tier 1: title match  (+3)
│   ├─ Tier 2: tag match    (+2)
│   ├─ Tier 3: summary match (+1)
│   ├─ Tier 4: filename grep (+0.5)  ← only if <3 results
│   └─ Tier 5: content grep (+0.25) ← only if still <3 results
│
├─ CANDIDATE_PAGES empty?
│   └─ YES → output "no relevant pages" message → STOP
│
├─ Load top 10 matching pages
│
├─ Synthesize answer (LLM task):
│   ├─ Facts only from loaded pages
│   ├─ Inline [[citations]] per claim
│   └─ Determine confidence (HIGH/MEDIUM/LOW)
│
├─ Output formatted answer:
│   ├─ ## Answer
│   ├─ ### Sources Referenced
│   ├─ ### Confidence
│   └─ ### Gaps Identified
│
└─ Save-back offer (if 3+ pages synthesized):
    ├─ yes → create page, update index.md + log.md → confirm
    └─ no  → STOP (no file changes)
```

---

## Error Handling

- **Config not found**: stop immediately with "No second-brain found. Run `/second-brain init` first."
- **index.md not found or empty**: treat as zero pages; output the no-results message with `TOTAL_PAGES = 0`.
- **Page file missing** (in index but not on disk): skip silently, continue with remaining pages.
- **All candidate pages missing from disk**: treat as zero results; output the no-results message.
- **Template not found** (during save-back): report the error and which template path was expected; do not save a partial page.
- **index.md or log.md write fails** (during save-back): report the error; warn the user that the page file was written but index/log were not updated. Do not attempt a partial rollback.

---

## Obsidian Compatibility Notes

- Inline citations use `[[page-filename]]` with the filename **without** `.md`.
- The synthesized answer in the output uses these links as display text — they are readable as-is in Obsidian.
- Any save-back page created must use YAML frontmatter between `---` delimiters (from the template).
- Tags in save-back pages must be a YAML list: `tags: [tag1, tag2]`.
- Never include raw HTML in generated page content.
- The `sources: [query-synthesis]` value in save-back frontmatter signals that this page was created by query synthesis, not file ingestion.
