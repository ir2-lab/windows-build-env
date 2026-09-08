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

| input            | default   | purpose                                                                |
| ---------------- | --------- | ---------------------------------------------------------------------- |
| `extra-packages` | `''`      | Extra pacman specs to install at latest version, after the pinned set. |
| `path-type`      | `inherit` | Passed to `msys2/setup-msys2`.                                         |

## What is pinned

The **entire dependency closure** of the June-2026 build — 99 packages, the full
`Packages (…)` list from that run's `pacman` step (minus gdb, plus eigen3) — is
pinned and installed as one self-contained `pacman -U` transaction. Nothing is
left to float. (A partial pin was tried first and failed: an unpinned newer
`jsoncpp` broke the pinned CMake; an unpinned newer `gcc-libgfortran` forced the
pinned HDF5 to be skipped.)

See [`toolchain.yml`](toolchain.yml) for the exact list and the CI run it came
from. Highlights:

|                       | version                 |
| --------------------- | ----------------------- |
| GCC                   | 16.1.0-5                |
| binutils              | 2.46-3                  |
| mingw-w64 crt/headers | 14.0.0.r59.g93753750c-1 |
| Eigen3                | 3.4.0 (†)               |
| Qt5                   | 5.15.19+kde+r96         |
| HDF5                  | 2.1.1                   |
| CMake / Ninja         | 4.3.3 / 1.13.2          |
| `msys2/setup-msys2`   | v2.31.1 (`e9898307…`)   |

Transitive dependencies with stable ABIs (icu, harfbuzz, freetype, openssl, …)
are left floating.

(†) 8/9/2026 : OpenTRIM compiles fine with Eigen3 v5.0.1, can be changed in next release

## Versioning

| tag       | meaning                                                           |
| --------- | ----------------------------------------------------------------- |
| `2026.06` | Immutable — the June-2026 snapshot. Consumers should pin to this. |
| `v1`      | Moving — always points at the newest snapshot. Convenience only.  |

Both are placed on the same commit. When the pin is bumped, a new immutable
`YYYY.MM` tag is cut and `v1` is moved forward.

## Bumping the pin

Do this only when you have a reason (new compiler/Qt/library you want, a
security fix in a pinned lib, or a pinned build got purged from repo.msys2.org
and CI now 404s). There is no schedule — the point is not to track upstream.
Expect once or twice a year.

### 1. Get a fresh known-good package list

Run the **Resolve current toolchain closure** workflow from the Actions tab
(`workflow_dispatch`). It installs the needed packages from *current* MSYS2 with
nothing pinned, then writes to the run summary:

- the complete dependency closure, already formatted as a `toolchain.yml`
  `packages:` block — copy it verbatim;
- the latest `msys2/setup-msys2` release tag and its commit SHA.

(The default package set it resolves is
`toolchain cmake ninja hdf5 qt5-base qt5-svg qt5-tools qwt-qt5 angleproject
eigen3` — the union of what all five consumers need. Override via the workflow
input if that changes.)

If you'd rather not run it, the same list is the `Packages (N) …` line from the
`pacman -S` step of any recent green CI run that installed those packages
unpinned.

### 2. Edit `toolchain.yml`

Replace the `packages:` list with the block from step 1. Bump `setup_msys2_sha`
to the reported SHA only if you actually want a newer action — and if you do,
also update the `uses:` line in `action.yml` to the same SHA.

### 3. Test

Push a branch. The **Smoke test** workflow must go green — it now derives the
expected versions from `toolchain.yml` itself (asserts every pinned package is
installed at its pinned version), then compiles a C++ file and a CMake+Qt5
project. For extra confidence, temporarily point one subproject's CI at the
branch (`uses: ir2-lab/windows-build-env@my-branch`) and run it.

### 4. Release

Merge, then cut a new immutable tag and move `v1`:

```bash
git tag 2026.NN
git tag -f v1
git push origin 2026.NN
git push -f origin v1
```

### 5. Roll out

Change `@2026.06` → `@2026.NN` in the five consumer workflows (one line each;
see [`consumers/`](consumers/)). Optionally refresh each project's release
assets afterward by force-pushing its release tag, so published binaries match
the new toolchain.

### 6. Changelog

Add an entry below.

## Reproducibility caveat

Packages are fetched live from `https://repo.msys2.org/mingw/ucrt64/`, a rolling
mirror. It currently retains these superseded builds, but if MSYS2 ever purges
one the action fails with a 404. Recovery: either bump to the nearest available
older build, or attach the `.pkg.tar.zst` files to a GitHub Release here and
repoint `package_base` in `toolchain.yml` at
`https://github.com/ir2-lab/windows-build-env/releases/download/<tag>`.

## Changelog

| tag       | date    | notes                                                                                                                                                                                                              |
| --------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `2026.06` | 2026-08 | Initial pin. Snapshot from OpenTRIM CI run `27217912639` (tag `v1.1.6`, 2026-06-09). Chosen as the last green Windows build before GCC 16.2 / mingw-w64 headers r302 removed the non-standard `uint` typedef leak. |
