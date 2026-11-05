# CodeLLDB for OpenFlame

This branch is upstream CodeLLDB (`vadimcn/codelldb`) at release tag `v1.12.3`
plus the small set of changes OpenFlame workspaces need. The OpenFlame
workspace image (code-server) installs the `linux-x64` package built from this
branch in place of the stock extension.

Branch: `openflame-1.12.3` (from tag `v1.12.3`).
Extension version: `1.12.3-openflame.1`, extension id unchanged
(`vadimcn.vscode-lldb`) so rust-analyzer's Debug code lens and other
integrations keep finding it.

## How OpenFlame attaches

OpenFlame services run inside the developer's workspace pod with
`OPENFLAME_SERVICE=<name> OPENFLAME_LANG=RUST ...` on their command line. The
OpenFlame process picker extension (`openflame-vscode-process-picker`)
contributes the `${command:PickOpenflameProcess}` and
`${command:RunOpenflameProcess}` variables for the `lldb` debug type; it finds
the service's process and returns its pid. A Rust service's launch.json uses:

```json
{ "type": "lldb", "request": "attach", "name": "Attach", "pid": "${command:PickOpenflameProcess}" }
```

## OpenFlame changes

1. **Version and display name** (`CMakeLists.txt`, `package.json`).
   `VERSION_SUFFIX` defaults to `-openflame.1` instead of a `-dev.<timestamp>`
   suffix, so a plain `cmake --build build --target vsix_full` produces
   `1.12.3-openflame.1`; the display name is "CodeLLDB (OpenFlame)". The README
   and image links in the package keep pointing at the upstream `v1.12.3` tag
   (`UPSTREAM_VERSION`), since no `v1.12.3-openflame.1` tag exists upstream.
   Why: the build is distinguishable from the marketplace release in the
   Extensions view and in `extensions.json`. Installed from a VSIX, the
   extension is recorded as pinned, so code-server does not auto-update it to
   the marketplace build.

2. **OpenFlame attach snippets** (`package.json`, `configurationSnippets`).
   "CodeLLDB: Attach to OpenFlame Service" and "CodeLLDB: Run and Attach to
   OpenFlame Service" insert attach configurations whose `pid` is
   `${command:PickOpenflameProcess}` / `${command:RunOpenflameProcess}` and
   whose `sourceLanguages` is `["rust"]` (loads the Rust value formatters).
   Why: lets a developer add a working OpenFlame attach configuration from
   "Add Configuration..." without knowing the picker's variable names.

## Checked against upstream and not carried

- **7-digit PIDs.** The 2021/2022 OpenFlame forks of other debuggers fixed
  fixed-width `ps` parsing that assumed 5-digit PIDs. CodeLLDB's own picker
  (`extension/pickProcess.ts`) parses `platform process list` output by the
  header's column offsets and a leading-digits regex, so it has no such
  limit; with OpenFlame the pid comes from the OpenFlame picker anyway.
- **Command-variable values treated as process names.** Some debuggers
  resolved a `processId` they did not recognise as a process *name* before
  VS Code substituted `${command:...}`. CodeLLDB does not inspect `pid` in
  `resolveDebugConfiguration`, and the adapter accepts `pid` as a number or a
  numeric string (`Either<u64, String>` in `codelldb-types`), so the picker's
  string pid works unchanged.
- **Picker variables.** VS Code merges `debuggers[].variables` contributed by
  several extensions for the same type, so the process picker's own
  `lldb` contribution is enough; this extension does not repeat it.

## Build

The `linux-x64` package is built the way upstream CI builds it
(`.github/workflows/build.yml`): Rust 1.91.0, upstream's clang-15 and sysroot
blobs, LLDB package `codelldb/22.x-74` from `vadimcn/lldb-build`, then

```bash
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_TOOLCHAIN_FILE=$PWD/cmake/toolchain-x86_64-linux-gnu.cmake \
      -DPLATFORM_ID=linux-x64 \
      -DLLDB_PACKAGE=<path>/lldb--x86_64-linux-gnu.zip \
      -G Ninja -B build -S .
cmake --build build --target vsix_full   # -> build/codelldb-full.vsix
```

## Rebasing onto the next upstream release

1. `git fetch upstream --tags`
2. `git checkout -b openflame-<version> v<version>`
3. Cherry-pick the OpenFlame commit(s) from the previous `openflame-*` branch
   (they touch only `CMakeLists.txt`, `package.json` and this file). Conflicts,
   if any, are in the `VERSION`/`VERSION_SUFFIX` lines or next to the
   "CodeLLDB: Attach to Process" snippet.
4. Re-check the "not carried" items above against the new release
   (`extension/pickProcess.ts`, `extension/main.ts` `resolveDebugConfiguration*`,
   `src/codelldb-types/src/lib.rs` `AttachRequestArguments.pid`).
5. Update `lldb_release`/`rust_version` in the build above from upstream's
   `build.yml`, build `vsix_full` for `linux-x64`, and replace
   `extensions/codelldb.vsix` in openflame-workspace.
