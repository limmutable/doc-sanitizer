---
name: doc-sanitizer
description: Update and maintain project documentation. Use when user asks to "update docs", "refresh documentation", "sync docs with code", "fix outdated docs", or mentions documentation being stale. Analyzes git history, discovers doc structure, identifies duplications, and produces current, consistent documentation.
---

# Documentation Sanitizer

A general-purpose skill for keeping project documentation current and consistent.

## Quick Reference

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

## Workflow

### Phase 1: Discovery

Auto-detect the project's documentation structure. Run these commands to understand the layout:

```bash
# Find all markdown files (limit to avoid noise)
find . -name "*.md" -type f -not -path "./.git/*" -not -path "./node_modules/*" -not -path "./vendor/*" | head -50

# Check common doc locations
ls -la docs/ 2>/dev/null || echo "No docs/ directory"
ls -la documentation/ 2>/dev/null || echo "No documentation/ directory"
ls -la .context/ 2>/dev/null || echo "No .context/ directory"

# Find AI context files
ls -la CLAUDE.md GEMINI.md AGENTS.md CURSOR.md README.md 2>/dev/null

# Check for optional config
cat .doc-sanitizer.yaml 2>/dev/null || echo "No config found - using discovery mode"
```

**Common patterns to detect:**
- `docs/` or `documentation/` - Main documentation directory
- `README.md` - Project and subdirectory readmes
- AI context files (`CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, `CURSOR.md`)
- Progress tracking (`.context/progress.md`, `CHANGELOG.md`)
- Spec files (`specs/`, `design/`, `rfcs/`)

### Phase 2: Analysis

Gather information about what has changed:

```bash
# Recent commits (understand recent changes)
git log --oneline -30

# Files changed recently
git diff --name-only HEAD~20..HEAD 2>/dev/null || git log --oneline --name-only -20

# Documentation-related commits
git log --oneline --grep="doc" --grep="readme" -i --since="30 days ago"

# Find broken internal links
grep -rEoh '\[([^\]]+)\]\(([^)]+\.md)\)' docs/ 2>/dev/null | head -30
```

**Analysis outputs:**
- List of all documentation files with last modified date
- Code changes that may need doc updates
- Broken internal links
- Duplicate content candidates
- Orphaned/unreferenced documents

### Phase 3: Planning

Create an update plan based on findings:

**Priority order** (general to specific):
1. Technical reference docs (architecture, API)
2. Getting started / setup guides
3. Concept explanations
4. Index/navigation files (README.md)
5. Changelog/progress tracking

**Planning checklist:**
- [ ] Identify authoritative sources for each topic
- [ ] Map duplications to consolidation targets
- [ ] List broken links to fix
- [ ] Note outdated content from git diff analysis

### Phase 4: Execution

Update documents following the plan:

| Task | Action |
|------|--------|
| Update outdated content | Sync with current code/features |
| Fix broken links | Update paths to moved/renamed files |
| Consolidate duplicates | Keep one source, reference from others |
| Add missing docs | Document new features found in git history |
| Prune stale content | Remove references to deleted features |

**Execution principles:**
- Make minimal, targeted changes
- Preserve document structure and formatting
- Update timestamps/version references
- Maintain consistent terminology

### Phase 5: Verification

Validate the updates:

```bash
# Verify all internal links exist
for link in $(grep -rEoh '\[([^\]]+)\]\(([^)]+\.md)\)' docs/ 2>/dev/null | grep -oE '\(([^)]+)\)' | tr -d '()'); do
  [ ! -f "$link" ] && echo "BROKEN: $link"
done

# Check for common issues
grep -r "TODO" docs/ 2>/dev/null | head -10
grep -r "FIXME" docs/ 2>/dev/null | head -10
```

## Invocation Options

| Option | Description |
|--------|-------------|
| `--dry-run` | Preview changes without modifying files |
| `--target <path>` | Update specific file or directory only |
| `--since <date>` | Only consider changes since date (e.g., `--since 2024-01-01`) |
| `--fix-links` | Only fix broken internal links |
| `--find-duplicates` | Only identify duplicate content |
| `--discover` | Show detected doc structure and exit |
| `--verbose` | Show detailed output during execution |

## Optional Configuration

Projects can create `.doc-sanitizer.yaml` for customization. If absent, the skill uses discovery-based defaults.

```yaml
# .doc-sanitizer.yaml (optional)
docs:
  root: docs/           # Main docs directory
  readme: README.md     # Project readme
  changelog: CHANGELOG.md

ai_context:
  - CLAUDE.md
  - GEMINI.md
  - AGENTS.md

ignore:
  - node_modules/
  - vendor/
  - .git/
  - dist/
  - build/

priorities:
  high:
    - docs/reference/
    - docs/architecture/
  low:
    - docs/examples/

authoritative_sources:
  architecture: docs/reference/architecture.md
  api: docs/reference/api.md
  setup: docs/getting-started.md
```

## Duplication Strategy

### Identification

Look for patterns of duplication:
- Same code examples in multiple files
- Repeated explanations of concepts
- Setup instructions duplicated across guides
- Configuration snippets copied to multiple docs

### Resolution

1. **Choose authoritative source:**
   - Most detailed version
   - Most frequently updated
   - Most logically placed

2. **Consolidate:**
   - Keep full content in authoritative file
   - Replace duplicates with cross-references: `See [Topic Guide](path/to/guide.md)`
   - Or use includes if supported: `<!-- include: shared/setup.md -->`

3. **Exception:** AI context files (CLAUDE.md, etc.) may intentionally duplicate for self-containment

## Link Fixing Strategy

```bash
# Find all markdown links
grep -rEo '\[([^\]]+)\]\(([^)]+\.md)\)' . --include="*.md" 2>/dev/null

# For each broken link:
# 1. Check if target file exists
# 2. If not, search for similar filename in project
# 3. If found moved, update path
# 4. If truly deleted, remove link or redirect to related content
```

## Common Tasks

| Task | Command / Action |
|------|-----------------|
| Full doc update | `/doc-sanitizer --dry-run` then `/doc-sanitizer` |
| Check doc structure | `/doc-sanitizer --discover` |
| Fix links only | `/doc-sanitizer --fix-links` |
| Update single file | `/doc-sanitizer --target path/to/file.md` |
| Find duplicates | `/doc-sanitizer --find-duplicates` |
| Recent changes only | `/doc-sanitizer --since "7 days ago"` |

## Output

The skill produces:
- Updated documentation files
- Summary of changes made
- List of remaining issues (if any)
- Recommendations for manual review

## Notes

- This skill is project-agnostic and works with any codebase
- Always use `--dry-run` first for significant updates
- AI context files are updated conservatively to preserve self-containment
- The skill respects `.gitignore` patterns when discovering files
