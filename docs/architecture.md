# Architecture

## Philosophy: Karpathy LLM Wiki

This plugin implements the LLM Wiki pattern described by Andrej Karpathy:
- **Human directs** — you tell the LLM what to ingest, query, or maintain
- **LLM maintains** — the LLM reads, writes, and cross-references the wiki
- **Knowledge compounds** — each session adds to a persistent store that grows more useful over time

The key insight: LLMs are ephemeral, but markdown files are permanent. By externalizing knowledge into a structured wiki, you create a knowledge base that survives context windows, model upgrades, and session resets.

Reference: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

## Three-Layer Architecture

```
┌─────────────────────────────────────┐
│         Raw Sources Layer           │
│  .md files, .txt, .pdf, URLs, etc.  │
│  (unstructured, read-only inputs)   │
└──────────────┬──────────────────────┘
               │ /second-brain ingest
               ▼
┌─────────────────────────────────────┐
│           Wiki Layer                │
│  ~/.second-brain/ or .second-brain/ │
│  ├── index.md       (catalog)       │
│  ├── log.md         (ledger)        │
│  ├── *.md           (wiki pages)    │
│  └── second-brain-config.json       │
└──────────────┬──────────────────────┘
               │ schemas/
               ▼
┌─────────────────────────────────────┐
│           Schema Layer              │
│  config.schema.json                 │
│  frontmatter.schema.json            │
│  (validation + lint rules)          │
└─────────────────────────────────────┘
```

### Raw Sources Layer
Anything the user wants to preserve: meeting notes, research papers, chat logs, code comments, URLs. The ingest skill reads these and distills them into wiki pages. Raw sources are never modified.

### Wiki Layer
The living knowledge base. Every page is Obsidian-compatible markdown with YAML frontmatter. The `index.md` is an auto-maintained catalog; `log.md` is an append-only audit trail. All writes go through skill prompts so they conform to format rules.

### Schema Layer
JSON Schema files that define the shape of valid config and valid frontmatter. The lint skill uses these to validate the wiki. Skills reference these schemas in their prompts to enforce consistency.

## Storage Modes

| Mode | Path | Use Case |
|------|------|----------|
| Personal | `~/.second-brain/` | Individual knowledge across all projects |
| Team | `./.second-brain/` | Project-specific knowledge, checked into git |

Team mode brains can be committed to a repository (the `.gitignore` in this plugin repo excludes `.second-brain/` for the plugin's own development, but user project repos may choose to commit theirs).

## File Format Decisions

**Obsidian compatibility** was chosen because:
1. Obsidian is the dominant local-first markdown knowledge base tool
2. `[[wiki-links]]` are a widely understood convention
3. YAML frontmatter is standard across static site generators and note tools
4. Files remain human-readable and editable without any tooling

**Kebab-case filenames** were chosen because:
- URLs and filesystem paths are unambiguous
- No case-sensitivity issues across macOS/Linux/Windows
- Consistent with web conventions

**Append-only log** prevents accidental history loss and provides a clear audit trail for automated operations.
