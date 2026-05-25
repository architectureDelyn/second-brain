# Skill: `/second-brain init`

This skill implements the `init` wizard for the second-brain plugin. When a user runs `/second-brain init`, follow every step in this document precisely and in order. Ask questions one at a time — never batch multiple questions in a single prompt.

---

## Overview

The init wizard creates the full second-brain directory structure by:
1. Detecting whether a brain already exists
2. Collecting configuration via interactive questions
3. Creating directories and files from templates
4. Outputting a success summary with next steps

---

## Step 1 — Detect Existing Brain

Before asking anything, check whether a brain is already initialized.

Check both locations using the Read tool (each check will return an error if the file does not exist — that is expected):

- `.second-brain/second-brain-config.json` (relative to current working directory)
- `~/.second-brain/second-brain-config.json` (user home directory)

**If either file exists:**

Warn the user:

```
⚠️  A second-brain already exists at: <path where config was found>

What would you like to do?
  (a) Reconfigure — overwrite the existing config and re-run the wizard
  (b) Abort — keep the existing brain unchanged
```

Wait for the user's response.
- If the user chooses **(b) Abort** (or any equivalent: "abort", "cancel", "stop", "no", "keep it"): output "Aborted. Your existing brain is unchanged." and stop immediately. Do not proceed.
- If the user chooses **(a) Reconfigure** (or any equivalent: "reconfigure", "overwrite", "redo", "yes"): continue with Step 2 and overwrite during creation steps.

**If no config file exists:** proceed directly to Step 2.

---

## Step 2 — Ask: Personal or Team Mode?

Ask exactly this question (nothing else in the prompt):

```
What mode should this second-brain use?

  (1) Personal — knowledge base for your individual use, stored in ~/.second-brain/
  (2) Team     — shared knowledge base committed to git, stored in ./.second-brain/

Enter 1 or 2 (or type "personal" / "team"):
```

Wait for the user's response. Accept:
- `1`, `personal`, `p` → mode = `"personal"`
- `2`, `team`, `t` → mode = `"team"`

Store the chosen mode for use in later steps.

---

## Step 3 — Ask: Brain Name

Determine the default name:
- Team mode default: the name of the current working directory (basename only, e.g., if cwd is `/Users/alice/projects/my-app`, default is `my-app`)
- Personal mode default: `Personal Brain`

Ask exactly this question, substituting the default:

```
What should this brain be called?

Press Enter to accept the default, or type a new name.
Default: <DEFAULT_NAME>
```

Wait for the user's response.
- If the user presses Enter or submits blank input: use the default.
- Otherwise: use the user-provided name as-is.

Store the brain name.

---

## Step 4 — Ask: Storage Path

Determine the default storage path:
- Personal mode default: `~/.second-brain/`
- Team mode default: `./.second-brain/`

Ask exactly this question, substituting the default:

```
Where should the brain be stored?

Press Enter to accept the default, or enter a custom path.
Default: <DEFAULT_PATH>
```

Wait for the user's response.
- If blank: use the default path.
- If the user provides a path: use it exactly as given (do not normalize or expand `~` in the stored value; keep it as the user entered it).

Store the storage path. This will be written verbatim into the config as `storage_path`.

---

## Step 5 — Ask: Naming Convention

Ask exactly this question:

```
Which file naming convention should wiki pages use?

  (1) kebab-case  — my-topic-name.md  (recommended)
  (2) camelCase   — myTopicName.md
  (3) free-form   — any name you like

Enter 1, 2, or 3 (default: 1 kebab-case):
```

Wait for the user's response.
- `1`, `kebab-case`, blank/Enter → `"kebab-case"`
- `2`, `camelCase`, `camel` → `"camelCase"`
- `3`, `free-form`, `free`, `freeform` → `"free-form"`

Store the naming convention.

---

## Step 6 — Ask: Stale Threshold

Ask exactly this question:

```
After how many days without an update should a wiki page be flagged as stale?

Enter a number of days (default: 90):
```

Wait for the user's response.
- If blank/Enter: use `90`.
- Otherwise: parse as an integer. If the input is not a positive integer, re-ask once with: "Please enter a positive whole number (e.g., 90):"

Store the stale threshold as an integer.

---

## Step 7 — Create Directory Structure

Now create all files. Do not ask any more questions — execute all creation steps silently and report results at the end.

### 7a — Resolve Timestamps

Get the current date and time. Use the Bash tool:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"   # CREATED_AT (ISO 8601 UTC)
date -u +"%Y-%m-%d"               # CREATED_DATE
```

Store:
- `CREATED_AT` = full ISO 8601 datetime string (e.g., `2026-05-25T14:30:00Z`)
- `CREATED_DATE` = date only (e.g., `2026-05-25`)
- `UPDATED_AT` = same value as `CREATED_AT`

### 7b — Create Directories

Use the Bash tool to create the required directories:

```bash
mkdir -p "<STORAGE_PATH>/pages"
mkdir -p "<STORAGE_PATH>/sources"
```

Where `<STORAGE_PATH>` is the storage path collected in Step 4. Expand `~` to the actual home directory path when calling mkdir (use `$HOME` in the shell command).

### 7c — Create `pages/.gitkeep`

Create an empty `.gitkeep` file so the `pages/` directory is tracked by git even when empty:

```bash
touch "<STORAGE_PATH>/pages/.gitkeep"
```

### 7d — Create `sources/manifest.json`

Write the file `<STORAGE_PATH>/sources/manifest.json` with this exact content:

```json
{"sources": []}
```

### 7e — Create `second-brain-config.json`

Read the template from `templates/config.json` (path relative to the plugin root — use the path where CLAUDE.md lives).

Replace every token:

| Token | Replace with |
|-------|-------------|
| `{{MODE}}` | `personal` or `team` (from Step 2) |
| `{{BRAIN_NAME}}` | brain name (from Step 3) |
| `{{STORAGE_PATH}}` | storage path string (from Step 4) |
| `{{CREATED_AT}}` | ISO 8601 datetime (from Step 7a) |

Also substitute user-configured values into the nested `schema` object:
- `"stale_threshold_days": 90` → replace `90` with the stale threshold from Step 6
- `"naming_convention": "kebab-case"` → replace `"kebab-case"` with the chosen convention from Step 5

Write the completed JSON to `<STORAGE_PATH>/second-brain-config.json`.

**Validation**: Confirm the written JSON is valid (parseable) and contains all required fields: `version`, `mode`, `name`, `storage_path`, `created_at`, `schema`, `ingest`. If any field is missing, stop and report the error.

### 7f — Create `index.md`

Read the template from `templates/index.md`.

Replace every token:

| Token | Replace with |
|-------|-------------|
| `{{BRAIN_NAME}}` | brain name (from Step 3) |
| `{{UPDATED_AT}}` | UPDATED_AT datetime string (from Step 7a) |

Write the completed file to `<STORAGE_PATH>/index.md`.

The resulting file must contain the full catalog table header:

```markdown
| Page | Summary | Tags | Updated |
|------|---------|------|---------|
```

(The template already includes this — do not remove it.)

### 7g — Create `log.md`

Read the template from `templates/log.md`.

Replace every token:

| Token | Replace with |
|-------|-------------|
| `{{BRAIN_NAME}}` | brain name (from Step 3) |
| `{{CREATED_DATE}}` | CREATED_DATE (from Step 7a, format: YYYY-MM-DD) |

Write the completed file to `<STORAGE_PATH>/log.md`.

The first log entry must be:

```
## [YYYY-MM-DD] init | Created second brain "<BRAIN_NAME>"
```

---

## Step 8 — Team Mode Extras

If mode is `"team"`, output this reminder after all files are created:

```
📌 Team mode reminder:
   Add .second-brain/ to git and commit — this is your shared team knowledge base.
   Do NOT add .second-brain/ to .gitignore.

   git add .second-brain/
   git commit -m "feat: initialize second-brain knowledge base"
```

If mode is `"personal"`, skip this step.

---

## Step 9 — Output Success Message

After all files are created successfully, output a structured success summary:

```
✅ Second Brain initialized successfully!

  Brain name:   <BRAIN_NAME>
  Mode:         <personal|team>
  Storage path: <STORAGE_PATH>

Files created:
  <STORAGE_PATH>/second-brain-config.json
  <STORAGE_PATH>/index.md
  <STORAGE_PATH>/log.md
  <STORAGE_PATH>/pages/          (empty, with .gitkeep)
  <STORAGE_PATH>/sources/manifest.json

Next steps:
  • Add knowledge:   /second-brain ingest <file>
  • Search the brain: /second-brain query <question>
  • Validate pages:  /second-brain lint
```

---

## Decision Tree Summary

```
/second-brain init
│
├─ Config file exists?
│   ├─ YES → warn → ask (a) reconfigure or (b) abort
│   │          └─ abort → STOP
│   └─ NO  → continue
│
├─ [Q1] Personal (1) or Team (2)?
├─ [Q2] Brain name? (default: dir name or "Personal Brain")
├─ [Q3] Storage path? (default: ~/.second-brain/ or ./.second-brain/)
├─ [Q4] Naming convention? (default: kebab-case)
├─ [Q5] Stale threshold in days? (default: 90)
│
└─ Create files:
    ├─ mkdir pages/ sources/
    ├─ touch pages/.gitkeep
    ├─ write sources/manifest.json
    ├─ write second-brain-config.json  ← template/config.json + tokens
    ├─ write index.md                  ← templates/index.md + tokens
    ├─ write log.md                    ← templates/log.md + tokens
    ├─ (team) print git reminder
    └─ print success message
```

---

## Error Handling

- If a directory cannot be created (e.g., permission denied): stop immediately and report the full error. Do not create partial files.
- If a template file cannot be read: stop and report which template is missing and where it was expected.
- If the final config JSON is invalid: stop and report the validation error.
- If re-running init over an existing brain (reconfigure path): overwrite files without further prompting — the user already confirmed in Step 1.

---

## Obsidian Compatibility Notes

All generated files must be Obsidian-compatible:
- `index.md` and `log.md` do not have YAML frontmatter (they are special system files, not wiki pages).
- Wiki pages created by other skills (`ingest`, etc.) use `[[wiki-link]]` double-bracket syntax for internal links.
- The `pages/` directory is where all wiki pages will live; it must exist after init.
- The config, index, and log file names are exact and must not be varied.
