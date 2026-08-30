# windows-build-env

Pinned Windows (MSYS2 / UCRT64) build toolchain for **OpenTRIM** and its
subprojects (`libdedx`, `QMatPlotWidget`, `QtDataBrowser`, `QtVectorEdit`).

MSYS2 is a rolling distribution: `pacman -S mingw-w64-ucrt-x86_64-gcc` installs
whatever GCC is current that day. That silently changed the OpenTRIM Windows
build from GCC 16.1 to GCC 16.2 in mid-2026 and broke it (`'uint' was not
declared`). This repo freezes a known-good snapshot so CI builds the same way
every time until the pin is deliberately moved.

## Usage

Replace the `msys2/setup-msys2` step (and the `mingw-w64-ucrt-x86_64-*` install
list) in a Windows workflow with:

```yaml
jobs:
  build:
    runs-on: windows-latest
    defaults:
      run:
        shell: msys2 {0}
    steps:
      - uses: actions/checkout@v4

      - name: Set up pinned toolchain
        uses: ir2-lab/windows-build-env@2026.06

      - name: Configure
        run: cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
      # ...
```

The action runs `msys2/setup-msys2` internally, so later steps can use
`shell: msys2 {0}` as usual. It installs the full pinned set (GCC + Qt +
HDF5 + CMake/Ninja) unconditionally; projects that don't use Qt simply ignore
those packages.

### Inputs

| input | default | purpose |
|---|---|---|
| `extra-packages` | `''` | Extra pacman specs to install at latest version, after the pinned set. |
| `path-type` | `inherit` | Passed to `msys2/setup-msys2`. |

## What is pinned

The **entire dependency closure** of the June-2026 build — 99 packages, the full
`Packages (…)` list from that run's `pacman` step (minus gdb, plus eigen3) — is
pinned and installed as one self-contained `pacman -U` transaction. Nothing is
left to float. (A partial pin was tried first and failed: an unpinned newer
`jsoncpp` broke the pinned CMake; an unpinned newer `gcc-libgfortran` forced the
pinned HDF5 to be skipped.)

See [`toolchain.yml`](toolchain.yml) for the exact list and the CI run it came
from. Highlights:

| | version |
|---|---|
| GCC | 16.1.0-5 |
| binutils | 2.46-3 |
| mingw-w64 crt/headers | 14.0.0.r59.g93753750c-1 |
| Qt5 | 5.15.19+kde+r96 |
| HDF5 | 2.1.1 |
| CMake / Ninja | 4.3.3 / 1.13.2 |
| `msys2/setup-msys2` | v2.31.1 (`e9898307…`) |

Transitive dependencies with stable ABIs (icu, harfbuzz, freetype, openssl, …)
are left floating.

## Versioning

| tag | meaning |
|---|---|
| `2026.06` | Immutable — the June-2026 snapshot. Consumers should pin to this. |
| `v1` | Moving — always points at the newest snapshot. Convenience only. |

Both are placed on the same commit. When the pin is bumped, a new immutable
`YYYY.MM` tag is cut and `v1` is moved forward.

## Bumping the pin

1. Pick a new known-good CI run and extract its package versions (open its job
   log, find the `Packages (NNN) …` line from the `pacman -S` step).
2. Edit **only** `toolchain.yml` — package stems and `setup_msys2_sha`. If the
   `setup-msys2` SHA changes, also update the `uses:` line in `action.yml`.
3. Push a branch; the **Smoke test** workflow must go green (it asserts the
   installed versions match the pin and builds a trivial CMake+Qt project).
4. Merge, then `git tag 2026.NN && git tag -f v1 && git push --tags --force`.
5. Add a changelog entry below.

## Reproducibility caveat

Packages are fetched live from `https://repo.msys2.org/mingw/ucrt64/`, a rolling
mirror. It currently retains these superseded builds, but if MSYS2 ever purges
one the action fails with a 404. Recovery: either bump to the nearest available
older build, or attach the `.pkg.tar.zst` files to a GitHub Release here and
repoint `package_base` in `toolchain.yml` at
`https://github.com/ir2-lab/windows-build-env/releases/download/<tag>`.

## Changelog

| tag | date | notes |
|---|---|---|
| `2026.06` | 2026-08 | Initial pin. Snapshot from OpenTRIM CI run `27217912639` (tag `v1.1.6`, 2026-06-09). Chosen as the last green Windows build before GCC 16.2 / mingw-w64 headers r302 removed the non-standard `uint` typedef leak. |
