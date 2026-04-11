# CLAUDE.md

## Project overview

This is the `heretic-lazy-shot-deps` repo -- a public GitHub repo that builds and publishes prebuilt native C/C++ dependencies (Tesseract OCR + Leptonica + transitive libs) as GitHub Release assets. The consumer is the private `heretic-lazy-shot` repo which downloads these assets in CI instead of compiling from source.

## Repo structure

```
.github/workflows/build-tesseract-windows.yml  # Builds Tesseract via vcpkg on Windows, publishes tar.zst as a release
versions.txt                                    # Pinned tesseract version + vcpkg commit
README.md                                       # Tech overview, commands, CI integration docs
LICENSE                                         # MIT (build scripts only)
```

## Key concepts

- **Trigger:** workflow runs on tag push matching `tesseract-*-vcpkg` or via `workflow_dispatch`
- **Triplet:** `x64-windows-static-md` -- static libs, dynamic MSVC CRT (`/MD`), matches Rust default ABI
- **Output:** `tesseract-windows-x64-static-md.tar.zst` containing headers, static `.lib` files, debug libs, vcpkg metadata, and a MANIFEST.txt
- **Versioning:** all versions pinned in `versions.txt` (tesseract version + vcpkg commit hash)
- **Release tag format:** `tesseract-{version}-vcpkg` (e.g., `tesseract-5.5.0-vcpkg`)

## Common commands

```bash
# Trigger a build (push a tag)
git tag tesseract-5.5.0-vcpkg && git push origin tesseract-5.5.0-vcpkg

# Manual workflow dispatch
gh workflow run build-tesseract-windows.yml --repo giglabo/heretic-lazy-shot-deps

# Check workflow runs
gh run list --repo giglabo/heretic-lazy-shot-deps

# List releases
gh release list --repo giglabo/heretic-lazy-shot-deps
```

## Guidelines

- Do not commit build artifacts (`.tar.zst`, `.tar.gz`, `.zip`) -- they are in `.gitignore`
- `versions.txt` is the source of truth for what gets built -- always update it before tagging
- The workflow is expected to run ~once per quarter; keep it simple and reliable
- Consumer repo (`heretic-lazy-shot`) references release tags in its CI workflow -- coordinate updates
- This repo is **public** so the consumer CI can download release assets with the default `GITHUB_TOKEN` (no extra PAT or secret needed)
