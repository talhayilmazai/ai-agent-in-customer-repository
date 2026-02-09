---
name: changelog-manager
description: Analyzes git diffs or user descriptions to generate semantic changelog entries and updates the CHANGELOG.md file.
---

## 🎯 Goal
Analyze recent code modifications (via git diff or user description) to generate concise changelog entries and automatically update the /CHANGELOG.md file. Build the format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and make sure that project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).


## 🧠 Analysis Strategy
Detect Changes: If the user asks to summarize "recent changes," run git diff and git diff --cached to identify modified files and logic.

Classify: Categorize the change into one of the standard types (Added, Changed, Fixed, etc.).

Summarize: Draft a 1-2 sentence description focusing on the "what" and "why."

Identify Impact: List all affected files (e.g., rocks_extension/module/file.py).

Update File: Use file editing tools to insert the entry into /CHANGELOG.md under the ## [Released] section.

## Changelog Entry Format

### Opening Sentence of Document
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

### Standard Entry

```markdown
### [Type] Brief Title  - YYYY-MM-DD

Date: YYYY-MM-DD

Description of change in 1-2 sentences explaining what was added/modified and why.

**Affected:** `module/file.py`, `config.yml`
```

### Entry Types

| Type | Emoji | Use For |
|------|-------|---------|
| Added | ✨ | New features, functions, files |
| Changed | 🔄 | Modifications to existing behavior |
| Fixed | 🐛 | Bug fixes |
| Removed | 🗑️ | Deleted code, deprecated features |
| Security | 🔒 | Security-related changes |
| Performance | ⚡ | Optimizations |
| Docs | 📚 | Documentation updates |
| Refactor | ♻️ | Code restructuring without behavior change |

---

## Examples

### ✨ Added: New allocation smoothing module

Added `allocation_smoothing.py` to handle gradual transitions between forecast versions. This prevents abrupt changes in demand predictions that could disrupt downstream planning.

**Affected:** `rocks_extension/noob/allocation_smoothing.py`

**Date:** 2025-11-28

---

### 🔄 Changed: Updated schema for customer_forecast table

Modified the customer_forecast schema to include `confidence_interval` column. Existing readers will continue to work; new column is nullable.

**Affected:** `config_schema_tables.yml`, `forecast/writer.py`

**Date:** 2025-11-28

---

### 🐛 Fixed: Date parsing in historical data loader

Fixed timezone handling in `try_read_latest()` that caused incorrect date filtering for UTC+0 regions.

**Affected:** `rocks_extension/noob/data_loader.py:L45-52`

**Date:** 2025-11-28

---

### 🗑️ Removed: Deprecated legacy_forecast endpoint

Removed the deprecated `legacy_forecast` table writer. All consumers should now use `forecast_v2`.

**Affected:** `rocks_extension/noob/legacy/`, `config_metadata.yml`

---

## Quick Templates

### For Feature Additions
```
✨ Added [feature name] - [what it does in one sentence]. [why it was needed] - [date].
```

### For Bug Fixes
```
🐛 Fixed [issue] - [what was wrong] → [what now works correctly] - [date].
```

### For Config Changes
```
🔄 Changed [config/setting] - Updated [parameter] from [old] to [new] to [reason] - [date].
```

### For Refactoring
```
♻️ Refactored [component] - Restructured [what] for [benefit: readability/performance/maintainability] - [date].
```

---

## Generating from Git Diff
1. Review the git diff
2. Identify the type of change (add/modify/delete)
3. Summarize the functional impact
4. Format as changelog entry
5. **Edit CHANGELOG.md to insert the new entry**
  - Ensure each entry includes a `Date: YYYY-MM-DD` line or the date appended to the title (e.g. `### ✨ Added: Title - 2025-11-28`).

---

## Important: File Update Behavior

**Always edit the CHANGELOG.md file directly.** 

When generating a changelog entry:
1. Use the `replace_string_in_file` or `edit_file` tool to update `CHANGELOG.md`
2. Insert new entries under the appropriate heading in `## [Released]`:
   - `### ✨ Added` - for new features
   - `### 🔄 Changed` - for modifications
   - `### 🐛 Fixed` - for bug fixes
   - `### 🗑️ Removed` - for removals
3. If the heading doesn't exist, create it under `## [Released]`
4. Confirm to the user that the file was updated

---

## Best Practices

1. **Be specific** - "Updated forecast logic" → "Updated ensemble weights to favor recent models"
2. **Focus on impact** - What does this change mean for users/other modules?
3. **Keep it brief** - 1-2 sentences max for the description
4. **Link related changes** - Group related file changes under one entry
5. **Note breaking changes** - Flag anything that requires updates elsewhere
6. **Add dates** - Include date of change for historical context
7. **Always update the file** - Never just print the entry; always edit CHANGELOG.md

---

## File Location

The changelog file is located at: `/CHANGELOG.md` (workspace root)

Structure:
```markdown
# Changelog

## [Released]

### ✨ Added
- New entries go here...

### 🔄 Changed
- Modified entries go here...

### 🐛 Fixed
- Bug fix entries go here...

## [Released]
- Previous releases...
```