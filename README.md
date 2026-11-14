# OrcaSlicer Linux packages (deb + rpm)

Community-built `.deb` and `.rpm` packages for [OrcaSlicer](https://github.com/OrcaSlicer/OrcaSlicer).
Upstream ships an AppImage and a Flatpak; this repo compiles the slicer from source on
GitHub Actions and wraps the resulting tree into native distro packages.

## How it works

`.github/workflows/build-deb-rpm.yml`:

1. resolves an upstream tag (default: the latest OrcaSlicer release),
2. `git clone`s it and installs the build deps with upstream's own `./build_linux.sh -ur`,
3. builds with `./build_linux.sh -dsrlL` — the same steps upstream CI runs, minus AppImage and tests,
4. audits and packages `build/package/` (the AppDir upstream generates) with
   [nfpm](https://nfpm.goreleaser.com) into:
   - `orca-slicer_<version>-1_amd64.deb`
   - `orca-slicer-<version>-1.x86_64.rpm`
5. verifies package contents and the library bundle, uploads artifacts, and publishes a
   release tagged like upstream (`v2.5.1`).

Installed layout: `/opt/OrcaSlicer` (bundled runtime), `/usr/bin/orca-slicer` symlink,
`/usr/share/applications/com.orcaslicer.OrcaSlicer.desktop`, hicolor icons 16–512.

## Running it

Actions → **build-deb-rpm** → *Run workflow*:

| input | meaning |
| --- | --- |
| `upstream_ref` | tag/ref to build, empty = latest upstream release |
| `force` | rebuild even when the release for that tag already exists |

A build takes a few hours on a hosted runner (full C++ compile, no ccache yet) and needs
roughly 25–30 GB of scratch space.

## Installing

```bash
sudo apt install ./orca-slicer_<version>-1_amd64.deb
sudo dnf install ./orca-slicer-<version>-1.x86_64.rpm
```

Because the binaries are compiled on Ubuntu 24.04 like upstream's AppImage, they need
**glibc >= 2.39**: Ubuntu 24.04+, Debian 13+, Fedora 40+ and equivalents. On older
distros use upstream's AppImage.

## Notes

- These packages are unofficial. Bug reports about the slicer itself belong upstream.
- `nfpm.yaml` lists the host libraries upstream deliberately does not bundle; the workflow's
  `ldd` check fails the build if the bundle ever starts depending on an undeclared one.
