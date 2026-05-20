---
name: release-write
description: |
  Write release notes for a new version and cut the release.
  Use when the user wants to ship a tagged release after merging
  the release CHANGELOG commit. Produces a clean GitHub Release
  body sourced from CHANGELOG.md, then tags + publishes.
allowed-tools: Bash(git tag:*), Bash(git push:*), Bash(git log:*), Bash(gh release create:*), Bash(gh release view:*), Bash(sed:*), Bash(awk:*), Bash(cat:*), Read, Write
---

## Context

This repo's release path is half-manual:

| Step | Trigger | Workflow |
|---|---|---|
| Push `chore(release): X.Y.Z` to main | `push.branches: main` | `release.yml` → Docker build/push |
| Push tag `vX.Y.Z` | (none) | nothing |
| Create GitHub Release on the tag | `release.types: [published]` | `publish.yml` → PyPI + MCP Registry |

**Creating the GitHub Release is what actually triggers PyPI publish.** It is not optional. CHANGELOG.md is manual (per CLAUDE.md). This skill writes the GitHub Release body so users opening the Releases tab or the PyPI project page see polished notes, not raw commit history.

## Prerequisites

- The release commit (`chore(release): X.Y.Z`) is already on `origin/main` — i.e. CHANGELOG.md has been renamed from `[Unreleased]` to `[X.Y.Z] — YYYY-MM-DD`, committed, and pushed.
- `make can-release` is green.
- You are on `main`, up to date with `origin/main`.

If any prereq is missing, stop and tell the user what to do first.

## Instructions

### 1. Extract the CHANGELOG section for this version

Find the start line of `## [X.Y.Z]` and the start line of the *next* version section below it. Read everything in between, dropping the leading `## [...] — date` heading line (GitHub uses the release title instead).

```bash
# Example for v3.0.0:
awk '/^## \[3\.0\.0\]/{flag=1; next} /^## \[/{flag=0} flag' CHANGELOG.md
```

Save the extracted body to a temp file (e.g. `/tmp/release-notes-X.Y.Z.md`).

### 2. Polish for GitHub Releases UI (lightweight)

Apply these adjustments to the temp file — do NOT change the CHANGELOG.md itself:

- **Add a TL;DR callout box at the top** if the release ships measured metrics or a headline number. Use GitHub's blockquote callout syntax:
  ```markdown
  > [!NOTE]
  > **v3.0.0 in one line:** ~3,300-3,500 fewer tokens per session and routing accuracy 80% → 86% on Haiku 4.5.
  ```
- **Always add a "PRs in this release" section at the bottom** linking the merged PRs. Generate it from git log:
  ```bash
  git log v<PREVIOUS_VERSION>..v<CURRENT_VERSION> --grep '^Merge pull request' --pretty='- %s' | sed 's/Merge pull request #\([0-9]*\) from .*/#\1/'
  ```
  Manually verify the list before pasting — `git log` sometimes catches noise.
- **For breaking-change releases (major version bump), add a "Migration from vX.Y" section** below the standard CHANGELOG content. List each breaking change with a one-liner code-snippet showing the before/after shape. Reference the relevant CHANGELOG bullet rather than re-explaining the whole change.

Don't:
- Rewrite the CHANGELOG content into marketing prose. The CHANGELOG is the source of truth; the release notes mirror it.
- Add emojis unless the user explicitly asks for them (per project convention).
- Include co-author lines or AI attribution (per [feedback_commits.md] memory).

### 3. Show the polished notes to the user for review

Print the contents of the temp file. The user may want to tweak before publishing. Wait for explicit go-ahead before step 4.

### 4. Cut the release (only after user confirms)

```bash
# Create the tag (no automation; just a marker)
git tag vX.Y.Z
git push origin vX.Y.Z

# Create the GitHub Release using the polished notes file
# This is what triggers publish.yml → PyPI + MCP Registry
gh release create vX.Y.Z --title "vX.Y.Z" --notes-file /tmp/release-notes-X.Y.Z.md
```

Do NOT use `--generate-notes` — that uses GitHub's auto-generated PR/commit list, which is noisier than what we just polished.

Do NOT use `--draft`. The release needs to be published to trigger `publish.yml`.

### 5. Verify the release published correctly

```bash
gh release view vX.Y.Z --json url,tagName,publishedAt
```

Report the URL to the user. Then watch the PyPI publish workflow:

```bash
gh run list --workflow=publish.yml --limit=1
```

Report status. If it fails, surface the failure and stop — do not retry without user confirmation.

## Anti-patterns (don't do these)

- **Don't edit CHANGELOG.md from this skill.** The release commit is already on main; further edits create drift between the canonical changelog and the release body.
- **Don't tag without pushing the release commit first.** Tagging an older commit means the published artifact references stale CHANGELOG.
- **Don't auto-publish without showing the user the polished notes.** Even if the CHANGELOG looks fine, the polish step (TL;DR, PR list, migration guide) is judgment-based and the user should approve.
- **Don't use `--draft` then forget to publish.** `publish.yml` only fires on `release.types: [published]`, not on draft creation.
- **Don't generate "what's changed" via `--generate-notes` for a polished release.** That's GitHub's auto-listing; it's noisier and less narrative than the CHANGELOG section.

## Quality bar

The release body should answer three questions for a reader scanning the Releases tab:
1. **What changed at a glance?** — the TL;DR callout.
2. **What do I need to know to upgrade?** — the Changed (breaking) bullets + Migration section.
3. **Where did this come from?** — the PRs section.

If a reader can answer those without clicking into the CHANGELOG, the polish is sufficient.
