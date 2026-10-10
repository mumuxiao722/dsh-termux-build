# dsh-termux

Source patches and instructions for building and running [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`) on Termux (Android).

[中文](README_CN.md)

## What this is

dsh has no official Android support, and the published npm artifacts do not run on bionic. This repository provides:

1. **`dsh-termux-android.patch`** — the runtime source patch: 6 JavaScript files across 13 hunks. **Everyone needs this one.**
2. **`koffi-android.patch`** — a compile-time C++ patch for koffi (3 hunks). **Only needed if you compile koffi yourself.**
3. **Prebuilt `.node` binaries** in the [Release](https://github.com/mumuxiao722/dsh-termux-build/releases), so you can skip compiling entirely.
4. **This document** — the full manual process: dependency checks, native modules, patch application, verification.

There is no build script. Installing dependencies, compiling, and how you compile are all up to you.

### Which patch do I need?

| Situation | `dsh-termux-android.patch` (JS) | `koffi-android.patch` (C++) |
|---|---|---|
| Using prebuilt koffi from the Release | **required** | not needed — already baked into `koffi.node` |
| Compiling koffi from source | **required** | **required** |

The split exists because the C++ patch only matters at compile time. It is already baked into the prebuilt `koffi.node`, so if you take the prebuilt you never need it — and applying it would be a no-op since you are not rebuilding. Both files are applied with plain `patch -p1`; see [4. Apply the source patch](#4-apply-the-source-patch).

## Scope

The patch is **minimal by design**: it only touches points that actually fail on Android, and changes no dsh business logic. Every JS change is guarded by a runtime `process.platform === "android"` / `!== "android"` check, so it is dead code on other platforms.

Applies to: **dsh 0.2.0-rc.2** (the `version` field in `package.json`). Other versions are unverified.

## Prerequisites

Check the environment first; install what is missing, how you like.

### Required

| Item | Check | If missing |
|---|---|---|
| Node.js `^22.19 \|\| >=24` | `node -v` | `pkg install nodejs-lts` |
| npm | `npm -v` | `pkg install npm` |
| Android API level | `getprop ro.build.version.sdk` | Verified on API 24; lower levels need your own check |
| `patch` | `command -v patch` | `pkg install patch` |
| Termux `ripgrep` | `command -v rg` | `pkg install ripgrep` |
| git | `command -v git` | `pkg install git` |

### Only needed if you compile the native modules

Skip this if you take the prebuilt binaries from the [Release](https://github.com/mumuxiao722/dsh-termux-build/releases).

| Item | Check | If missing |
|---|---|---|
| Toolchain | `command -v clang cmake make python pkg-config` | `pkg install clang cmake make python pkg-config` |
| zlib / openssl headers | `ls $PREFIX/include/zlib.h $PREFIX/include/openssl/ssl.h` | `pkg install zlib openssl` |

## Process

> **Short path:** prebuilt binaries for the four native modules are attached to the
> [GitHub Release](https://github.com/mumuxiao722/dsh-termux-build/releases). If you are on
> ARM64 / API 24, you can skip straight to [4](#4-apply-the-source-patch) after installing dsh
> and its runtime dependencies. Compiling from source is the fallback when the prebuilts do
> not fit your environment, or when you want a reproducible build.

### 1. Install dsh

Skip all install scripts — native modules are built separately:

```bash
npm i -g @deepseek-ai/dsh@0.2.0-rc.2 --ignore-scripts --no-audit --no-fund
```

Note the install path, you will need it:

```bash
DSH_ROOT="$(npm root -g)/@deepseek-ai/dsh"
NM="$DSH_ROOT/node_modules"
```

### 2. Add Android runtime dependencies

```bash
npm i -g @img/sharp-wasm32 esbuild @esbuild/android-arm64 --ignore-scripts --no-audit --no-fund
```

`sharp` has no native Android package and falls back to WASM; `esbuild` needs the android-arm64 platform package specified explicitly.

Install `ripgrep` via `pkg install ripgrep` (see the table above).

### 3. Provide the native modules (required)

All four must be present, one way or another — dsh will not start without them. Either
download the prebuilts from the [Release](https://github.com/mumuxiao722/dsh-termux-build/releases)
or compile them yourself; the Release notes give the exact install paths for each.

| Module | Why it is needed |
|---|---|
| `koffi` | dsh goes through FFI; Node cannot load the glibc prebuilt |
| `node-pty` | Bionic has no Android prebuilds |
| `node-addon-require-builtin` | Upstream ships no Android artifact at all |
| `node-addon-system` `flock` | Single-writer session lock; upstream ships glibc only |

#### Compiling from source

**koffi** — apply `koffi-android.patch` first, since it patches `base.cc`, then compile:

```bash
cd "$DSH_ROOT"
patch -p1 -N --forward < /path/to/koffi-android.patch
cd "$NM/koffi"
node ./cnoke.cjs -P . -D src/koffi --release
```

**node-pty**:

```bash
cd "$NM/node-pty"
node-gyp rebuild --nodedir="$PREFIX"
```

**node-addon-require-builtin** — the build entry point is `native-src/` inside the package. Upstream ships no Android artifact, so build it from source:

```bash
cd "$NM/node-addon-require-builtin/native-src"
npm i
NARB_PRODUCT=1 npx node-gyp rebuild --nodedir="$PREFIX"
```

`narb_product` defaults to `internal-builtin`; **you must pass `NARB_PRODUCT=1`** to get a loadable binding.

Place the artifact where the loader expects it. It looks up `build/<backend>/<abi>-<platform>/require_builtin.node`, where `<abi>` is fixed at `napi-v9` (Node-API 9):

```bash
mkdir -p "$NM/node-addon-require-builtin/build/napi/napi-v9-android-arm64"
cp build/Release/require_builtin.node "$NM/node-addon-require-builtin/build/napi/napi-v9-android-arm64/"
```

**flock** — upstream ships `node-addon-system-{linux,darwin}-*` with no android package. The loader picks `bin/glibc` or `bin/musl` based on `report.header.glibcVersionRuntime`; on bionic that field is empty, so the artifact goes in `bin/musl`:

```bash
cd "$NM/@deepseek-ai/node-addon-system/build-src"
node-gyp rebuild --nodedir="$PREFIX"
mkdir -p "$NM/node-addon-system-android-arm64/bin/musl"
cp build/Release/system.node "$NM/node-addon-system-android-arm64/bin/musl/"
printf '{"name":"@deepseek-ai/node-addon-system-android-arm64","version":"0.1.2-termux","main":"index.js"}\n' \
  > "$NM/node-addon-system-android-arm64/package.json"
echo 'module.exports = {};' > "$NM/node-addon-system-android-arm64/index.js"
```

`build-src/flock.c` is the implementation this addon needs. The adjacent `src/main.c` is a standalone Landlock launcher CLI, unrelated to flock; bionic has no Landlock either and `binding.gyp` does not build it.

### 4. Apply the source patch

```bash
cd "$DSH_ROOT"

# Always required
patch -p1 -N --forward < /path/to/dsh-termux-android.patch

# Only if you compiled koffi yourself (skip if you used the prebuilt koffi.node)
patch -p1 -N --forward < /path/to/koffi-android.patch
```

Re-running is safe: `-N` skips already-applied hunks. `patch` leaves `.rej` files for hunks it skipped, which is expected; remove them with:

```bash
find node_modules -name '*.rej' -delete
```

### 5. Verify

```bash
node -e "require('koffi'); console.log('koffi', require('koffi').version)"
node -e "require('node-pty'); console.log('node-pty OK')"
node -e "require('sharp'); console.log('sharp OK')"
```

No errors from all three means the native modules are good. Start dsh:

```bash
dsh web
```

The Web UI prints a URL with a token.

### Verify the patch took effect per file

When a patch silently fails to apply, symptoms are subtle — tools degrade quietly rather than erroring. Spot-check each one:

```bash
cd "$(npm root -g)/@deepseek-ai/dsh"

# session single-writer lock
grep -c "platform !== 'android'" node_modules/@deepseek-ai/node-addon-system/lib/flock.js

# write tool atomic publication
grep -c COPYFILE_EXCL node_modules/@deepseek-ai/dsh-fs-local/lib/index.js

# image read/write
grep -c publishNoReplace node_modules/@deepseek-ai/dsh-attachment-local/lib/index.js

# grep / glob
grep -c 'join(executable.dir, "rg")' node_modules/@deepseek-ai/dsh-tool-fs-search/lib/index.js

# terminal
grep -rc 'platform === "android"' node_modules/@deepseek-ai/dsh-subprocess-local/lib/runner-launch-*.js

```

Each should print `1` or more. A `0` means that file was not patched — re-run `patch -p1 -N --forward`.

## What this fixes

Without this patch, dsh on Termux is not merely "a bit rough" — an entire class of features either fails outright or silently degrades. The table below lists symptoms you would actually hit.

| Symptom | Root cause | Fixed in |
|---|---|---|
| **Sessions cannot be saved at all**, every turn fails with `ERR_FLOCK_UNSUPPORTED_PLATFORM` | dsh uses a file lock to guarantee a single writer; flock's platform allowlist admits only linux/darwin and Termux reports `android` | `node-addon-system/lib/flock.js` + a self-built `system.node` |
| **The `write` tool fails with `EACCES`** | Android's SELinux forbids hard link creation in the app domain, and dsh publishes atomic writes via `link()` | `dsh-fs-local/lib/index.js` |
| **`read_image` / `save_image` fail with `EACCES`** | Same cause, at the two `link()` call sites in attachment publishing | `dsh-attachment-local/lib/index.js` |
| **`read_image` / `save_image` fail with `open '/data/data': EACCES`** | `ensureDurableHome` deliberately fsyncs every ancestor of `$HOME` up to the filesystem root for durability. On Termux `$HOME` is `/data/data/com.termux/files/home`, so the chain passes through the app data root, which the sandbox forbids opening | `dsh-attachment-local/lib/index.js` |
| **The `grep` / `glob` tools always fail with `SEARCH_FAILED`** | dsh's search does not consult `PATH`; it imports `@vscode/ripgrep` and uses that package's `rgPath`, but it publishes no android-arm64 variant (registry has linux/darwin/win32 only) | `dsh-tool-fs-search/lib/index.js` + `pkg install ripgrep` |
| **The terminal in the Web UI will not open**, startup logs `terminal inspection is unsupported on platform android` | `createProcessInspector` is a hardcoded three-branch allowlist with no `android` branch, so it throws at load time; Android *is* Linux and that inspector only reads `/proc` and signals processes it spawned itself, both of which work on Termux, so folding it into the linux branch is enough | `dsh-subprocess-local/lib/runner-launch-*.js` |
| FFI-related features fail to compile or load | `statx` is hidden behind `__INTRODUCED_IN(30)` on Bionic, so C++ treats it as a constructor; `posix_spawn` file actions need API 28 | `koffi/lib/native/base/base.cc` (compile-time) |

Keyword list, so you can tell whether a problem you hit falls in this scope:

```
ERR_FLOCK_UNSUPPORTED_PLATFORM
SEARCH_FAILED / ripgrep launch failed
Could not find @vscode/ripgrep-android-arm64
terminal inspection is unsupported on platform android
EACCES (link / hard link)
EACCES: open '/data/data'
```

### Why grep / glob use Termux's rg instead of an npm package

Worth calling out, because this one cannot be solved by "just install something".

dsh's search tools do **not** consult `PATH`. They import `@vscode/ripgrep` directly and use the `rgPath` recorded inside that package. `@vscode/ripgrep` publishes platform packages for linux / darwin / win32 only — there is no android-arm64. So on Termux this is not "hard to install"; Node throws at import time:

```
Could not find @vscode/ripgrep-android-arm64
```

and every search becomes `SEARCH_FAILED`. The patch bypasses the package on Android and points at the `rg` that Termux ships (`executable.dir` is Termux's bin directory).

Termux's `rg` is a bionic build, runs fine, and is syntax-compatible. **Do not look for an Android ripgrep on npm** — it does not exist.

## dsh dependency chain

Understanding this helps you reason about where the patch lands and what changes on upgrade. dsh is a cordis plugin system: the top-level package flat-depends on ~130 `@deepseek-ai/dsh-*` packages, which all end up under `node_modules/@deepseek-ai/dsh/node_modules/` at runtime.

Where the 8 patched files sit in that chain:

```
@deepseek-ai/dsh  (top level, ~130 flat deps)
├── @deepseek-ai/dsh-app-boot ──────────► node-addon-require-builtin ─┐
│                                            └── node-addon-native-custom-loader
│                                                 └── build/napi/napi-v9-android-arm64/  ← self-built
├── @deepseek-ai/cordis-plugin-loader ───► node-addon-require-builtin
│
├── @deepseek-ai/dsh-web-app
│   └── @deepseek-ai/dsh-tool-fs-search  ◄── patch: point at Termux's rg
│
└── @deepseek-ai/dsh-base
    ├── @deepseek-ai/dsh-session-persistence-jsonl  ◄── patch: rename instead of link
    │   └── @deepseek-ai/node-addon-system  ◄── patch: allow android
    │       └── @deepseek-ai/node-addon-system-android-arm64/bin/musl/system.node  ← self-built
    ├── @deepseek-ai/dsh-fs-local  ◄── patch: link → copyFile(EXCL)
    │   └── @deepseek-ai/dsh-fs-sandbox
    ├── @deepseek-ai/dsh-attachment-local  ◄── patch: link → copyFile(EXCL) + durability walk
    ├── @deepseek-ai/dsh-subprocess-local  ◄── patch: android → LinuxProcessInspector
    └── @deepseek-ai/dsh-sandbox-local ──► node-addon-system
```

Three things worth knowing:

- **`dsh-base` is the hub** — session persistence, filesystem, attachments, and subprocess patches all hang off it. When it breaks, large areas fail at once rather than one tool at a time.
- **The two native modules use different loaders with different rules.** `node-addon-system` chooses between `bin/glibc` and `bin/musl` based on `glibcVersionRuntime`; `node-addon-require-builtin` looks under `build/<backend>/<abi>-<platform>/`. Putting the artifact in the wrong place shows up as "the file is right there but it won't load".
- **Termux never takes the glibc branch** — `glibcVersionRuntime` is empty on bionic, so the flock artifact must go in `bin/musl`.

## What the patch changes

### `koffi-android.patch` (compile-time, 3 hunks, 1 file)

Only needed when compiling koffi yourself; the prebuilt `koffi.node` already has these.

- **`statx` fallback** — Bionic hides the `statx` declaration behind `__INTRODUCED_IN(30)`, so below API 30 C++ treats `statx(...)` as a `struct statx` construction. Excluding `__ANDROID__` routes it to the generic `fstatat`/`fstat` branch.
- **Disable `posix_spawn` file actions below API 28** — that helper needs API 28. dsh's FFI path never calls it, so return `ENOSYS`.
- **Move `return true` outside the `#endif`** — so the success path returns on every platform.

### `dsh-termux-android.patch` (runtime, 13 hunks, 6 files)

### `@deepseek-ai/dsh-subprocess-local` (2 hunks)

`createProcessInspector` only accepts `linux` / `darwin` / `win32` and throws on anything else:

```
subprocess-local: terminal inspection is unsupported on platform android
```

Termux reports `process.platform === "android"`, so the plugin fails to load and the local terminal feature is entirely unavailable.

Android *is* Linux. That inspector only reads `/proc/<pid>/stat`, lists `/proc`, and calls `ps`/`kill`, and it only inspects child processes it spawned itself — it never crosses an app boundary and so never touches SELinux. Folding `android` into the linux branch is sufficient.

> This file has a content hash in its name (`runner-launch-<hash>.js`), which changes on dsh upgrades.

### `@deepseek-ai/node-addon-system` `lib/flock.js` (1 hunk)

The platform allowlist admits only linux/darwin, so Termux throws `ERR_FLOCK_UNSUPPORTED_PLATFORM` and **entire session writes fail**. `flock()` itself works on bionic; only the check needs relaxing.

### `@deepseek-ai/dsh-session-persistence-jsonl` `lib/index.js` (2 hunks)

The Android sandbox rejects `link(2)` with EACCES, so publishing the initial JSONL session fails. Use `rename()` on Android instead.

### `@deepseek-ai/dsh-fs-local` `lib/index.js` (2 hunks)

Android's SELinux forbids hard link creation in the app domain, so `link()` always fails with EACCES. The `createIfAbsent` branch of `writeFileAtomic` falls back to `copyFile(COPYFILE_EXCL)`, which creates the target with `O_CREAT|O_EXCL` — semantically equivalent.

### `@deepseek-ai/dsh-attachment-local` `lib/index.js` (5 hunks)

Two `link()`-based publications (`publishImmutableAlias`, `publishStagedObject`) get the same `copyFile` fallback. Also fixes `ensureDurableHome`'s durability walk: it deliberately fsyncs every ancestor of `$HOME` up to the filesystem root, and on Termux `$HOME=/data/data/com.termux/files/home`, so the chain passes through `/data/data` — which the Android sandbox forbids apps from opening, making every `saveImage`/`read_image` fail. A level that cannot be opened is outside this app's sync scope, so stop the walk instead of failing the whole save.

### `@deepseek-ai/dsh-tool-fs-search` `lib/index.js` (1 hunk)

dsh's grep/glob does not consult PATH; it uses `@vscode/ripgrep`'s `rgPath`, and that package has no `@vscode/ripgrep-android-arm64` (the registry only has linux/darwin/win32). The import throws `Could not find`, turning every search into `SEARCH_FAILED`. Point it at Termux's `rg` on Android.

## Known limitations

### Plugins: pnpm and hard links on Android (out of scope for this patch)

dsh manages profile plugins with pnpm. **The Android sandbox forbids hard link creation**, and pnpm relies on hard links by default to reuse packages from its store — so plugin installs either fail or leave a broken `node_modules`.

This is neither a dsh bug nor something this patch fixes: the patch only covers dsh's own packages, and **plugins are out of scope**. You need two settings in the profile's `pnpm-workspace.yaml`:

```yaml
nodeLinker: hoisted
packageImportMethod: copy
```

- `packageImportMethod: copy` — use copying instead of hard links. **This one is required.**
- `nodeLinker: hoisted` — flatten the tree into the flat `node_modules` layout dsh expects

Two further things worth knowing:

- Pass `--ignore-scripts` when installing plugins that carry `install`/`prepare` scripts. Some depend on Linux-native binaries (e.g. `cloudflared`) that cannot be built on Android, and pnpm 12 treats "ignored build scripts" as a hard error that aborts the whole install.
- Installing from GitHub fails more often than from npm, since it needs `git ls-remote`. Prefer a version from the npm registry when one exists, to avoid git entirely.

> **Note:** GitHub is frequently unreachable from Termux networks — `git ls-remote` and raw downloads time out or fail with `SSL_ERROR_SYSCALL`. The npm registry is far more reliable from Termux. So when a package exists on npm, install it by version (`npm i <pkg>@<version>`) rather than from a GitHub spec. The same applies to the Release tarball: if `github.com` is slow for you, the binaries inside are reproducible from npm sources plus the patch.

### Other

- **No sandbox backend** — Termux has neither Landlock nor bubblewrap, so container isolation is unavailable. Set command execution to Manual Approval or Full Access in the Web UI.
- **`link()` degrades** — hard links are unavailable on Android, so all atomic publications go through `copyFile`. Functionally equivalent, but you lose the shared-inode property.
- **flock binary** — must be built by hand, or downloaded from the Release.
- **Version-bound** — the patch targets dsh 0.2.0-rc.2. dsh is a developer preview, so upgrades will likely need a regenerated patch: take a clean npm copy of the new version and re-run `diff -u`.
- **`subprocess-local` filename hash** — the hash changes on dsh upgrades, so the path in the patch file must be updated to match.

## Uninstall

```bash
npm rm -g @deepseek-ai/dsh
```

The patch only modifies files inside the install directory and writes nowhere else; the `node_modules/*-android-arm64/` directories you created are removed as part of the uninstall.

## License

MIT — see [LICENSE](LICENSE). The patches modify third-party works, whose copyright notices are retained in the patched files: DeepSeek Harness (© 2026 DeepSeek) and koffi (© 2026 Niels Martignène).
