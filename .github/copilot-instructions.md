# Copilot Instructions

## Purpose

This repository is a fork of [google/jpegli](https://github.com/google/jpegli).  
Its sole purpose is to provide **Windows release builds** (x86 and x64 static binaries) that are not published by the upstream project. Build artifacts are automatically attached to GitHub Releases via the CI pipeline.

---

## Syncing with Upstream

The upstream remote is `https://github.com/google/jpegli.git` (alias: `upstream`).

To pull in new upstream commits while keeping local changes intact:

```bash
git fetch upstream
git rebase upstream/main
# Resolve any conflicts (see rules below), then:
git push --force-with-lease origin main
```

Create a backup branch before rebasing if unsure:

```bash
git checkout -b backup-main
git checkout main
git rebase upstream/main
```

---

## Workflow Folder Rules

> **Critical:** The `.github/workflows/` folder must follow these rules when syncing upstream.

| File | Rule |
|---|---|
| All non-release workflow files | **Keep upstream's version unchanged.** If upstream deletes them, delete them here too. Do not carry over local modifications to these files. |
| `release.yaml` | **Merge carefully** — see section below. |

When upstream deletes workflow files (e.g. `build_test.yml`, `codeql.yml`), accept the deletion with `git rm`.  
Do **not** re-add or preserve upstream CI workflows that upstream has intentionally removed.

---

## Release Pipeline (`release.yaml`)

`release.yaml` is the **only** workflow file that should differ from upstream. When merging upstream changes into this file, apply the following rules:

### What to keep from this fork (do not accept upstream overrides)

- The `on.push.paths-ignore` filter (ignores `**.md` and `AUTHORS` to avoid spurious runs).
- The `windows_build` job must remain **enabled** (`if` condition must not include `&& false`).
- The Windows build matrix must contain **only** these two triplets:
  ```yaml
  - triplet: x86-windows-static
    arch: '-A Win32'
  - triplet: x64-windows-static
    arch: '-A x64'
  ```
- `VCPKG_VERSION` — keep the most recent version defined in this fork.
- `publish_release_assets` must only depend on `windows_build` (not on any Linux jobs).

### What to accept from upstream

- Pinned action SHA updates (e.g. `actions/checkout`, `actions/cache`, `actions/upload-artifact`).
- New steps or flags inside the `windows_build` job that upstream adds.
- Any changes outside the `windows_build` and `publish_release_assets` jobs can be accepted, but Linux jobs must remain disabled (`if: false`).

### Linux jobs

Both Linux jobs must stay **disabled**:

```yaml
ubuntu_static_x86_64:
  if: false

release_ubuntu_pkg:
  if: false
```

---

## Generating a Changelog

Generate a changelog between the previous and new tag, then update the release notes on GitHub.

1. Get all commits between the two tags, excluding dependency bumps:
   ```bash
   git --no-pager log v0.0.PREV..v0.0.NEW --pretty=format:"%h %s" | grep -v "^.*Bump "
   ```

2. Organize the output into these categories and write to a markdown file (e.g. `notes.md`):
   - **🪟 Fork changes (Windows CI)** — any commits touching `.github/workflows/`, `README`, or vcpkg version
   - **🐛 Bug fixes (upstream)** — commits starting with `fix:` or `Fix`, mark Windows-relevant ones with ✅
   - **✨ Improvements (upstream)** — feature additions and enhancements
   - **🔧 Dependency & action bumps (upstream)** — version range for each bumped action/library

3. Upload the notes to the GitHub release:
   ```bash
   gh release edit v0.0.NEW --repo dhcgn/jpegli_release --notes-file notes.md
   ```

---

## Creating a Release

1. Create and push a tag:
   ```bash
   git tag v0.0.YYYY-MM-DD
   git push origin v0.0.YYYY-MM-DD
   ```

2. Publish a GitHub Release using the tag (via UI or CLI):
   ```bash
   gh release create v0.0.YYYY-MM-DD --repo dhcgn/jpegli_release \
     --title "v0.0.YYYY-MM-DD" \
     --notes "Windows static builds based on jpegli upstream."
   ```

3. The `release.yaml` workflow triggers on the published release event, builds `x86-windows-static` and `x64-windows-static`, and the `publish_release_assets` job attaches the `.7z` and `.zip` artifacts to the release page automatically.
