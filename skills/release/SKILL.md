---
name: release
description: "Guide a project release checklist: verify and update the changelog (date-based), clean up debug code, update documentation, and brainstorm blog post topics. Use this skill whenever the user says \"release\", \"ship\", \"cut a release\", \"prepare a release\", or wants to go through a release checklist. Trigger even if they only mention one part of the process."
---

# Release Skill

You are a thorough release manager. Walk the user through each phase below in order, pausing for confirmation before making any changes.

---

## Phase 1 — Changelog Review

The goal is to make sure the changelog accurately reflects everything in this release.

1. Run `git log --oneline` to find the date of the last changelog entry. If no `CHANGELOG.md` exists yet, use all commits.
2. Run `git log --oneline --since="<last entry date>"` to list commits since the last release.
3. Run `git diff <last-release-commit>..HEAD` to review the actual code changes. Use the diff as the source of truth — commit messages can be vague or incomplete. Cross-reference both to build an accurate picture of what changed.
4. For each commit, check whether it references a PR. Squash-merged commits usually end with `(#1234)` in the message — that's the PR number. If a commit doesn't show one and `gh` is available, you can try `gh pr list --search "<sha>" --state merged --json number` to look it up, but don't spend a lot of effort chasing PRs that clearly aren't there (e.g. direct commits to the branch). It's fine for a changelog entry to have no PR reference.
5. Group changes by type:
   - **Added** — new functionality that didn't exist before
   - **Fixed** — bugs fixed in **pre-existing** features only. If a bug was introduced and fixed within this same release cycle (e.g., caught during development of a new feature), do not list it under Fixed — fold it into the relevant Added entry or omit it entirely.
   - **Changed** — behavior changes, refactors, or updates to existing functionality
6. Check whether `CHANGELOG.md` already has an entry dated today or covering these commits.
   - If yes and it looks accurate: confirm with the user and move on.
   - If missing or stale: draft a new entry and show it to the user for approval before writing it.

**Changelog format** — date-based, no version numbers. When a line item has a known PR number, append it in parentheses as a markdown link to that PR (e.g. `(#1234)` linking to `<repo-url>/pull/1234`); get `<repo-url>` from `git remote get-url origin`. Omit the parenthetical entirely if there's no PR to reference — don't write `(#N/A)` or similar filler:
```
## YYYY-MM-DD

### Added
- **Short title**: one or two sentence description of what it does and why it matters. ([#1234](https://github.com/org/repo/pull/1234))

### Fixed
- **Short title**: one sentence describing the bug and its user-visible impact.

### Changed
- **Short title**: one sentence describing the behavior change. ([#1235](https://github.com/org/repo/pull/1235))

### Plugin Updates
- **Plugin Name**: vX.Y.Z → vA.B.C
```

Writing entries — keep every bullet at the "what changed for a reader" altitude, not the "how it was implemented" altitude:
- One to two sentences per bullet. If a description needs a third sentence, it's usually two entries or too much implementation detail — cut it down rather than let it grow.
- Do not mention function/method/class names, filter or hook names, CSS classes, HTML attributes, or file paths. Those belong in code comments or the PR, not the changelog. Exception: hooks/filters that are genuinely new extensibility points for other developers can be named once, briefly — not enumerated.
- Describe the effect ("editors can now add a French URL to links") not the mechanism ("added `data-fr-url` attribute and four new filters").
- Third-party plugin version bumps never get their own Added/Fixed/Changed narrative bullet — list them only under **Plugin Updates** as `Plugin Name: vFrom → vTo`, with no elaboration. If a plugin was added or removed entirely (not just updated), that still goes under Added/Changed with a one-line reason.
- Before finalizing, reread each bullet and ask: "would a non-engineer stakeholder skimming this understand what changed?" If not, simplify it.

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
