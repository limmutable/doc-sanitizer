# doc-sanitizer

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for updating and maintaining project documentation. It analyzes git history, discovers documentation structure, identifies duplications and broken links, and produces current, consistent documentation.

## What is a Claude Code Skill?

Claude Code skills are reusable prompt templates that extend Claude's capabilities with specialized knowledge and workflows. They can be invoked via slash commands (e.g., `/doc-sanitizer`) within Claude Code sessions.

## What doc-sanitizer Does

This skill automates common documentation maintenance tasks:

- **Discover** documentation structure across any project
- **Analyze** git history to find code changes that need doc updates
- **Identify** broken internal links and duplicate content
- **Update** outdated documentation to match current code
- **Consolidate** duplicated content into authoritative sources

### Workflow

The skill follows a 5-phase process:

1. **Discovery** - Auto-detect markdown files, doc directories, AI context files
2. **Analysis** - Examine git history, find broken links, identify duplicates
3. **Planning** - Prioritize updates, map duplications, list fixes needed
4. **Execution** - Update content, fix links, consolidate duplicates
5. **Verification** - Validate links, check for remaining issues

## Installation

### Via Symlink (Recommended)

```bash
# Clone this repository
git clone https://github.com/yourusername/doc-sanitizer.git ~/Projects/doc-sanitizer

# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Symlink the skill
ln -s ~/Projects/doc-sanitizer/skills ~/.claude/skills/doc-sanitizer
```

### Manual Installation

Copy the `skills/SKILL.md` file to `~/.claude/skills/doc-sanitizer/SKILL.md`.

## Usage

Invoke the skill within a Claude Code session using slash commands:

```bash
# Incremental scan (fast, recent changes only) - DEFAULT
/doc-sanitizer

# Full scan (thorough, all files)
/doc-sanitizer --full

# Discover doc structure
/doc-sanitizer --discover

# Preview all changes (dry-run)
/doc-sanitizer --dry-run

# Find deletion candidates (orphaned, obsolete docs)
/doc-sanitizer --prune

# Update specific file
/doc-sanitizer --target docs/architecture.md

# Fix broken links only
/doc-sanitizer --fix-links
```

### Scan Modes

| Mode | Use Case | Speed |
|------|----------|-------|
| Incremental (default) | Daily development, CI/CD | Fast |
| `--full` | Pre-release, audits, initial setup | Thorough |
| `--target <path>` | Single file updates | Fastest |

### Options

| Option | Description |
|--------|-------------|
| `--full` | Full scan - analyze all documentation files |
| `--prune` | Find deletion candidates (orphaned, obsolete, duplicate) |
| `--discover` | Show detected doc structure and exit |
| `--dry-run` | Preview changes without modifying files |
| `--target <path>` | Update specific file or directory only |
| `--fix-links` | Only fix broken internal links |
| `--find-duplicates` | Only identify duplicate content |
| `--check-toc` | Validate table of contents against headers |
| `--check-code` | Verify code examples reference existing files/functions |
| `--check-images` | Verify referenced images and assets exist |
| `--check-terminology` | Flag inconsistent terminology |
| `--check-staleness` | Find docs not updated but related code changed |
| `--since <date>` | Only consider changes since date |
| `--verbose` | Show detailed output during execution |

## Configuration (Optional)

Projects can create `.doc-sanitizer.yaml` for customization. If absent, the skill uses discovery-based defaults.

```yaml
# .doc-sanitizer.yaml
docs:
  root: docs/
  readme: README.md
  changelog: CHANGELOG.md

ai_context:
  - CLAUDE.md
  - GEMINI.md

ignore:
  - node_modules/
  - vendor/
  - .git/

authoritative_sources:
  installation: docs/getting-started.md
  architecture: docs/reference/architecture.md
  api: docs/reference/api.md

terminology:
  prefer:
    configuration: [config, conf]
    repository: [repo]

staleness_threshold: 90  # days

# Scan mode settings
scan:
  default_mode: incremental  # "incremental" or "full"
  incremental_window: 7 days # Fallback if no git tags

# Pruning settings
prune:
  suggest_deletion:
    orphaned: true           # Docs not linked from anywhere
    obsolete_features: true  # Docs referencing deleted code
  protect:
    - README.md
    - LICENSE*
    - CHANGELOG.md
```

See [skills/SKILL.md](skills/SKILL.md) for full documentation.

## License

MIT
