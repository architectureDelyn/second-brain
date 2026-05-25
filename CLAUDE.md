# second-brain Plugin

<!-- plugin-manifest
name: second-brain
version: 1.0.0
description: Persistent, compounding knowledge base maintained by LLM. Inspired by Karpathy's LLM Wiki.
-->

## Commands

### `/second-brain init`
**Skill:** `skills/init.md`
Initialize a new second-brain in personal (`~/.second-brain/`) or team (`./.second-brain/`) mode.

### `/second-brain ingest`
**Skill:** `skills/ingest.md`
Ingest raw sources (markdown, text, PDFs) into the wiki layer, creating or updating wiki pages with proper frontmatter.

### `/second-brain query`
**Skill:** `skills/query.md`
Query the knowledge base using natural language. Returns relevant wiki pages, cross-references, and summaries.

### `/second-brain lint`
**Skill:** `skills/lint.md`
Validate all wiki pages against the schema: frontmatter completeness, naming conventions, stale pages, broken links, and oversized pages.

---

## Global Wiki Format Rules

All skills MUST adhere to these rules when reading or writing wiki content.

### Markdown Format
- All wiki pages use **Obsidian-compatible markdown**
- YAML frontmatter block at the top of every page (between `---` delimiters)
- Internal links use `[[wiki-link]]` syntax (double-bracket)
- No raw HTML in wiki pages

### Storage Paths
| Mode | Path |
|------|------|
| Personal | `~/.second-brain/` |
| Team | `./.second-brain/` (relative to project root) |

The active mode and path are read from `second-brain-config.json` in the storage root.

### Required Frontmatter Fields

Every wiki page MUST have these fields in its YAML frontmatter:

```yaml
---
title: "Human-readable page title"
tags: [tag1, tag2]          # at least one tag required
created: YYYY-MM-DD         # ISO date, set once at creation
updated: YYYY-MM-DD         # ISO date, update on every edit
sources: [url-or-path]      # list of source references (may be empty list)
aliases: []                 # always present; alternative names for the page (may be empty)
---
```

Validation schema: `schemas/frontmatter.schema.json`

### Log Entry Format

`log.md` is an append-only ledger. Every operation appends an entry:

```
## [YYYY-MM-DD] action | description
```

Examples:
```
## [2026-05-25] init | Created second brain "my-brain"
## [2026-05-25] ingest | Added 3 pages from ~/notes/project-alpha.md
## [2026-05-25] lint | Found 2 stale pages, 0 broken links
```

Valid actions: `init`, `ingest`, `update`, `query`, `lint`, `archive`, `delete`

### Index Entry Format

`index.md` maintains a catalog table. Each wiki page gets one row:

```
| [[page-name]] | one-line summary | tag1, tag2 | YYYY-MM-DD |
```

Full table header:
```markdown
| Page | Summary | Tags | Updated |
|------|---------|------|---------|
```

The index is auto-maintained — skills update it on ingest and lint. Humans should not edit `index.md` directly.

### Naming Convention

- All wiki page filenames: **kebab-case** (e.g., `machine-learning-basics.md`)
- No spaces, no uppercase, no underscores in filenames
- Special files: `index.md`, `log.md`, `second-brain-config.json` (exact names, no variation)

### Page Size Limit

- Maximum **2000 words** per wiki page
- If a page exceeds 2000 words during ingest or update, split it into sub-pages:
  - Parent: `topic.md` (overview + links to sub-pages)
  - Children: `topic-part-1.md`, `topic-part-2.md`, etc.
- The lint skill flags oversized pages as errors

### Cross-Referencing

- When ingesting, identify concepts that match existing page titles and insert `[[wiki-links]]`
- `auto_cross_reference` setting in config controls whether this is automatic or manual
- Always update the index when adding or renaming pages

### Config File

Every second-brain has a `second-brain-config.json` at the storage root. Schema: `schemas/config.schema.json`. Template: `templates/config.json`.
