# Development Guide

## Local Testing

Clone the repo and add it as a local plugin:

```bash
git clone https://github.com/architectureDelyn/second-brain ~/dev/second-brain-plugin
claude plugin add ~/dev/second-brain-plugin
```

Claude Code reads `CLAUDE.md` from the plugin root and registers the slash commands. Changes to skill `.md` files take effect immediately — no build step required.

To remove and re-add after changes:

```bash
claude plugin remove second-brain
claude plugin add ~/dev/second-brain-plugin
```

## Iterating on Skill Prompts

Skill files in `skills/` are plain markdown. Edit them and the next invocation picks up the changes automatically. There is no compilation, bundling, or restart required.

Workflow:
1. Edit `skills/<name>.md`
2. Open a new Claude Code session (or the same session if the plugin reloads live)
3. Run `/second-brain <command>` and observe behavior
4. Refine prompt, repeat

**Tips:**
- Be explicit about file paths in prompts — the LLM needs to know exactly where to read/write
- Include the relevant schema reference so the LLM validates its own output
- Test edge cases: empty brain, missing config, oversized pages, stale entries

## Schema Evolution

When the config or frontmatter schema needs to change:

1. Bump the `version` field in `schemas/config.schema.json` title/description
2. Update `templates/config.json` with new fields and defaults
3. Add a migration note to `skills/init.md` — the init skill should detect old schema versions and offer to upgrade
4. Document the breaking change in `docs/architecture.md`

**Version strategy:** `MAJOR.MINOR.PATCH`
- PATCH: Non-breaking additions (new optional fields)
- MINOR: New required fields with defaults (init can auto-migrate)
- MAJOR: Structural changes requiring manual migration

## Testing Workflow

Use a scratch directory to avoid polluting your real brains:

```bash
mkdir -p /tmp/sb-test && cd /tmp/sb-test
```

Then run skill commands against this scratch space. After testing:

```bash
rm -rf /tmp/sb-test
```

### Shell Assertion Examples

Verify a brain was initialized correctly:

```bash
BRAIN=~/.second-brain

# Config exists and has required fields
test -f "$BRAIN/second-brain-config.json" && echo "PASS: config exists"
python3 -c "import json; c=json.load(open('$BRAIN/second-brain-config.json')); assert 'version' in c" && echo "PASS: version field"

# index.md and log.md exist
test -f "$BRAIN/index.md" && echo "PASS: index exists"
test -f "$BRAIN/log.md" && echo "PASS: log exists"

# Log has an init entry
grep -q "init |" "$BRAIN/log.md" && echo "PASS: init log entry"
```

Verify a page has valid frontmatter:

```bash
PAGE="$BRAIN/my-page.md"

# Has frontmatter delimiters
head -1 "$PAGE" | grep -q "^---$" && echo "PASS: frontmatter start"

# Has required fields
grep -q "^title:" "$PAGE" && echo "PASS: title"
grep -q "^tags:" "$PAGE" && echo "PASS: tags"
grep -q "^created:" "$PAGE" && echo "PASS: created"
grep -q "^updated:" "$PAGE" && echo "PASS: updated"
grep -q "^sources:" "$PAGE" && echo "PASS: sources"
```

## Plugin Distribution

Once skills are complete (Steps 2-5), publish to GitHub and users install with:

```bash
claude plugin add github.com/architectureDelyn/second-brain
```

Claude Code fetches the repo and reads `CLAUDE.md` to register commands. No npm publish, no PyPI — just a GitHub URL.
