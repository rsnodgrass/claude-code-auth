---
name: release
description: Cut a new tagged release of cc-auth — bump __version__, generate release notes from git log, commit, push, tag, and publish a GitHub Release. Use when the user says "release", "ship a release", "cut vX.Y.Z", or "tag a new version" in this repo.
---

# Release cc-auth

End-to-end skill for shipping a new cc-auth version. The tool's update check
reads GitHub Releases (with a Tags fallback), so a release that skips either
of those is invisible to users.

## Preconditions

- Working tree is clean (`git status` shows no unstaged/uncommitted changes).
- On `main`, up to date with `origin/main`.
- `gh` CLI installed and authenticated (`gh auth status`).

If any precondition fails, stop and report — don't try to fix it silently.

## Steps

### 1. Determine the new version

- Read the current `__version__` from `cc-auth` (top of file).
- Read commits since the most recent tag:
  `git log $(git describe --tags --abbrev=0 2>/dev/null || git rev-list --max-parents=0 HEAD)..HEAD --oneline`.
- Propose a bump using conventional-commit prefixes:
  - `fix:` / `chore:` / `docs:` only → patch (Z)
  - any `feat:` → minor (Y), reset Z to 0
  - any `BREAKING CHANGE` / `!:` → major (X), reset Y and Z to 0
- Show the proposed bump and the commit list to the user; confirm before continuing.

### 2. Bump `__version__`

Edit `cc-auth` to replace `__version__ = "X.Y.Z"` with the agreed value.

### 3. Compose release notes

Group the commits into sections — order them as **Added** → **Changed** → **Fixed** → **Removed** → **Internal**. Map prefixes:

- `feat:` → Added
- `fix:` → Fixed
- `refactor:` / `perf:` / non-breaking `chore:` user-visible → Changed
- `BREAKING` → top of Changed with a ⚠ marker
- `docs:` / `test:` / `chore:` (build, deps, ci) → Internal (collapsed)
- The version-bump commit itself is omitted.

Rewrite each line as a present-tense user-facing sentence, not the raw commit subject. Drop noise (typo fixes, whitespace).

Write the result to `.release-notes-vX.Y.Z.md` (gitignored — see step 7) and show it to the user. Iterate until they approve.

### 4. Commit the bump

```
git add cc-auth
git commit -m "chore(release): vX.Y.Z"
git push
```

### 5. Tag and publish

```
git tag -a vX.Y.Z -m "vX.Y.Z"
git push --tags
gh release create vX.Y.Z \
  --title "vX.Y.Z" \
  --notes-file .release-notes-vX.Y.Z.md
```

If `gh release create` fails because a release with that tag already exists, stop and ask the user — don't `--clobber` without permission.

### 6. Verify

- `./cc-auth version` should report `up to date` against the new tag.
- Optionally clear the update-check cache (`rm ~/.claude/cc-auth/update-check.json`) before re-running so the test exercises the network path.

### 7. Clean up

- Delete `.release-notes-vX.Y.Z.md` from the working tree.
- Make sure `.gitignore` contains `.release-notes-v*.md` (add and commit if missing — separate small commit, not part of the release commit).

## Rules

- Never `--no-verify` to bypass hooks. If a hook fails, fix the cause.
- Never force-push tags. If you tagged the wrong commit, ask the user before deleting.
- Don't reference Claude / Claude Code in commit messages or release notes.
- If the user wants a pre-release (`-rc1`, `-beta1`), pass `--prerelease` to `gh release create`.
