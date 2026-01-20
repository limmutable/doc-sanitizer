# doc-sanitizer

A Claude Code skill for updating and maintaining project documentation.

## Installation

This skill is installed via symlink at `~/.claude/skills/doc-sanitizer`.

## Usage

```bash
# Discover doc structure
/doc-sanitizer --discover

# Preview all changes (dry-run)
/doc-sanitizer --dry-run

# Update specific file
/doc-sanitizer --target docs/architecture.md

# Fix broken links only
/doc-sanitizer --fix-links

# Find duplicate content
/doc-sanitizer --find-duplicates
```

See [SKILL.md](SKILL.md) for full documentation.

## Development

```bash
# Install dependencies
uv sync

# Run tests
uv run pytest

# Run linting
uv run ruff check .

# Format code
uv run ruff format .
```
