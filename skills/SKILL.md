---
name: doc-sanitizer
description: Update and maintain project documentation. Use when user asks to "update docs", "refresh documentation", "sync docs with code", "fix outdated docs", or mentions documentation being stale. Analyzes git history, discovers doc structure, identifies duplications, and produces current, consistent documentation.
---

# Documentation Sanitizer

A general-purpose skill for keeping project documentation current and consistent.

## TL;DR

```bash
/doc-sanitizer                  # Incremental scan (fast, recent changes)
/doc-sanitizer --full           # Full scan (thorough, all files)
/doc-sanitizer --discover       # See doc structure first
/doc-sanitizer --dry-run        # Preview changes
/doc-sanitizer --prune          # Find deletion candidates
```

## Safety Rules

**You MUST follow these rules:**

1. **ASK before deleting** - Proactively identify deletion candidates (orphaned, obsolete, redundant docs) and present them to the user with rationale, but always get confirmation before removing
2. **ALWAYS run --dry-run first** for batch operations affecting more than 3 files
3. **PRESERVE custom formatting** and styling choices made by authors
4. **ASK before consolidating** AI context files (CLAUDE.md, GEMINI.md, etc.) - they may be intentionally self-contained
5. **SHOW original content** before making significant changes so user can verify
6. **STOP and ASK** if you encounter documentation in languages you're uncertain about

## Invocation Options

| Option | Description |
|--------|-------------|
| `--full` | Full scan - analyze all documentation files (slower, thorough) |
| `--prune` | Find deletion candidates (orphaned, obsolete, redundant docs) |
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
| `--since <date>` | Only consider changes since date (e.g., `--since 2024-01-01`) |
| `--verbose` | Show detailed output during execution |

**Default behavior (no flags):** Incremental scan - only analyzes files changed in the last 7 days (or since last git tag). Use `--full` for complete analysis.

---

## Scan Modes

### Incremental Scan (Default)

**Use for:** Day-to-day documentation maintenance, quick checks, CI/CD pipelines.

Incremental scan is fast because it only examines:
- Documentation files changed since last git tag (or last 7 days if no tags)
- Source files changed in the same period (to detect staleness)
- Broken links only in changed files

```bash
# Determine incremental scope
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)
if [ -n "$LAST_TAG" ]; then
  echo "Scanning changes since tag: $LAST_TAG"
  CHANGED_DOCS=$(git diff --name-only "$LAST_TAG"..HEAD -- "*.md")
  CHANGED_CODE=$(git diff --name-only "$LAST_TAG"..HEAD | grep -v "\.md$")
else
  echo "No tags found, scanning last 7 days"
  CHANGED_DOCS=$(git diff --name-only --since="7 days ago" HEAD -- "*.md")
  CHANGED_CODE=$(git diff --name-only --since="7 days ago" HEAD | grep -v "\.md$")
fi

echo "Docs to analyze: $(echo "$CHANGED_DOCS" | wc -l | tr -d ' ')"
echo "Code changes to check: $(echo "$CHANGED_CODE" | wc -l | tr -d ' ')"
```

**Incremental scan skips:**
- Full duplicate detection across all files
- Complete terminology consistency check
- Comprehensive TOC validation
- Full image/asset verification

### Full Scan (`--full`)

**Use for:** Pre-release checks, periodic audits, initial setup, after major refactors.

Full scan examines everything:
- All documentation files in the project
- Complete broken link detection
- Full duplicate content analysis
- Comprehensive terminology check
- All images and assets

```bash
# Full scan scope
ALL_DOCS=$(find . -name "*.md" -type f \
  -not -path "./.git/*" \
  -not -path "./node_modules/*" \
  -not -path "./vendor/*" \
  -not -path "./.venv/*" \
  -not -path "./dist/*" \
  -not -path "./build/*")

echo "Full scan: $(echo "$ALL_DOCS" | wc -l | tr -d ' ') documentation files"
```

### When to Use Each Mode

| Scenario | Recommended Mode |
|----------|-----------------|
| Daily development | Incremental (default) |
| Before PR merge | Incremental |
| Before release | `--full` |
| After major refactor | `--full` |
| Initial project setup | `--full` |
| CI/CD pipeline | Incremental (fast) |
| Quarterly audit | `--full` |
| Fixing specific file | `--target <path>` |

---

## Phase 1: Discovery

**You MUST run these discovery commands FIRST before any other action.**

### Step 1.1: Find All Documentation Files

```bash
# Find all markdown files (exclude common non-doc directories)
find . -name "*.md" -type f \
  -not -path "./.git/*" \
  -not -path "./node_modules/*" \
  -not -path "./vendor/*" \
  -not -path "./.venv/*" \
  -not -path "./dist/*" \
  -not -path "./build/*" \
  | head -100
```

### Step 1.2: Check Standard Documentation Locations

```bash
# Check for common doc directories
for dir in docs documentation doc .context wiki guides; do
  [ -d "$dir" ] && echo "FOUND: $dir/" && ls -la "$dir"
done

# Check for root-level doc files
ls -la README.md CHANGELOG.md CONTRIBUTING.md LICENSE.md CODE_OF_CONDUCT.md 2>/dev/null
```

### Step 1.3: Find AI Context Files

```bash
# AI assistant context files
ls -la CLAUDE.md GEMINI.md AGENTS.md CURSOR.md COPILOT.md .cursorrules 2>/dev/null
```

### Step 1.4: Find API/Spec Documentation

```bash
# OpenAPI/Swagger specs
find . -name "openapi.*" -o -name "swagger.*" -o -name "*.openapi.json" -o -name "*.openapi.yaml" 2>/dev/null | head -20

# README files in subdirectories (often document modules)
find . -name "README.md" -not -path "./.git/*" | head -30
```

### Step 1.5: Check for Configuration

```bash
# Check for doc-sanitizer config
cat .doc-sanitizer.yaml 2>/dev/null || echo "No config found - using discovery mode"
```

### Discovery Output Format

After running discovery, output this summary:

```
## Documentation Structure

### Main Documentation
- docs/                    # [X files] Main documentation directory
- README.md               # Project readme

### AI Context Files
- CLAUDE.md               # Claude-specific context
- (none found)

### API Documentation
- docs/api/openapi.yaml   # OpenAPI spec

### Other Documentation
- CHANGELOG.md            # Version history
- CONTRIBUTING.md         # Contribution guidelines

### Configuration
- .doc-sanitizer.yaml     # (not found - using defaults)

Total: X documentation files found
```

---

## Phase 2: Analysis

**Run these commands to understand what needs updating.**

### Step 2.1: Recent Code Changes

```bash
# Commits in the last 30 days
git log --oneline --since="30 days ago" | head -30

# Files changed recently (helps identify what docs might be stale)
git diff --name-only HEAD~30..HEAD 2>/dev/null | grep -v "\.md$" | head -50
```

### Step 2.2: Documentation Changes

```bash
# Recent doc changes
git log --oneline --name-only --since="30 days ago" -- "*.md" | head -50

# Docs that haven't been updated in 90+ days
find . -name "*.md" -not -path "./.git/*" -print0 2>/dev/null | head -z -n 50 | while IFS= read -r -d '' f; do
  last_commit=$(git log -1 --format="%ci" -- "$f" 2>/dev/null | cut -d' ' -f1)
  [ -n "$last_commit" ] && echo "$last_commit $f"
done | sort | head -20
```

### Step 2.3: Broken Link Detection

```bash
# Find all internal markdown links
grep -rEoh '\[([^\]]+)\]\(([^)]+)\)' --include="*.md" . 2>/dev/null | \
  grep -E '\]\([^http][^:]*\)' | \
  head -50

# For each link, you MUST verify the target exists
```

### Step 2.4: Duplicate Content Detection

Look for these duplication patterns:
- Same code examples appearing in multiple files
- Repeated setup/installation instructions
- Identical explanations of concepts
- Copy-pasted configuration snippets

```bash
# Find similar section headers (potential duplicates)
grep -rh "^## " --include="*.md" . 2>/dev/null | sort | uniq -c | sort -rn | head -20
```

### Step 2.5: Image and Asset Verification

```bash
# Find image references
grep -rEoh '!\[([^\]]*)\]\(([^)]+)\)' --include="*.md" . 2>/dev/null | head -30

# Find HTML image tags
grep -rEoh '<img[^>]+src="([^"]+)"' --include="*.md" . 2>/dev/null | head -20
```

### Step 2.6: TOC Validation

For files with table of contents, verify:
1. All TOC entries have corresponding headers
2. All major headers appear in TOC
3. Anchor links are correctly formatted

### Step 2.7: Code Block Validation

```bash
# Find code blocks with file references
grep -rEB2 '```' --include="*.md" . 2>/dev/null | grep -E "(file:|path:|from |import )" | head -20
```

### Step 2.8: Terminology Consistency Check

Look for inconsistent usage of:
- Project name variations
- Technical terms (e.g., "config" vs "configuration")
- Command names
- Feature names

### Analysis Output Format

```
## Analysis Results

### Staleness Report
| File | Last Updated | Related Code Changed |
|------|--------------|---------------------|
| docs/api.md | 2024-01-15 | Yes (src/api/ changed 2024-03-01) |
| docs/setup.md | 2024-03-10 | No |

### Broken Links (X found)
- docs/guide.md:45 → `../api/endpoints.md` (file not found)
- README.md:12 → `docs/old-guide.md` (file not found)

### Duplicate Content (X instances)
- Installation instructions duplicated in:
  - README.md (lines 20-45)
  - docs/getting-started.md (lines 10-35)
  - CONTRIBUTING.md (lines 5-30)

### Missing Images (X found)
- docs/architecture.md:78 → `images/diagram.png` (not found)

### TOC Issues (X found)
- docs/guide.md: TOC missing entry for "## Advanced Usage"

### Terminology Inconsistencies
- "config" (5 occurrences) vs "configuration" (12 occurrences)
- "repo" (3 occurrences) vs "repository" (8 occurrences)

### Summary
- X files need updates
- X broken links
- X duplicate sections
- X missing images
```

---

## Phase 3: Planning

**Create an update plan based on analysis. Present this to the user BEFORE making changes.**

### Priority Order

Update documents in this order (general → specific):

1. **Critical** - Broken links, missing images, incorrect information
2. **High** - Technical reference docs (architecture, API)
3. **Medium** - Setup guides, getting started
4. **Low** - Concept explanations, examples
5. **Maintenance** - Index files, changelog, navigation

### Decision Trees

#### When to Update vs. Reference (Duplicates)

```
Is the content duplicated?
├── Yes
│   ├── Both files in docs/ directory?
│   │   └── Consolidate to the more detailed/authoritative file
│   ├── One is README.md?
│   │   └── Keep brief version in README, full version in docs/
│   ├── One is AI context file (CLAUDE.md, etc.)?
│   │   └── ASK USER - these may be intentionally self-contained
│   └── Different audiences (user vs developer)?
│       └── Keep both, ensure they don't contradict
└── No
    └── No action needed
```

#### When to Suggest Deletion (Pruning)

```
Should this file be deleted?
├── Is it a protected file? (README.md, LICENSE, CHANGELOG, etc.)
│   └── NO - Never suggest deletion
├── Is it an orphaned file? (no links pointing to it)
│   └── SUGGEST DELETION - "This file is not linked from anywhere"
├── Does it reference deleted features/code?
│   ├── Feature completely removed from codebase?
│   │   └── SUGGEST DELETION - "Documents removed feature X"
│   └── Feature renamed/moved?
│       └── UPDATE - Fix references to new location
├── Is it an exact duplicate of another file?
│   └── SUGGEST DELETION - "Exact duplicate of [other file]"
├── Is it effectively empty? (<50 chars of content)
│   └── SUGGEST DELETION - "File has no meaningful content"
├── Is it severely outdated? (>1 year, related code changed significantly)
│   └── SUGGEST DELETION or major rewrite - present options to user
└── Just old but still accurate?
    └── KEEP - Update timestamps, verify still accurate
```

**When suggesting deletion, always:**
1. Show the file content (or summary if large)
2. Explain WHY deletion is recommended
3. Offer alternatives (archive, update, merge)
4. Wait for explicit user confirmation

#### When to Archive vs. Delete

```
User confirmed file should be removed:
├── Has historical value? (old design decisions, deprecated API docs)
│   └── ARCHIVE - Move to docs/archive/ or similar
├── Referenced in git history discussions?
│   └── ARCHIVE - May be useful for archaeology
├── Simple/small file with no special value?
│   └── DELETE - git history preserves it anyway
└── User explicitly says "just delete it"?
    └── DELETE
```

#### Choosing Authoritative Source

When consolidating duplicates, choose the file that is:
1. Most detailed and complete
2. Most recently updated
3. In the most logical location (e.g., `docs/setup.md` over `README.md` for detailed setup)
4. Most frequently linked to

### Planning Output Format

Present this plan to the user:

```
## Update Plan

### Phase 1: Critical Fixes
1. [ ] Fix 3 broken links in docs/guide.md
2. [ ] Add missing image assets/diagram.png

### Phase 2: Content Updates
3. [ ] Update docs/api.md - API endpoints changed
4. [ ] Update docs/setup.md - New dependencies added

### Phase 3: Consolidation
5. [ ] Consolidate installation instructions
   - Keep authoritative version in: docs/getting-started.md
   - Update with reference: README.md, CONTRIBUTING.md

### Phase 4: Maintenance
6. [ ] Update TOC in docs/guide.md
7. [ ] Standardize terminology: use "configuration" consistently

### Files to Modify
- docs/guide.md (links, TOC)
- docs/api.md (content update)
- docs/setup.md (content update)
- docs/getting-started.md (consolidation target)
- README.md (add reference)
- CONTRIBUTING.md (add reference)

Proceed with this plan? [Waiting for user confirmation]
```

---

## Phase 4: Execution

**Only proceed after user confirms the plan (or if --dry-run was NOT specified and changes are minor).**

### Execution Principles

1. **Make minimal, targeted changes** - Don't rewrite entire documents
2. **Preserve structure and formatting** - Keep the author's style
3. **Update one file at a time** - Easier to review and revert
4. **Show before/after for significant changes** - Let user verify

### Task-Specific Instructions

#### Fixing Broken Links

```
1. Identify the broken link
2. Search for the target file's new location:
   find . -name "filename.md" -type f
3. If found: Update the link path
4. If not found: ASK USER what to do (remove link, point elsewhere, or create file)
```

**Example:**

Before:
```markdown
See the [API documentation](../api/old-endpoints.md) for details.
```

After:
```markdown
See the [API documentation](../reference/api.md) for details.
```

#### Updating Outdated Content

```
1. Read the current documentation
2. Read the related source code or git diff
3. Identify specific outdated sections
4. Update ONLY those sections
5. Preserve surrounding content and formatting
```

**Example:**

Before:
```markdown
## Installation

Requires Python 3.8 or higher.

```bash
pip install mypackage
```
```

After:
```markdown
## Installation

Requires Python 3.10 or higher.

```bash
pip install mypackage
```
```

#### Consolidating Duplicates

```
1. Identify the authoritative source
2. Ensure authoritative source has complete content
3. Replace duplicates with cross-references
4. Use consistent reference format
```

**Example - Converting duplicate to reference:**

Before (in README.md):
```markdown
## Installation

1. Clone the repository
2. Install dependencies with `npm install`
3. Copy `.env.example` to `.env`
4. Run `npm run setup`
5. Start with `npm start`
```

After (in README.md):
```markdown
## Installation

See the [Getting Started Guide](docs/getting-started.md#installation) for detailed installation instructions.

Quick start:
```bash
git clone <repo> && cd <repo> && npm install && npm start
```
```

#### Fixing TOC

**Example:**

Before:
```markdown
## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)

## Introduction
...

## Installation
...

## Advanced Usage    <!-- Missing from TOC! -->
...
```

After:
```markdown
## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Advanced Usage](#advanced-usage)

## Introduction
...
```

#### Standardizing Terminology

```
1. Identify the preferred term (more common, more formal, or per style guide)
2. Use find-and-replace carefully - check context
3. Don't change: code identifiers, quotes, proper nouns
```

**Example:**

Before:
```markdown
Update the config file at `~/.myapp/config.yaml`. The configuration
options are documented below. See config reference for details.
```

After:
```markdown
Update the configuration file at `~/.myapp/config.yaml`. The configuration
options are documented below. See configuration reference for details.
```

---

## Phase 5: Verification

**After making changes, verify the updates.**

### Verification Checklist

Run these checks after execution:

```bash
# 1. Verify all internal links exist
find . -name "*.md" -not -path "./.git/*" -print0 2>/dev/null | while IFS= read -r -d '' f; do
  grep -oE '\]\([^)]+\.md[^)]*\)' "$f" 2>/dev/null | while IFS= read -r link; do
    target=$(echo "$link" | sed 's/.*(\([^)#]*\).*/\1/')
    dir=$(dirname "$f")
    [ ! -f "$dir/$target" ] && [ ! -f "$target" ] && echo "BROKEN in $f: $target"
  done
done

# 2. Check for remaining TODOs
grep -rn "TODO\|FIXME\|XXX\|HACK" --include="*.md" . 2>/dev/null | head -10

# 3. Verify images exist
grep -roh '!\[[^\]]*\]([^)]*' --include="*.md" . 2>/dev/null | \
  sed 's/.*](//' | while IFS= read -r img; do
    [ ! -f "$img" ] && echo "MISSING IMAGE: $img"
  done
```

### Verification Output Format

```
## Verification Results

### Links
✓ All internal links verified (X links checked)

### Images
✓ All images verified (X images checked)

### Remaining Issues
- docs/advanced.md:123 - TODO: Add example for edge case
- docs/api.md:45 - FIXME: Verify response schema

### Changes Summary
| File | Changes |
|------|---------|
| docs/guide.md | Fixed 3 broken links, updated TOC |
| docs/api.md | Updated endpoint documentation |
| README.md | Replaced duplicate content with reference |

### Recommendations
1. Consider adding a diagram to docs/architecture.md
2. The API documentation could benefit from more examples
3. CHANGELOG.md hasn't been updated since v1.2.0
```

---

## Pruning (`--prune`)

**Use `--prune` to find deletion candidates.** This is useful for keeping documentation lean and removing clutter.

### Step 1: Find Orphaned Files

Files not linked from anywhere:

```bash
# Get all internal links
ALL_LINKS=$(grep -roh '\]([^)]*\.md' --include="*.md" . 2>/dev/null | sed 's/\](//' | sort -u)

# Find files not in any link (potential orphans)
find . -name "*.md" -not -path "./.git/*" -not -path "./node_modules/*" -print0 2>/dev/null | while IFS= read -r -d '' doc; do
  docname=$(basename "$doc")
  if ! echo "$ALL_LINKS" | grep -q "$docname"; then
    echo "ORPHANED: $doc"
  fi
done
```

### Step 2: Find Obsolete Documentation

Files referencing deleted code:

```bash
# Get deleted files from recent history and check references
git log --diff-filter=D --name-only --since="6 months ago" 2>/dev/null | \
  grep -E "\.(js|ts|py|go|rs|java|rb)$" | sort -u | while IFS= read -r deleted; do
    filename=$(basename "$deleted" | sed 's/\.[^.]*$//')
    grep -rl "$filename" --include="*.md" . 2>/dev/null | while IFS= read -r doc; do
      echo "OBSOLETE: $doc references deleted file $deleted"
    done
  done
```

### Step 3: Find Empty/Near-Empty Files

```bash
# Find files with less than 100 bytes of content
find . -name "*.md" -not -path "./.git/*" -size -100c
```

### Step 4: Find Exact Duplicates

```bash
# Find files with identical content (by hash)
find . -name "*.md" -not -path "./.git/*" -exec md5sum {} \; | \
  sort | uniq -w32 -d | cut -c35-
```

### Prune Output Format

```
## Pruning Analysis

### Deletion Candidates (X files)

#### Orphaned Files (not linked from anywhere)
| File | Last Modified | Size | Recommendation |
|------|---------------|------|----------------|
| docs/old-feature.md | 2023-06-15 | 2.3KB | Delete (orphaned) |
| docs/draft-proposal.md | 2024-01-20 | 1.1KB | Review (may be WIP) |

#### Obsolete Documentation (references deleted code)
| File | References | Recommendation |
|------|------------|----------------|
| docs/api/v1-endpoints.md | src/api/v1/ (deleted) | Delete or archive |

#### Empty/Stub Files
| File | Content | Recommendation |
|------|---------|----------------|
| docs/placeholder.md | "# TODO" only | Delete |

#### Exact Duplicates
| File | Duplicate Of | Recommendation |
|------|--------------|----------------|
| docs/setup-copy.md | docs/setup.md | Delete duplicate |

### Protected Files (skipped)
- README.md
- CHANGELOG.md
- LICENSE.md

### Summary
- X deletion candidates found
- X files protected
- Estimated cleanup: X KB

---
Confirm deletions? [List files to delete, or "none"]
```

### Prune Safety

When user confirms deletions:

1. **Show each file's content** before deleting (or first 50 lines if large)
2. **Offer archive option**: "Move to docs/archive/ instead of deleting?"
3. **Delete one at a time** with confirmation, OR
4. **Batch delete** only if user explicitly says "delete all"

**Example interaction:**

```
Found 3 deletion candidates:

1. docs/old-feature.md (orphaned, last modified 2023-06-15)
   Content preview:
   # Old Feature
   This feature was part of v1.0...

   → Delete / Archive / Skip?

2. docs/api/v1-endpoints.md (obsolete, references deleted src/api/v1/)
   → Delete / Archive / Skip?

3. docs/notes/scratch.md (empty, contains only "# Notes")
   → Delete / Archive / Skip?
```

---

## Output Templates

### Summary Report (Default Output)

Always end with this summary:

```
## Documentation Sanitizer - Summary

### Completed
- ✓ Fixed X broken links
- ✓ Updated X outdated documents
- ✓ Consolidated X duplicate sections
- ✓ Verified X images
- ✓ Fixed X TOC entries

### Files Modified
1. docs/guide.md - Fixed links, updated TOC
2. docs/api.md - Updated for v2.0 API changes
3. README.md - Consolidated installation section

### Remaining Issues (Manual Review Needed)
1. docs/advanced.md - Contains TODO items
2. images/old-screenshot.png - May need updating

### Recommendations
1. Add more code examples to docs/api.md
2. Consider creating docs/troubleshooting.md
3. Update screenshots to reflect new UI

---
Documentation last sanitized: [DATE]
Files checked: X | Updated: X | Issues found: X
```

### Dry-Run Output

When `--dry-run` is specified:

```
## Documentation Sanitizer - Dry Run

### Would Fix (X items)

#### Broken Links
1. docs/guide.md:45
   - Current: `[API Docs](../api/old.md)`
   - Would change to: `[API Docs](../reference/api.md)`

2. README.md:23
   - Current: `[Setup](docs/setup.md)`
   - Would change to: `[Setup](docs/getting-started.md)`

#### Outdated Content
3. docs/api.md
   - Section "Authentication" references deprecated OAuth flow
   - Would update to reflect new JWT authentication

#### Duplicate Consolidation
4. Installation instructions (3 locations)
   - Would keep: docs/getting-started.md (authoritative)
   - Would update: README.md (add reference)
   - Would update: CONTRIBUTING.md (add reference)

### No Changes Needed
- docs/architecture.md (up to date)
- docs/concepts.md (up to date)
- CHANGELOG.md (up to date)

---
Run without --dry-run to apply these changes.
```

---

## Optional Configuration

Projects can create `.doc-sanitizer.yaml` for customization:

```yaml
# .doc-sanitizer.yaml

# Documentation locations
docs:
  root: docs/
  readme: README.md
  changelog: CHANGELOG.md
  contributing: CONTRIBUTING.md

# AI context files (handled conservatively)
ai_context:
  - CLAUDE.md
  - GEMINI.md
  - AGENTS.md
  - CURSOR.md

# Directories to ignore
ignore:
  - node_modules/
  - vendor/
  - .venv/
  - dist/
  - build/
  - .git/

# Update priorities
priorities:
  critical:
    - docs/reference/
    - docs/api/
  high:
    - docs/guides/
    - docs/getting-started.md
  low:
    - docs/examples/
    - docs/advanced/

# Authoritative sources for topics (consolidation targets)
authoritative_sources:
  installation: docs/getting-started.md
  configuration: docs/reference/configuration.md
  api: docs/reference/api.md
  architecture: docs/architecture.md

# Terminology preferences
terminology:
  prefer:
    configuration: [config, conf]
    repository: [repo]
    documentation: [docs]
  ignore:
    - code blocks
    - urls
    - file paths

# Staleness threshold (days)
staleness_threshold: 90

# Scan mode settings
scan:
  default_mode: incremental      # "incremental" or "full"
  incremental_window: 7 days     # Fallback if no git tags found
  full_scan_triggers:            # Auto-switch to full scan for these
    - branch: main               # Full scan on main branch
    - branch: release/*          # Full scan on release branches
    - tag: v*                    # Full scan when on a version tag

# Pruning settings (deletion candidates)
prune:
  suggest_deletion:              # Proactively suggest deleting these
    - orphaned: true             # Docs not linked from anywhere
    - obsolete_features: true    # Docs referencing deleted code
    - empty_files: true          # Files with no meaningful content
    - duplicate_exact: true      # Exact duplicates of other files
  protect:                       # Never suggest deleting these
    - README.md
    - LICENSE*
    - CHANGELOG.md
    - CONTRIBUTING.md
    - CODE_OF_CONDUCT.md
    - "*.context/*"              # AI context directories
```

---

## Common Scenarios

### Scenario 1: Quick Link Fix

User says: "Fix broken links in docs"

```
1. Run: /doc-sanitizer --fix-links --dry-run
2. Show user the broken links found
3. If user approves, run: /doc-sanitizer --fix-links
4. Output summary of fixes
```

### Scenario 2: Full Documentation Audit

User says: "Update all documentation"

```
1. Run: /doc-sanitizer --discover
2. Run: /doc-sanitizer --dry-run
3. Present full plan to user
4. Wait for approval
5. Execute updates
6. Run verification
7. Output summary
```

### Scenario 3: Single File Update

User says: "Update docs/api.md"

```
1. Read the file
2. Analyze related code changes
3. Show user what needs updating
4. Make targeted updates
5. Verify links in file
6. Output changes made
```

### Scenario 4: Pre-Release Documentation Check

User says: "Check docs before release"

```
1. Run all checks: links, images, TOC, code blocks, terminology
2. Compare doc versions with code versions
3. Flag any version mismatches
4. Output comprehensive report
5. Recommend fixes before release
```

---

## Notes

- This skill is project-agnostic and works with any codebase
- Always use `--dry-run` first for significant updates
- AI context files are updated conservatively to preserve self-containment
- The skill respects `.gitignore` patterns when discovering files
- When in doubt, ASK the user rather than making assumptions
