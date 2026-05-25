# Second Brain for Claude Code

**A persistent, compounding knowledge base inspired by Karpathy's LLM Wiki.**

You direct it. The LLM maintains it. Knowledge compounds across sessions, projects, and teams.

---

## Install

```bash
claude plugin add github.com/architectureDelyn/second-brain
```


For company GitLab:

```bash
claude plugin add gitlab.<your-company>.com/architectureDelyn/second-brain
```

---

## Quick Start

```bash
# 1. Set up your brain
/second-brain init

# 2. Add knowledge
/second-brain ingest ./docs

# 3. Search your brain
/second-brain query "how does our auth system work?"
```

---

## Commands

| Command | Description | Example |
|---------|-------------|---------|
| `/second-brain init` | Interactive setup wizard — personal or team mode | `/second-brain init` |
| `/second-brain ingest <source>` | Ingest local files (`.md`, `.txt`, `.pdf`) or folders into the wiki | `/second-brain ingest ./architecture-notes.md` |
| `/second-brain query <question>` | Search your brain and get synthesized answers with citations | `/second-brain query "what are our API rate limits?"` |
| `/second-brain lint [--fix]` | Health check your brain — broken links, stale pages, schema errors. Add `--fix` to auto-repair safe issues | `/second-brain lint --fix` |

---

## Obsidian Integration

Open your `.second-brain/` folder as an Obsidian vault. All wiki pages use Obsidian-compatible `[[wiki-links]]` and YAML frontmatter — the graph view works out of the box.

---

## Storage Modes

| Mode | Path | When to use |
|------|------|-------------|
| **Personal** | `~/.second-brain/` | Private notes, cross-project knowledge, your own research |
| **Team** | `./.second-brain/` | Committed to git alongside your project, shared across the team |

The active mode and path are stored in `second-brain-config.json` at the storage root. Switch modes at any time via `/second-brain init`.

---

## Philosophy

This plugin follows the **Karpathy LLM Wiki** model: humans decide what matters and point the LLM at sources; the LLM does the work of structuring, linking, and maintaining the knowledge base. You never manually format a wiki page. Knowledge compounds — every ingest enriches the existing graph rather than replacing it.

---

## Contributing

1. Fork this repo
2. Create a feature branch: `git checkout -b feat/your-feature`
3. Open a PR against `main`

See [docs/development.md](docs/development.md) for local setup, skill authoring guidelines, and the schema reference.

---

## License

MIT
