# heretic-lazy-shot-deps

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

**Purpose:** this repository exists solely to produce and host prebuilt static libraries for Tesseract OCR (and its transitive C/C++ dependencies) on Windows. Compiling Tesseract from source via vcpkg takes ~20 minutes per CI run; downloading the prebuilt tarball takes under 30 seconds. Consumer repositories link against these static `.lib` files directly instead of rebuilding the native toolchain on every job.

## What's Built

| Library | Version | License | Triplet |
|---|---|---|---|
| [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) | Pinned in `versions.txt` | Apache-2.0 | `x64-windows-static`, `x64-windows-static-md` |
| [Leptonica](https://github.com/DanBloomberg/leptonica) | Transitive via vcpkg | BSD-2-Clause | same |
| libpng, libspng, libjpeg-turbo, libtiff, zlib (`zs.lib`), libwebp, giflib, openjpeg, libarchive, libcurl, OpenSSL, liblzma, bzip2, lz4, zstd | Transitive via vcpkg | Various OSS | same |

Both triplets produce **static** libraries (`.lib` archives). They differ in the MSVC runtime the objects are compiled for, and each is a separate asset of the same release:

| Asset | Triplet | MSVC runtime | Consumer |
|---|---|---|---|
| `tesseract-windows-x64-static.tar.zst` | `x64-windows-static` | static (`/MT`) | Rust with `-C target-feature=+crt-static`: the exe imports only system DLLs and needs no VC++ redistributable. **Used by heretic-lazy-shot since 2026-11-10.** |
| `tesseract-windows-x64-static-md.tar.zst` | `x64-windows-static-md` | dynamic (`/MD`) | Rust's default ABI; the exe imports `MSVCP140.dll`, so users need the VC++ redistributable. |

Never mix them: the linker refuses `/MT` and `/MD` C++ objects in one binary (`LNK2038 RuntimeLibrary mismatch`).

**Toolset.** The workflow runs on `windows-2022` (VS 2022, MSVC 14.44), not `windows-latest`. Static libraries link only with the same or a newer `link.exe`; libs from `windows-latest` (VS 2026, MSVC 14.51) fail on VS 2022 with `LNK1104: cannot open file 'volatileaccessu.lib'`. Build with the oldest toolset any consumer uses (heretic-lazy-shot's win10-2: VS 2022 17.14). `MANIFEST.txt` records the `msvc` version.

### Tarball Contents

```
tesseract-windows-x64-static.tar.zst   (or …-x64-static-md.tar.zst)
├── include/          # Public C/C++ headers (tesseract/, leptonica/)
├── lib/              # Release static libraries (.lib)
├── debug/lib/        # Debug static libraries (.lib)
├── vcpkg-info/       # vcpkg port metadata (for reproducibility)
└── MANIFEST.txt      # Version pins, build date, runner info
```

## Version Pinning

All versions are locked in [`versions.txt`](versions.txt):

```
tesseract=5.5.3
vcpkg_commit=2026.10.16
```

- **`tesseract`** -- the vcpkg port version to install.
- **`vcpkg_commit`** -- the exact vcpkg repository commit, ensuring identical port recipes and transitive dependency versions across builds.

Old vcpkg commits stop building: they pin MSYS2 packages (pkgconf, the autotools libiconv needs) that MSYS2 later drops from its mirrors (404). `2024.11.16` (Tesseract 5.5.0) broke that way by 2026-11; bump to a recent vcpkg release tag rather than patching around it. `2026.10.16` gives Tesseract 5.5.3 / Leptonica 1.87.0, the versions heretic-lazy-shot's macOS build uses.

## Workflow Triggers

The build workflow (`.github/workflows/build-tesseract-windows.yml`) runs on:

- **Tag push** matching `tesseract-*-vcpkg` (e.g., `tesseract-5.5.3-vcpkg`) -- create a tag to trigger a build and release.
- **`workflow_dispatch`** -- manual trigger with an optional version override.

Expected frequency: once per quarter or less.

## Release Tags

Format: `tesseract-{version}-vcpkg` (e.g., `tesseract-5.5.3-vcpkg`).

Push a tag to trigger the build. The release is created from that tag automatically.

## Consuming in CI

```yaml
- name: Download prebuilt Tesseract
  shell: pwsh
  run: |
    $tag = "tesseract-5.5.3-vcpkg"
    $dest = "$env:RUNNER_TEMP\tesseract"
    New-Item -ItemType Directory -Force $dest
    gh release download $tag `
      -R giglabo/heretic-lazy-shot-deps `
      -p 'tesseract-windows-x64-static.tar.zst' `
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
    echo "TESSERACT_LINK_LIBS=tesseract55,leptonica-1.87.0,libcurl,archive,libssl,libcrypto,lzma,bz2,lz4,zstd,libpng16,spng_static,jpeg,tiff,zs,libwebp,libwebpmux,libwebpdecoder,libwebpdemux,libsharpyuv,gif,openjp2,crypt32,ws2_32,wldap32,normaliz,advapi32,bcrypt,user32,secur32,iphlpapi,xmllite" >> $env:GITHUB_ENV
```

> `GITHUB_TOKEN` has read access to public repos by default -- no additional PAT needed.

## Local Development (Windows)

Developers can skip vcpkg and use the prebuilt tarball directly:

```powershell
gh release download tesseract-5.5.3-vcpkg -R giglabo/heretic-lazy-shot-deps -p 'tesseract-windows-x64-static.tar.zst' -D C:\tesseract-prebuilt
tar --zstd -xf C:\tesseract-prebuilt\*.tar.zst -C C:\tesseract-prebuilt

[System.Environment]::SetEnvironmentVariable("TESSERACT_INCLUDE_PATHS", "C:\tesseract-prebuilt\include", "User")
[System.Environment]::SetEnvironmentVariable("TESSERACT_LINK_PATHS", "C:\tesseract-prebuilt\lib", "User")
[System.Environment]::SetEnvironmentVariable("TESSERACT_LINK_LIBS", "tesseract55,leptonica-1.87.0,libcurl,archive,libssl,libcrypto,lzma,bz2,lz4,zstd,libpng16,spng_static,jpeg,tiff,zs,libwebp,libwebpmux,libwebpdecoder,libwebpdemux,libsharpyuv,gif,openjp2,crypt32,ws2_32,wldap32,normaliz,advapi32,bcrypt,user32,secur32,iphlpapi,xmllite", "User")
```

Alternatively, build from source via vcpkg:

```powershell
git clone https://github.com/microsoft/vcpkg.git C:\vcpkg
C:\vcpkg\bootstrap-vcpkg.bat -disableMetrics
C:\vcpkg\vcpkg.exe install tesseract:x64-windows-static      # or :x64-windows-static-md
```

Both approaches also require [LLVM](https://github.com/llvm/llvm-project/releases) for `bindgen` (`libclang`):

```powershell
winget install LLVM.LLVM
[System.Environment]::SetEnvironmentVariable("LIBCLANG_PATH", "C:\Program Files\LLVM\bin", "User")
```

## Commands

### Trigger a new build and release

```bash
# 1. Update versions if needed
vim versions.txt

# 2. Commit and push
git add versions.txt
git commit -m "Bump tesseract to 5.5.3"
git push origin main

# 3. Create and push the tag (this triggers the workflow)
git tag tesseract-5.5.3-vcpkg
git push origin tesseract-5.5.3-vcpkg
```

### Trigger manually via workflow dispatch

```bash
gh workflow run build-tesseract-windows.yml --repo giglabo/heretic-lazy-shot-deps

# With a version override
gh workflow run build-tesseract-windows.yml --repo giglabo/heretic-lazy-shot-deps \
  -f tesseract_version=5.5.3

# One triplet only (the default builds both); the other asset of the release stays as it is
gh workflow run build-tesseract-windows.yml --repo giglabo/heretic-lazy-shot-deps \
  -f triplets=x64-windows-static
```

A run uploads its assets into the release (creating it if missing) and replaces only assets of the same name.

### Check workflow status

```bash
gh run list --repo giglabo/heretic-lazy-shot-deps
gh run view <run-id> --repo giglabo/heretic-lazy-shot-deps
gh run watch <run-id> --repo giglabo/heretic-lazy-shot-deps
```

### List available releases

```bash
gh release list --repo giglabo/heretic-lazy-shot-deps
```

### Download a release asset

```bash
gh release download tesseract-5.5.3-vcpkg \
  --repo giglabo/heretic-lazy-shot-deps \
  -p 'tesseract-windows-x64-static.tar.zst' \
  -D ./tesseract

tar --zstd -xf ./tesseract/tesseract-windows-x64-static.tar.zst -C ./tesseract
```

### Delete a tag and re-trigger

```bash
# Delete remote tag
git push origin :refs/tags/tesseract-5.5.3-vcpkg
# Delete local tag
git tag -d tesseract-5.5.3-vcpkg
# Recreate and push
git tag tesseract-5.5.3-vcpkg
git push origin tesseract-5.5.3-vcpkg
```

## Version Bump Procedure

1. Edit `versions.txt` with the new Tesseract version and/or vcpkg commit.
2. Commit and push to `main`.
3. Create and push a tag: `git tag tesseract-5.5.3-vcpkg && git push origin tesseract-5.5.3-vcpkg`
4. Wait ~20-30 min for the build.
5. Update the release tag reference in the consumer repo's CI workflow.

## License

Build scripts in this repo are [MIT](LICENSE).

Produced artifacts contain libraries under their respective licenses:
Tesseract (Apache-2.0), Leptonica (BSD-2-Clause), libpng (PNG Reference Library License),
libjpeg-turbo (IJG/BSD), libtiff (MIT), zlib (zlib License), libwebp (BSD-3-Clause),
giflib (MIT), openjpeg (BSD-2-Clause).
