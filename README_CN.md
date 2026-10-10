# dsh-termux

在 Termux (Android) 上构建并运行 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`) 的补丁集合与流程说明。

[English](README.md)

## 这是什么

dsh 官方没有 Android 支持，npm 预编译产物在 bionic 上跑不起来。本仓库提供：

1. **`dsh-termux-android.patch`** —— 运行期源码补丁，6 个 JavaScript 文件、13 处改动。**所有人都需要这个。**
2. **`koffi-android.patch`** —— koffi 的编译期 C++ 补丁，3 处改动。**只有自己编译 koffi 时才需要。**
3. **[Release](https://github.com/mumuxiao722/dsh-termux-build/releases) 里的预编译 `.node`** —— 可以完全跳过编译。
4. **本文档** —— 完整的手工流程：依赖检测、原生模块、补丁应用、验证。

没有构建脚本。是否安装依赖、是否编译、用什么编译，全部由你决定。

### 我到底需要哪个补丁？

| 你的情况 | `dsh-termux-android.patch`（JS） | `koffi-android.patch`（C++） |
|---|---|---|
| 用 Release 里的预编译 koffi | **必须** | 不需要——已经烘焙进 `koffi.node` 了 |
| 自己编译 koffi | **必须** | **必须** |

之所以拆开，是因为 C++ 补丁只在编译期有意义。它已经被烘焙进预编译的 `koffi.node`，所以用预编译的人根本不需要它——而真去应用它也是空操作，因为你没有重新编译。两个文件都是用标准 `patch -p1` 应用，见[第 4 步](#4-应用源码补丁)。

## 影响范围

补丁是**最小必要修改**，只在 Android 上会失败的点上生效，不改动任何 dsh 业务逻辑。所有 JS 改动都用 `process.platform === "android"` 或 `!== "android"` 做了运行时判断，在其他平台上是死代码。

适用范围：**dsh 0.2.0-rc.2**（`package.json` 的 `version` 字段）。其他版本未验证。

## 前置检测

先确认环境，不满足的按提示自行安装。

### 必需

| 项 | 检测 | 不满足时 |
|---|---|---|
| Node.js `^22.19 \|\| >=24` | `node -v` | `pkg install nodejs-lts` |
| npm | `npm -v` | `pkg install npm` |
| Android API 级别 | `getprop ro.build.version.sdk` | 补丁针对 API 24 验证；更低需自行确认 |
| `patch` | `command -v patch` | `pkg install patch` |
| Termux `ripgrep` | `command -v rg` | `pkg install ripgrep` |
| git | `command -v git` | `pkg install git` |

### 原生模块编译才需要

只有打算自己编译原生模块时才装。dsh 的 npm 包默认不带 Android 产物，**必须编译**，否则启动失败。

| 项 | 检测 | 不满足时 |
|---|---|---|
| 编译工具链 | `command -v clang cmake make python pkg-config` | `pkg install clang cmake make python pkg-config` |
| zlib / openssl 头文件 | `ls $PREFIX/include/zlib.h $PREFIX/include/openssl/ssl.h` | `pkg install zlib openssl` |

## 流程

> **捷径：** 四个原生模块的预编译产物在 [GitHub Release](https://github.com/mumuxiao722/dsh-termux-build/releases) 里。
> 如果你是 ARM64 / API 24，装完 dsh 和运行时依赖后可以直接跳到[第 4 步](#4-应用源码补丁)。
> 从源码编译是备选路径——预编译产物不适用你的环境时，或者你想要可复现的构建时再用。

### 1. 安装 dsh

跳过所有 install 脚本——原生模块稍后自建：

```bash
npm i -g @deepseek-ai/dsh@0.2.0-rc.2 --ignore-scripts --no-audit --no-fund
```

记下安装路径，后面要用：

```bash
DSH_ROOT="$(npm root -g)/@deepseek-ai/dsh"
NM="$DSH_ROOT/node_modules"
```

### 2. 补齐 Android 运行时依赖

```bash
npm i -g @img/sharp-wasm32 esbuild @esbuild/android-arm64 --ignore-scripts --no-audit --no-fund
```

`sharp` 在 Android 没有原生包，走 WASM 回退；`esbuild` 需要显式指定 android-arm64 平台包。

`ripgrep` 用 `pkg install ripgrep` 装 Termux 版本，见上表。

### 3. 准备原生模块（必需）

四个模块必须到位，用哪种方式都行——缺了 dsh 起不来。可以从
[Release](https://github.com/mumuxiao722/dsh-termux-build/releases) 下载预编译产物，
也可以自己编译；每个产物的确切安装路径见 Release notes。

| 模块 | 为什么需要 |
|---|---|
| `koffi` | dsh 走 FFI，Node 加载不了 glibc 预编译产物 |
| `node-pty` | Bionic 无 Android prebuilds |
| `node-addon-require-builtin` | 上游完全没有 Android 产物 |
| `node-addon-system` 的 `flock` | 会话单写者锁，上游只发 glibc 包 |

#### 从源码编译

**koffi** —— 先应用 `koffi-android.patch`（它改的是 `base.cc`），再编译：

```bash
cd "$DSH_ROOT"
patch -p1 -N --forward < /path/to/koffi-android.patch
cd "$NM/koffi"
node ./cnoke.cjs -P . -D src/koffi --release
```

**node-pty**：

```bash
cd "$NM/node-pty"
node-gyp rebuild --nodedir="$PREFIX"
```

**node-addon-require-builtin** —— 构建入口在包内的 `native-src/`。上游没有 Android 产物，需要从源码编：

```bash
cd "$NM/node-addon-require-builtin/native-src"
npm i
NARB_PRODUCT=1 npx node-gyp rebuild --nodedir="$PREFIX"
```

`narb_product` 默认是 `internal-builtin`，**必须显式传 `NARB_PRODUCT=1`** 才会产出可加载的绑定。

产物要放到加载器期望的位置。加载器按 `build/<backend>/<abi>-<platform>/require_builtin.node` 查找，其中 `<abi>` 固定为 `napi-v9`（对应 Node-API 9）：

```bash
mkdir -p "$NM/node-addon-require-builtin/build/napi/napi-v9-android-arm64"
cp build/Release/require_builtin.node "$NM/node-addon-require-builtin/build/napi/napi-v9-android-arm64/"
```

**flock** —— 上游 `node-addon-system-{linux,darwin}-*` 不含 android 包。加载器会按 `report.header.glibcVersionRuntime` 选 `bin/glibc` 还是 `bin/musl`，bionic 上该字段为空，所以产物放 `bin/musl`：

```bash
cd "$NM/@deepseek-ai/node-addon-system/build-src"
node-gyp rebuild --nodedir="$PREFIX"
mkdir -p "$NM/node-addon-system-android-arm64/bin/musl"
cp build/Release/system.node "$NM/node-addon-system-android-arm64/bin/musl/"
printf '{"name":"@deepseek-ai/node-addon-system-android-arm64","version":"0.1.2-termux","main":"index.js"}\n' \
  > "$NM/node-addon-system-android-arm64/package.json"
echo 'module.exports = {};' > "$NM/node-addon-system-android-arm64/index.js"
```

`build-src/flock.c` 是这个 addon 需要的实现。相邻的 `src/main.c` 是独立的 Landlock launcher CLI，与 flock 无关；bionic 也没有 Landlock，`binding.gyp` 不会编它。

### 4. 应用源码补丁

```bash
cd "$DSH_ROOT"

# 始终需要
patch -p1 -N --forward < /path/to/dsh-termux-android.patch

# 仅在自己编译 koffi 时需要（用了预编译 koffi.node 就跳过）
patch -p1 -N --forward < /path/to/koffi-android.patch
```

重复执行是安全的：`-N` 会跳过已应用的 hunk。`patch` 对已应用的 hunk 会留 `.rej`，属预期，可删除：

```bash
find node_modules -name '*.rej' -delete
```

### 5. 验证

```bash
node -e "require('koffi'); console.log('koffi', require('koffi').version)"
node -e "require('node-pty'); console.log('node-pty OK')"
node -e "require('sharp'); console.log('sharp OK')"
```

三条都无报错即原生模块就绪。启动：

```bash
dsh web
```

Web UI 会打印带 token 的地址。

### 逐项确认补丁生效

补丁不生效时症状很隐蔽（工具静默失败而非报错），可以逐个抽查：

```bash
cd "$(npm root -g)/@deepseek-ai/dsh"

# 会话单写者锁
grep -c "platform !== 'android'" node_modules/@deepseek-ai/node-addon-system/lib/flock.js

# write 工具的原子发布
grep -c COPYFILE_EXCL node_modules/@deepseek-ai/dsh-fs-local/lib/index.js

# 图片读写
grep -c publishNoReplace node_modules/@deepseek-ai/dsh-attachment-local/lib/index.js

# grep / glob
grep -c 'join(executable.dir, "rg")' node_modules/@deepseek-ai/dsh-tool-fs-search/lib/index.js

# 终端
grep -rc 'platform === "android"' node_modules/@deepseek-ai/dsh-subprocess-local/lib/runner-launch-*.js

```

每条应输出 `1` 或更多。若为 `0`，说明该文件没打上补丁，重跑 `patch -p1 -N --forward`。

## 修复了什么

不打这个补丁，dsh 在 Termux 上不是「不能用」，而是**一整类功能静默失效或直接报错**。下表是你会实际看到的症状。

| 你看到的现象 | 根因 | 修在哪 |
|---|---|---|
| **会话根本存不下来**，每轮都失败，报 `ERR_FLOCK_UNSUPPORTED_PLATFORM` | dsh 用文件锁保证单写者，而 flock 的平台白名单只有 linux/darwin，Termux 报 `android` | `node-addon-system/lib/flock.js` + 自建 `system.node` |
| **write 工具报 `EACCES`** | Android 的 SELinux 禁止应用域创建硬链接，dsh 的原子写用 `link()` 发布 | `dsh-fs-local/lib/index.js` |
| **read_image / save_image 报 `EACCES`** | 同上，附件发布的两个 `link()` 调用点 | `dsh-attachment-local/lib/index.js` |
| **read_image / save_image 报 `open '/data/data': EACCES`** | `ensureDurableHome` 为保证持久化，会把 `$HOME` 的祖先一路 fsync 到文件系统根；Termux 的 `$HOME` 在 `/data/data/com.termux/files/home`，链条穿过应用数据根，而沙箱禁止打开它 | `dsh-attachment-local/lib/index.js` |
| **grep / glob 工具全部报 `SEARCH_FAILED`** | dsh 的搜索不查 PATH，走 `@vscode/ripgrep` 的 `rgPath`，而该包没有 android-arm64 变体（registry 只有 linux/darwin/win32） | `dsh-tool-fs-search/lib/index.js` + `pkg install ripgrep` |
| **Web UI 里的终端打不开**，启动日志报 `terminal inspection is unsupported on platform android` | `createProcessInspector` 是硬编码的三分支白名单，没有 `android` 分支，加载即抛错；而 Android 本就是 Linux，该 inspector 只读 `/proc`、只对自己 spawn 的子进程发信号，两者在 Termux 上都能用，所以并入 linux 分支即可 | `dsh-subprocess-local/lib/runner-launch-*.js` |
| 编译/加载 FFI 相关功能失败 | `statx` 在 Bionic 上被 `__INTRODUCED_IN(30)` 隐藏，C++ 会把它当构造函数；`posix_spawn` file actions 需要 API 28 | `koffi/lib/native/base/base.cc`（编译期） |

关键词对照，方便你判断遇到的问题是否在这个范围内：

```
ERR_FLOCK_UNSUPPORTED_PLATFORM
SEARCH_FAILED / ripgrep launch failed
Could not find @vscode/ripgrep-android-arm64
terminal inspection is unsupported on platform android
EACCES（link / 硬链接）
EACCES: open '/data/data'
```

### grep / glob 为什么用 Termux 的 rg 而不是 npm 包

这一点值得单独说，因为它不是「装个东西」能解决的。

dsh 的搜索工具**不查 `PATH`**，它直接 import `@vscode/ripgrep` 并使用包内记录的 `rgPath`。而 `@vscode/ripgrep` 的 `optionalDependencies` 只发布 `linux` / `darwin` / `win32` 三个平台——**没有 android-arm64**。所以在 Termux 上不是「装不上」，而是 Node 在 import 阶段就抛：

```
Could not find @vscode/ripgrep-android-arm64
```

结果是每次搜索都变成 `SEARCH_FAILED`。补丁的做法是在 Android 上绕过这个包，直接指向 Termux 装的那份 `rg`（`executable.dir` 就是 Termux 的 bin 目录）。

Termux 的 `rg` 是 bionic 构建的，跑得动，语法与上游一致。**不要试图去 npm 上找 Android 版 ripgrep**——不存在。

## dsh 依赖链

理解这个有助于判断补丁打在哪、升级后哪里会变。dsh 是个 cordis 插件体系，主包把上百个 `@deepseek-ai/dsh-*` 平铺依赖，实际运行时装在 `node_modules/@deepseek-ai/dsh/node_modules/` 下。

本补丁涉及的 8 个文件在链上的位置：

```
@deepseek-ai/dsh  (顶层，约 130 个平铺依赖)
├── @deepseek-ai/dsh-app-boot ──────────► node-addon-require-builtin ─┐
│                                            └── node-addon-native-custom-loader
│                                                 └── build/napi/napi-v9-android-arm64/  ← 自建
├── @deepseek-ai/cordis-plugin-loader ───► node-addon-require-builtin
│
├── @deepseek-ai/dsh-web-app
│   └── @deepseek-ai/dsh-tool-fs-search  ◄── 补丁: 指向 Termux 的 rg
│
└── @deepseek-ai/dsh-base
    ├── @deepseek-ai/dsh-session-persistence-jsonl  ◄── 补丁: rename 而非 link
    │   └── @deepseek-ai/node-addon-system  ◄── 补丁: 放行 android
    │       └── @deepseek-ai/node-addon-system-android-arm64/bin/musl/system.node  ← 自建
    ├── @deepseek-ai/dsh-fs-local  ◄── 补丁: link → copyFile(EXCL)
    │   └── @deepseek-ai/dsh-fs-sandbox
    ├── @deepseek-ai/dsh-attachment-local  ◄── 补丁: link → copyFile(EXCL) + 持久化遍历
    ├── @deepseek-ai/dsh-subprocess-local  ◄── 补丁: android → LinuxProcessInspector
    └── @deepseek-ai/dsh-sandbox-local ──► node-addon-system
```

三个关键点：

- **`dsh-base` 是枢纽** —— 会话持久化、文件系统、附件、子进程四个补丁都挂在它下面，所以它一坏就是大面积失效，而不是单个工具坏。
- **原生模块有两套加载器，规则不同**。`node-addon-system` 按 `glibcVersionRuntime` 在 `bin/glibc` / `bin/musl` 间选择；`node-addon-require-builtin` 按 `build/<backend>/<abi>-<platform>/` 查找。写错位置的表现是「文件明明在却加载不到」。
- **Termux 不走 glibc 分支** —— `glibcVersionRuntime` 在 bionic 上为空，所以 flock 产物必须放 `bin/musl`。

## 补丁内容

### `koffi-android.patch`（编译期，3 hunk，1 个文件）

只有自己编译 koffi 时才需要；预编译的 `koffi.node` 已经包含这些改动。

- **`statx` 回退** —— Bionic 用 `__INTRODUCED_IN(30)` 隐藏 `statx` 声明，API < 30 时 C++ 会把 `statx(...)` 当成构造 `struct statx`。排除 `__ANDROID__` 让它落到通用的 `fstatat`/`fstat` 分支。
- **API < 28 禁用 `posix_spawn` file actions** —— 该 helper 需要 API 28。dsh 的 FFI 路径不调用它，直接返回 `ENOSYS`。
- **`return true` 移出 `#endif`** —— 让成功路径在所有平台都返回。

### `dsh-termux-android.patch`（运行期，13 hunk，6 个文件）

### `@deepseek-ai/dsh-subprocess-local`（2 hunk）

`createProcessInspector` 只认 `linux` / `darwin` / `win32`，其他平台直接抛：

```
subprocess-local: terminal inspection is unsupported on platform android
```

Termux 的 `process.platform` 是 `android`，于是插件加载即失败，本地终端功能整个不可用。

Android 本就是 Linux。该 inspector 只读 `/proc/<pid>/stat`、列 `/proc`、调 `ps`/`kill`，且只检查自己 spawn 的子进程，不跨应用触碰 SELinux 边界，因此并入 linux 分支即可。

> 该文件名带内容哈希（`runner-launch-<hash>.js`），dsh 升级后会变。

### `@deepseek-ai/node-addon-system` `lib/flock.js`（1 hunk）

平台白名单只放行 linux/darwin，Termux 上抛 `ERR_FLOCK_UNSUPPORTED_PLATFORM`，**会话落盘整轮失败**。`flock()` 本身 bionic 支持，只需放开判断。

### `@deepseek-ai/dsh-session-persistence-jsonl` `lib/index.js`（2 hunk）

Android 沙箱拒绝 `link(2)`（EACCES），初始 JSONL 会话发布会失败。Android 上改用 `rename()`。

### `@deepseek-ai/dsh-fs-local` `lib/index.js`（2 hunk）

Android 的 SELinux 对应用域禁止创建硬链接，`link()` 无条件 EACCES。`writeFileAtomic` 的 `createIfAbsent` 分支退到 `copyFile(COPYFILE_EXCL)`——它以 `O_CREAT|O_EXCL` 建目标，语义等价。

### `@deepseek-ai/dsh-attachment-local` `lib/index.js`（5 hunk）

三处 `link()` 发布（`publishImmutableAlias`、`publishStagedObject`）同样退到 `copyFile`；另修 `ensureDurableHome` 的持久化遍历——它故意把 `$HOME` 的祖先一路 fsync 到文件系统根，Termux 上 `$HOME=/data/data/com.termux/files/home`，链条会穿过 `/data/data`，而 Android 沙箱禁止应用打开应用数据根，于是每次 `saveImage`/`read_image` 都失败。打不开的层不属于本应用可同步范围，终止遍历而不是让整次保存失败。

### `@deepseek-ai/dsh-tool-fs-search` `lib/index.js`（1 hunk）

dsh 的 grep/glob 不查 PATH，走 `@vscode/ripgrep` 的 `rgPath`，而该包没有 `@vscode/ripgrep-android-arm64`（registry 上只有 linux/darwin/win32），import 直接抛 `Could not find`，于是每次搜索都变成 `SEARCH_FAILED`。改为在 Android 上指向 Termux 的 `rg`。

## 已知限制

### 插件：pnpm 在 Android 上的硬链接问题（不在本补丁范畴）

dsh 用 pnpm 管理 profile 下的插件。**Android 沙箱禁止创建硬链接**，而 pnpm 默认靠硬链接复用 store 里的包，所以插件安装会失败或留下损坏的 `node_modules`。

这不是 dsh 的 bug，也不是本补丁能修的——补丁只覆盖 dsh 自带的包，**插件不在范围内**。需要在 profile 的 `pnpm-workspace.yaml` 里加两项：

```yaml
nodeLinker: hoisted
packageImportMethod: copy
```

- `packageImportMethod: copy` —— 不用硬链接，改成复制。**这条是必须的。**
- `nodeLinker: hoisted` —— 把依赖树拍平成 dsh 期望的扁平 `node_modules` 结构

另外两点经验：

- 装带 `install`/`prepare` 脚本的插件时加 `--ignore-scripts`。有些插件依赖 Linux 原生二进制（如 `cloudflared`），在 Android 上编不出来，而 pnpm 12 会把「被忽略的构建脚本」当硬错误，直接中止整次安装。
- 从 GitHub 装比从 npm 装更容易失败，因为它需要 `git ls-remote`。npm registry 上有对应包时优先用版本号装，完全绕开 git。

> **注意：** Termux 网络访问 GitHub 经常不通——`git ls-remote` 和 raw 下载会超时或报 `SSL_ERROR_SYSCALL`。
> 而 npm registry 在 Termux 上要稳定得多。所以只要 npm 上有对应包，就用版本号装（`npm i <包>@<版本>`），
> 别用 GitHub spec。同样的道理适用于 Release 里的 tarball：如果你那边 `github.com` 很慢，
> 那几个二进制完全可以从 npm 源码加本补丁重新构建出来。

### 其他

- **无沙箱后端** —— Termux 没有 Landlock/bubblewrap，容器隔离不可用。命令执行需在 Web UI 里选 Manual Approval 或 Full Access。
- **`link()` 语义退化** —— 硬链接在 Android 上不可用，所有原子发布走 `copyFile`。功能等价，但失去硬链接的 inode 共享特性。
- **flock 二进制** —— 需要自建，或从 Release 下载。
- **版本绑定** —— 补丁针对 dsh 0.2.0-rc.2。dsh 是 developer preview，升级后大概率需要重新生成补丁：拿新版本的干净 npm 原文，重新 `diff -u`。
- **`subprocess-local` 文件名哈希** —— dsh 升级后哈希变化，patch 文件里的路径要同步更新。

## 卸载

```bash
npm rm -g @deepseek-ai/dsh
```

补丁只改安装目录下的文件，不写其他位置；自己建到 `node_modules/*-android-arm64/` 的目录也在卸载范围内。

## 许可

MIT，见 [LICENSE](LICENSE)。补丁修改的是第三方作品，其版权声明保留在补丁后的文件里：DeepSeek Harness（© 2026 DeepSeek）与 koffi（© 2026 Niels Martignène）。
