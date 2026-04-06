---
name: release
description: Guide a project release checklist: verify and update the changelog (date-based), clean up debug code, update documentation, and brainstorm blog post topics. Use this skill whenever the user says "release", "ship", "cut a release", "prepare a release", or wants to go through a release checklist. Trigger even if they only mention one part of the process.
---

# Release Skill

You are a thorough release manager. Walk the user through each phase below in order, pausing for confirmation before making any changes.

---

## Phase 1 — Changelog Review

The goal is to make sure the changelog accurately reflects everything in this release.

1. Run `git log --oneline` to find the date of the last changelog entry. If no `CHANGELOG.md` exists yet, use all commits.
2. Run `git log --oneline --since="<last entry date>"` to list commits since the last release.
3. Group commits by type:
   - **Added** — new functionality
   - **Fixed** — bug fixes
   - **Changed** — behavior changes, refactors, updates
4. Check whether `CHANGELOG.md` already has an entry dated today or covering these commits.
   - If yes and it looks accurate: confirm with the user and move on.
   - If missing or stale: draft a new entry and show it to the user for approval before writing it.

**Changelog format** — date-based, no version numbers:
```
## YYYY-MM-DD

### Added
- New feature description

### Fixed
- Bug fix description

### Changed
- Behavior change description
```

Add the new entry at the top of `CHANGELOG.md`, below any title/header line. Do not proceed until the changelog is confirmed.

---

## Phase 2 — Debug Code Cleanup

Search for common debug artifacts that shouldn't ship. Skip `node_modules/`, `vendor/`, `.git/`.

| Language | Patterns |
|----------|----------|
| JavaScript/TypeScript | `console.log`, `console.error`, `console.warn`, `debugger;` |
| PHP | `var_dump(`, `print_r(`, `error_log(`, `dd(`, `dump(`, `ray(` |
| Any | `// TODO`, `// FIXME`, `// HACK`, `# TODO`, `# FIXME` |

Group findings by file and show them. For each group, ask: **remove all**, **keep all**, or **review individually**. Apply approved removals.

If nothing is found, note it and move on.

---

## Phase 3 — Documentation Update

Review the changelog and git diff for anything user-facing:
- New features or user-visible behavior
- New settings, options, or configuration keys
- New filters, hooks, or extensibility points
- Changed or removed behavior

If any of the above: invoke the `project-documentation` skill to create or update documentation for those features.

If the release contains only fixes or internal changes with no user-facing impact: note that no documentation update is needed and move on.

---

## Phase 4 — Blog Post Ideas

Review the new features in this release. Suggest 3–5 blog post topic ideas — each with a title and a one-line angle explaining why it's useful or interesting to the reader.

```
1. **[Title]** — [Who it helps and why they'd care]
2. ...
```

Ask the user if they'd like any drafted now.

---

## Release Checklist Summary

After completing all phases, print a summary:

```
✓ Changelog updated (YYYY-MM-DD)
✓ Debug code reviewed
✓ Documentation: <updated / no changes needed>
✓ Blog ideas: <N suggested>
```
