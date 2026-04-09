# heretic-lazy-shot-deps

Prebuilt native dependencies for [heretic-lazy-shot](https://github.com/nicedoc/heretic-lazy-shot), published as GitHub Release assets.

## Architecture

This repo uses a GitHub Actions workflow to compile native C/C++ libraries via [vcpkg](https://github.com/microsoft/vcpkg) on Windows runners, then packages the result (headers + static libraries + metadata) into a compressed tarball (`tar.zst`) and publishes it as a GitHub Release asset.

Downstream CI workflows download the prebuilt tarball instead of compiling from source, reducing build times from ~20 minutes to under 30 seconds.

```
heretic-lazy-shot-deps                    Consumer CI
┌────────────────────────┐               ┌────────────────────────┐
│ GitHub Actions         │               │ GitHub Actions         │
│                        │               │                        │
│ vcpkg compile          │  Release      │ gh release download    │
│ (~20 min, rare)       ─┼──asset───────>│ (<30 sec, every build) │
│                        │               │                        │
│ Publish tar.zst        │               │ Extract & link         │
└────────────────────────┘               └────────────────────────┘
```

**Key design choice:** public GitHub repos get unlimited Releases storage and download bandwidth. By isolating the expensive native compilation into a separate public repo, the main repo's Actions cache quota is unaffected.

## What's Built

| Library | Version | License | Triplet |
|---|---|---|---|
| [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) | Pinned in `versions.txt` | Apache-2.0 | `x64-windows-static-md` |
| [Leptonica](https://github.com/DanBloomberg/leptonica) | Transitive via vcpkg | BSD-2-Clause | `x64-windows-static-md` |
| libpng, libjpeg-turbo, libtiff, zlib, libwebp, giflib, openjpeg | Transitive via vcpkg | Various OSS | `x64-windows-static-md` |

The `x64-windows-static-md` vcpkg triplet produces **static** libraries (`.lib` archives) linked against the **dynamic** MSVC runtime (`/MD`). This matches Rust's default Windows ABI.

### Tarball Contents

```
tesseract-windows-x64-static-md.tar.zst
├── include/          # Public C/C++ headers (tesseract/, leptonica/)
├── lib/              # Release static libraries (.lib)
├── debug/lib/        # Debug static libraries (.lib)
├── vcpkg-info/       # vcpkg port metadata (for reproducibility)
└── MANIFEST.txt      # Version pins, build date, runner info
```

## Version Pinning

All versions are locked in [`versions.txt`](versions.txt):

```
tesseract=5.5.0
vcpkg_commit=2024.11.16
```

- **`tesseract`** -- the vcpkg port version to install.
- **`vcpkg_commit`** -- the exact vcpkg repository commit, ensuring identical port recipes and transitive dependency versions across builds.

## Workflow Triggers

The build workflow (`.github/workflows/build-tesseract-windows.yml`) runs on:

- **Push** to `versions.txt` on `main` -- automated trigger when bumping versions.
- **`workflow_dispatch`** -- manual trigger with an optional version override.

Expected frequency: once per quarter or less.

## Release Tags

Format: `tesseract-{version}-vcpkg` (e.g., `tesseract-5.5.0-vcpkg`).

One release per Tesseract version. Re-running for the same version replaces the existing release (idempotent).

## Consuming in CI

```yaml
- name: Download prebuilt Tesseract
  shell: pwsh
  run: |
    $tag = "tesseract-5.5.0-vcpkg"
    $dest = "$env:RUNNER_TEMP\tesseract"
    New-Item -ItemType Directory -Force $dest
    gh release download $tag `
      -R owner/heretic-lazy-shot-deps `
      -p 'tesseract-windows-x64-static-md.tar.zst' `
      -D $dest
    tar --zstd -xf "$dest\*.tar.zst" -C $dest
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

- name: Configure tesseract-sys env
  shell: pwsh
  run: |
    $dest = "$env:RUNNER_TEMP\tesseract"
    echo "TESSERACT_INCLUDE_PATHS=$dest\include" >> $env:GITHUB_ENV
    echo "TESSERACT_LINK_PATHS=$dest\lib" >> $env:GITHUB_ENV
    echo "TESSERACT_LINK_LIBS=tesseract55,leptonica-1.84.1,libpng16,jpeg,tiff,zlib,libwebp,libwebpmux,libsharpyuv,gif,openjp2" >> $env:GITHUB_ENV
```

> `GITHUB_TOKEN` has read access to public repos by default -- no additional PAT needed.

## Local Development (Windows)

Developers can skip vcpkg and use the prebuilt tarball directly:

```powershell
gh release download tesseract-5.5.0-vcpkg -R owner/heretic-lazy-shot-deps -p '*windows*' -D C:\tesseract-prebuilt
tar --zstd -xf C:\tesseract-prebuilt\*.tar.zst -C C:\tesseract-prebuilt

[System.Environment]::SetEnvironmentVariable("TESSERACT_INCLUDE_PATHS", "C:\tesseract-prebuilt\include", "User")
[System.Environment]::SetEnvironmentVariable("TESSERACT_LINK_PATHS", "C:\tesseract-prebuilt\lib", "User")
[System.Environment]::SetEnvironmentVariable("TESSERACT_LINK_LIBS", "tesseract55,leptonica-1.84.1,libpng16,jpeg,tiff,zlib,libwebp,libwebpmux,libsharpyuv,gif,openjp2", "User")
```

Alternatively, build from source via vcpkg:

```powershell
git clone https://github.com/microsoft/vcpkg.git C:\vcpkg
C:\vcpkg\bootstrap-vcpkg.bat -disableMetrics
C:\vcpkg\vcpkg.exe install tesseract:x64-windows-static-md
```

Both approaches also require [LLVM](https://github.com/llvm/llvm-project/releases) for `bindgen` (`libclang`):

```powershell
winget install LLVM.LLVM
[System.Environment]::SetEnvironmentVariable("LIBCLANG_PATH", "C:\Program Files\LLVM\bin", "User")
```

## Version Bump Procedure

1. Edit `versions.txt` with the new Tesseract version and/or vcpkg commit.
2. Push to `main` (or trigger `workflow_dispatch`).
3. Wait ~20-30 min for the build.
4. Update the release tag reference in the consumer repo's CI workflow.

## License

Build scripts in this repo are [MIT](LICENSE).

Produced artifacts contain libraries under their respective licenses:
Tesseract (Apache-2.0), Leptonica (BSD-2-Clause), libpng (PNG Reference Library License),
libjpeg-turbo (IJG/BSD), libtiff (MIT), zlib (zlib License), libwebp (BSD-3-Clause),
giflib (MIT), openjpeg (BSD-2-Clause).
