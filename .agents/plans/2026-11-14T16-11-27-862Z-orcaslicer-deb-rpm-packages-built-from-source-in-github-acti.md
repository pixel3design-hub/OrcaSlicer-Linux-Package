# OrcaSlicer Linux packages (deb + rpm) from source, in CI

## Goal

Repo **pixel3design-hub/OrcaSlicer-Linux-Package** (already created, empty) gains a workflow that:

1. clones upstream `OrcaSlicer/OrcaSlicer` at a release tag (default: latest, currently **v2.5.1**, published 2026-11-14),
2. compiles it on an `ubuntu-24.04` runner using upstream's own `build_linux.sh` (same path upstream CI uses),
3. wraps the resulting AppDir into `orca-slicer_<ver>_amd64.deb` + `orca-slicer-<ver>-1.x86_64.rpm`,
4. publishes both as assets of a release tagged like upstream (`v2.5.1`).

Assumptions: x86_64 only, no code signing, no arm64, builds stay on GitHub-hosted runners.

## Key findings (paths are upstream-relative)

- `build_linux.sh` — upstream's single build entry point. Relevant flags: `-u` install apt deps (needs sudo), `-d` build deps into `deps/build/destdir`, `-s` build the slicer, `-l` clang, `-L` ld.lld, `-r` skip RAM/disk pre-flight, `-g` run in an Ubuntu 24.04 container (not needed on an Ubuntu runner), `-i` make the AppImage (not needed).
- Upstream CI (`.github/workflows/build_orca.yml`, step "Build slicer and tests") runs exactly `./build_linux.sh -istrlL` after `.github/actions/apt-install-deps` (= `sudo ./build_linux.sh -ur`). We drop `-i` and `-t`.
- **`build_linux.sh -s` already emits the full installable tree at `build/package/`** (`src/dev-utils/platform/unix/build_linux_image.sh.in`, step `[9/9] Generating Linux app`): `bin/orca-slicer`, `lib/` + `lib/orca-runtime/` (bundled dependency closure), `libexec/orca-slicer-env`, `share/`, `resources/`, plus a launcher script `build/package/orca-slicer` that does `DIR=$(dirname "$(readlink -f "$0")")`. That launcher being symlink-safe means a plain `/usr/bin/orca-slicer → /opt/OrcaSlicer/orca-slicer` symlink works — no wrapper script needed.
- Which libraries are **not** bundled is decided by `scripts/appimage_lib_policy.sh` (`appimage_is_host_library`): glibc/loader, GL/EGL/vulkan/va, X11+XCB+wayland, GTK/GDK/pango/atk/glib/gio, webkit2gtk/javascriptcore, gstreamer, libsecret, fontconfig/freetype/harfbuzz, libstdc++/libgcc, dbus/systemd. These become the package `Depends`. `scripts/check_appimage_libs.sh <appdir> <binary>` is upstream's audit for the same thing.
- Release metadata: `version.inc` at tag v2.5.1 has `set(SoftFever_VERSION "2.5.1")` → authoritative package version. License is AGPL-3.0 (`LICENSE.txt`).
- Desktop entry: `src/dev-utils/platform/unix/com.orcaslicer.OrcaSlicer.desktop` — `Exec=orca-slicer %U`, `Icon=OrcaSlicer`, `StartupWMClass=orca-slicer`, MIME incl. `x-scheme-handler/orcaslicer`. Icon source: `resources/images/OrcaSlicer.png`.
- `nfpm` (goreleaser) v2.49.0 produces both deb and rpm from one YAML; `os.Expand` is applied to `version`, `contents.src/dst`, `depends`, `arch` (confirmed in `nfpm.go`), and `type: tree` copies a directory recursively (confirmed in `files/files.go`). Asset: `nfpm_2.49.0_Linux_x86_64.tar.gz`.

## Files to add

```
.github/workflows/build-deb-rpm.yml
nfpm.yaml
README.md
.gitignore          # dist/, upstream/, *.deb, *.rpm
```

### `nfpm.yaml`

Single config, both formats; `${VERSION}` interpolated by nfpm from the environment.

```yaml
name: orca-slicer
arch: amd64
platform: linux
version: ${VERSION}
release: "1"
section: graphics
priority: optional
maintainer: OrcaSlicer Linux Package <noreply@pixel3design-hub>
vendor: OrcaSlicer
homepage: https://github.com/OrcaSlicer/OrcaSlicer
license: AGPL-3.0-or-later
description: |
  Orca slicer, a 3D printing slicer with rich preset support.
contents:
  - src: build/package            # upstream AppDir, produced by build_linux.sh -s
    dst: /opt/OrcaSlicer
    type: tree
  - src: .pkg-stage/icons          # generated in the workflow, 16..512 hicolor sizes
    dst: /usr/share/icons
    type: tree
  - src: src/dev-utils/platform/unix/com.orcaslicer.OrcaSlicer.desktop
    dst: /usr/share/applications/com.orcaslicer.OrcaSlicer.desktop
  - src: /opt/OrcaSlicer/orca-slicer
    dst: /usr/bin/orca-slicer
    type: symlink
overrides:
  deb:
    depends:
      - libc6 (>= 2.39)
      - libstdc++6
      - libgtk-3-0t64 | libgtk-3-0
      - libwebkit2gtk-4.1-0
      - libgl1
      - libopengl0
      - libglu1-mesa
      - libsecret-1-0
      - libpulse0
      - libudev1
  rpm:
    depends:
      - glibc >= 2.39
      - libstdc++
      - gtk3
      - webkit2gtk4.1
      - mesa-libGL
      - libglvnd-opengl
      - mesa-libGLU
      - libsecret
      - pulseaudio-libs
      - systemd-libs
```

Deliberately short dep lists: GTK3/webkit transitively pull X11, wayland, cairo, pango, glib, gstreamer, soup, fontconfig. Drift is caught by the `check_appimage_libs.sh` step below.

### `.github/workflows/build-deb-rpm.yml`

```yaml
name: build-deb-rpm
on:
  workflow_dispatch:
    inputs:
      upstream_ref: { description: "Upstream tag/ref (empty = latest release)", default: "" }
      force:        { description: "Rebuild even if this release exists", type: boolean, default: false }
permissions: { contents: write }
concurrency: { group: build-linux, cancel-in-progress: false }
```

Job `build`, `runs-on: ubuntu-24.04`, `timeout-minutes: 350`:

1. **resolve** — `ref="${INPUT_REF:-$(gh api repos/OrcaSlicer/OrcaSlicer/releases/latest -q .tag_name)}"`; if `force != true` and `gh release view "$ref"` succeeds → set `skip=true` and exit early.
2. **checkout this repo** (`actions/checkout@v4`) — needed for `nfpm.yaml`.
3. **checkout upstream** — `git clone --depth 1 --branch "$ref" https://github.com/OrcaSlicer/OrcaSlicer.git upstream`.
4. **deps** — `working-directory: upstream`, `sudo ./build_linux.sh -ur`.
5. **build** — `working-directory: upstream`, `./build_linux.sh -dsrlL` (deps + slicer, clang + lld, skip RAM check as upstream CI does).
6. **audit bundle** — `./scripts/check_appimage_libs.sh build/package build/package/bin/orca-slicer`.
7. **stage icons** — in `upstream/`:
   ```bash
   sudo apt-get install -y imagemagick
   for s in 16 24 32 48 64 128 256 512; do
     install -Dm644 resources/images/OrcaSlicer.png ".pkg-stage/icons/hicolor/${s}x${s}/apps/OrcaSlicer.png" 2>/dev/null || true
     convert resources/images/OrcaSlicer.png -resize ${s}x${s} ".pkg-stage/icons/hicolor/${s}x${s}/apps/OrcaSlicer.png"
   done
   ```
   (single `install` then `convert` overwrite; keep only the `convert` line in the real script, `mkdir -p` first).
8. **package** — `VERSION=$(sed -n 's/.*SoftFever_VERSION "\([^"]*\)".*/\1/p' version.inc | head -1)`, fetch nfpm 2.49.0 tarball, then from `upstream/`:
   `nfpm pkg -f ../nfpm.yaml -p deb -t ../dist/` and `-p rpm`.
9. **verify packages** — assert on both artifacts:
   - deb: `dpkg-deb -c` contains `opt/OrcaSlicer/bin/orca-slicer`, `usr/bin/orca-slicer`, `usr/share/applications/com.orcaslicer.OrcaSlicer.desktop`, and `usr/share/icons/hicolor/256x256/apps/OrcaSlicer.png`; `dpkg-deb -f` shows a non-empty `Depends`.
   - rpm: same via `rpm -qlp` / `rpm -qRp`.
   - `ldd` smoke check: extract the deb to `stage-root`, run `LD_LIBRARY_PATH=stage-root/opt/OrcaSlicer/lib:stage-root/opt/OrcaSlicer/bin ldd stage-root/opt/OrcaSlicer/bin/orca-slicer | grep 'not found'` → must be empty except `libwebkit2gtk|libjavascriptcoregtk|libGLU`, which are declared deps not present on the runner.
10. **upload artifact** — `dist/*.deb`, `dist/*.rpm`, retention 7 days.
11. **publish** — `gh release create "$ref" dist/*.deb dist/*.rpm --title "OrcaSlicer $VERSION (deb/rpm)" --notes ...` using `secrets.GITHUB_TOKEN`.

### `README.md`

What the workflow does, how to dispatch it (`upstream_ref`, `force`), asset names, install commands (`sudo apt install ./orca-slicer_*.deb`, `sudo dnf install ./orca-slicer-*.rpm`), and the **glibc 2.39 floor** (Ubuntu 24.04 / Debian 13 / Fedora 40+) that follows from building on ubuntu-24.04 like upstream does.

## Verification plan

1. Push, dispatch the workflow with an empty `upstream_ref`, watch `resolve → build`.
2. Build succeeds → both assets appear on release `v2.5.1`.
3. Download on this Fedora box: `sudo dnf install ./orca-slicer-2.5.1-1.x86_64.rpm` → `orca-slicer` launches, `rpm -q orca-slicer`, desktop entry + menu icon present, `/usr/bin/orca-slicer` resolves.
4. Deb check without a Debian box: `dpkg-deb -I`, `dpkg-deb -c`, and (optionally) `docker run --rm -v $PWD:/p ubuntu:24.04 bash -c 'apt update && apt install -y /p/*.deb && orca-slicer --help'`.
5. `systemctl`/mime sanity: `gio mime model/stl` lists the OrcaSlicer desktop entry.

## Risks

- **Cold-run wall clock**: no ccache and no deps cache in v1 (public repo ⇒ free minutes). Deps ≈ 40–70 min + slicer ≈ 90–150 min on a 4-vCPU runner; margin to the 6 h job cap exists but is not huge. If a cold run approaches ~5 h, add `hendrikmuhs/ccache-action` + an `actions/cache` on `upstream/deps/build/destdir` (keys from upstream's own CI) as the first follow-up.
- **Link-time RAM** on 16 GB / 4 cores: mitigated by `-L` (lld) and the 4-core default parallelism; a failure here means dropping to `CMAKE_BUILD_PARALLEL_LEVEL=3` or `-j 2`.
- **Upstream drift**: `build_linux.sh` flags, the `build/package` layout, or the launcher name changing breaks the workflow or the packaging paths. Both `check_appimage_libs.sh` and the verify step are the tripwires. This repo will need a look whenever upstream touches `src/dev-utils/platform/unix/` or `scripts/linux.d/`.
- **Dep-list rot**: curated depends can go stale if upstream starts distributing another host library instead of bundling it; the `ldd` check is the signal.
- **No runtime GUI test in CI** — the packages are only structurally verified there; step 3 of the verification plan is the real acceptance test.

## Rejected alternatives

- **Repackage the official AppImage** (minutes, no compile): user chose a source build.
- **Local build with `build_linux.sh -g`** on this Fedora box: 16 cores but only ~5 GB RAM free of 14 GB and ~25–30 GB of disk — RAM-tight linking, hours of occupancy, and nothing reusable afterwards.
- **`fpm` or hand-written `dpkg-deb` + `rpmbuild` spec**: two tools / ~40 lines of spec and control boilerplate; one `nfpm` static binary + one YAML covers both, and its `tree` type installs the AppDir verbatim.
- **`cmake --install` staging instead of the AppDir**: upstream only has partial `install()` rules (`src/CMakeLists.txt`); resources, bundled libs, `libexec/orca-slicer-env` and Python runtime are assembled by `build_linux_image.sh` only.

## Not in scope (follow-ups)

- ccache / deps cache, scheduled rebuilds, arm64, package signing, `postinstall` running `update-desktop-database`/`update-mime-database`, separate `-dev` or profile-validator packages.

## Implementation notes (as built)

- nfpm's `release: "1"` also lands in the deb version, so assets are
  `orca-slicer_2.5.1-1_amd64.deb` and `orca-slicer-2.5.1-1.x86_64.rpm` (README/release notes updated).
- `nfpm pkg -t <dir>/` requires that directory to already exist → `mkdir -p ../dist` before packaging.
- dpkg-deb lists directories with a trailing slash; both file lists are normalized
  (`sed -e 's|^\./||' -e 's|/$||'`, rpm also strips the leading `/`) before the path assertions.
- The ldd check also fails when `ldd` yields no output, so a broken/host-incompatible binary
  cannot pass the step silently.
- Verified locally in an ubuntu:24.04 container against a fake AppDir: nfpm config produces both
  packages, symlink and `type: tree` work, `Depends`/rpm `Requires` render as written, the icon
  and desktop paths land where the verify step expects, `appimage_is_host_library` classifies
  webkit/gtk/OpenGL as host and tbb-like libs as bundled, and `actionlint` is clean on the workflow.
- Local container runs need `--security-opt label=disable` on this Fedora host (SELinux), else
  bind-mounted writes fail with EACCES.

## Post-release bug (v2.5.1, run 43614740254): launcher not executable

Symptom on the installed rpm: `Avvio di «OrcaSlicer» non riuscito: Il programma
«orca-slicer» è stato trovato in «/usr/bin/orca-slicer» ma non dispone degli
attributi di esecuzione.` — `/opt/OrcaSlicer/orca-slicer` was `-rwxr-xr--` (0754).

Cause: upstream's `build_linux_image.sh` creates the launcher with `cat >` (0644) and then
`chmod ug+x`, giving 0754. Inside an AppImage that file is owned by whoever runs it, so u+x
suffices; as a package it is root:root, the user is neither owner nor group, and the o+x bit
is what matters.

Fixes applied (all in one rebuild):
- new `Normalize file modes` step before packaging: `o+x` on every file that already has a
  u+x bit, plus explicit 0755 on `build/package/orca-slicer` and `libexec/orca-slicer-env`.
- verify step asserts the deb's mode strings for launcher/env/binary and the rpm's launcher mode
  (`test -x` is not enough — it passes for the owner and for root even at 0754).
- new `Smoke test the deb` step: `apt-get install <deb>` (proves the Depends resolve on 24.04),
  then `test -x` on the installed paths **as the unprivileged runner user**, then
  `orca-slicer --help` (advisory; no display on the runner).
- identity audit against upstream's Flatpak (`scripts/flatpak/com.orcaslicer.OrcaSlicer.yml`):
  package `orca-slicer`, app/AppStream id `com.orcaslicer.OrcaSlicer`, desktop
  `com.orcaslicer.OrcaSlicer.desktop`, icon `OrcaSlicer` — all consistent, and upstream's
  `com.orcaslicer.OrcaSlicer.metainfo.xml` is now shipped to `/usr/share/metainfo/`.
- maintainer email was `noreply@pixel3design-hub` (not a valid domain) → GitHub noreply address.
- icons, desktop entry, MIME registration and `resources/profiles` verified present on the
  installed system; `desktop-file-validate` passes; hicolor declares 512x512.

## Second pass: node20 deprecation + Fedora runtime deps (local validation)

- The run warned `Node.js 20 is deprecated ... actions/checkout@v4, actions/upload-artifact@v4`.
  `checkout@v5+` and `upload-artifact@v6+` declare `runs: using: node24` (checked per tag via the
  API). Workflow now uses `actions/checkout@v7` and `actions/upload-artifact@v7`.
- Local end-to-end validation (node v24.10.0 inside the container, the exact step bodies run
  against a **real** AppDir: upstream's `OrcaSlicer_Linux_AppImage_Ubuntu2404_V2.5.1.AppImage`
  extracted, `AppRun` renamed to `orca-slicer` and chmod 0754 to reproduce the bug) found:
  - normalize + package + all verify assertions pass, `ldd` audit clean on the real bundle;
  - `apt-get install <deb>` succeeds (Depends resolvable on 24.04), non-root `test -x` true,
    `orca-slicer --help` prints `OrcaSlicer-2.5.1: Usage: ...` — the launcher needs no display,
    so it is now a hard gate in CI.
  - **rpm bug found**: on `fedora:latest` the binary died with
    `error while loading shared libraries: libSM.so.6`. `libSM`/`libICE` are in upstream's
    host-library policy (never bundled) but gtk3 does not pull them on Fedora. Added `libSM`,
    `libICE` (rpm) and `libsm6`, `libice6` (deb); both containers now run the packaged binary.
  - Icon source switched from the 154px `resources/images/OrcaSlicer.png` to the 192px
    `OrcaSlicer_192px.png` (largest raster available; the 256px one only exists inside an .ico).
- New hard gate: `Smoke test the rpm on Fedora` runs the rpm in `fedora:latest` via docker and
  requires `orca-slicer --help` to print. `dnf` resolving the dep names is the actual assertion —
  a Ubuntu-only `ldd` audit cannot catch distro-specific package naming.
