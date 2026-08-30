# Consumer wiring (Phase 3)

Drop-in replacements for each subproject's Windows workflow. Each one swaps the
inline `msys2/setup-msys2` + `mingw-w64-ucrt-x86_64-*` install block for a single
`uses: ir2-lab/windows-build-env@2026.06` step. Nothing else in the workflows
changes.

Apply order (build one, confirm green, move on):

1. `libdedx` — no Qt, smallest blast radius. → `libdedx-windows.yml`
2. `QMatPlotWidget` (`gapost/qmatplotwidget`) — → `qmatplotwidget-windows.yml`
3. `QtDataBrowser` — consumes QMatPlotWidget packages. → `qtdatabrowser-windows-build.yml`
4. `QtVectorEdit` — → `qtvectoredit-windows-build.yml`

For each: copy the file to the subproject's `.github/workflows/<original-name>`,
push to a branch, confirm the currently-released source still builds against the
pin, then merge to `main`. Do **not** bump the subproject version or cut a
release — this is a CI-process change only.

The `2026.06` tag must exist on `ir2-lab/windows-build-env` first (Phase 2).
