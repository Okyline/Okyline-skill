# Release Process

This document describes how to publish a new release of the Okyline skill.
Claude reads this file whenever the user asks to prepare a release.

## Versioning (SemVer)

- **MAJOR** (`X.0.0`): breaking changes — Okyline spec alignment bumps, removed/renamed directives, behavior changes that invalidate existing schemas.
- **MINOR** (`x.Y.0`): new features — new directives, new built-in formats, significant additions to `SKILL.md` or `references/`.
- **PATCH** (`x.y.Z`): fixes and clarifications — typos, wording fixes, small doc improvements, no behavior change.

If unsure which level to bump, ask the user.

## Preflight checks

Before doing anything:

1. `git status` — working tree clean? All intended changes committed?
2. Current branch is `main` and up-to-date with `origin/main`?
3. The previous tag is reachable: `git describe --tags --abbrev=0`

If any check fails, surface it to the user before proceeding.

## Steps

1. **Confirm the version number** with the user (e.g. `v2.1.0`).

2. **Analyze changes since the previous tag**:
   ```bash
   PREV=$(git describe --tags --abbrev=0)
   git log $PREV..HEAD --pretty=format:'%h %s'
   git diff $PREV..HEAD --stat
   ```
   Then read the meaningful diffs (focus on `SKILL.md`, `references/*.md`, `README.md`).

3. **Draft release notes** following the template below. Show them to the user for validation. Do not proceed without explicit approval.

4. **Create the tag + GitHub Release** in one command:
   ```bash
   gh release create vX.Y.Z \
     --target main \
     --title "vX.Y.Z" \
     --notes "$(cat <<'EOF'
   <validated notes here>
   EOF
   )"
   ```
   This creates the tag on the remote, which triggers `.github/workflows/release.yml`.

5. **Verify the workflow built and attached the ZIP**:
   ```bash
   gh run list --workflow=release.yml --limit 1
   gh release view vX.Y.Z
   ```
   The release should now contain `okyline-skill-vX.Y.Z.zip`.

## Release notes template

- **Start directly with sections.** No top-level heading like `## Release Notes`.
- Sections (omit empty ones, keep this order): `### Added`, `### Changed`, `### Fixed`, `### Removed`, `### Documentation`.
- One short sentence per bullet, present tense, user-focused (what the user gains or notices).
- **Skip purely internal changes** (CI tweaks, `.gitignore`, build scripts) unless they affect users.
- Mention Okyline spec version alignment when applicable (e.g. "Aligned with Okyline spec v1.5.0").

### Example

```markdown
### Added
- New `$forbiddenIf` directive support with full conditional expression syntax.
- Built-in `$Phone` format for international phone number validation.

### Changed
- Aligned with Okyline spec v1.5.0 — `$compute` expressions now support array reductions.

### Fixed
- Clarified nullable vs optional distinction in `references/syntax-reference.md`.
```

## Notes

- The workflow only builds and attaches the ZIP — it does **not** generate release notes. Notes come from the `gh release create` call.
- The ZIP contains a top-level `okyline/` directory so users can extract directly into `~/.claude/skills/`.
- `softprops/action-gh-release` updates the existing release (the one `gh release create` just created) rather than creating a duplicate.
- If the workflow fails, the release exists but has no ZIP — re-run via `gh run rerun <run-id>` after fixing.
