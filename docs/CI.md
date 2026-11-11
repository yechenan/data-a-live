# CI：这账号上能跑什么、为什么改了源码

记录于 2026-11-11，第一次在本仓库跑通 `xnu.yml`。

## runner 实测（`runner-probe` 矩阵，一次性）

| 标签 | 结果 |
|---|---|
| `macos-26` / `macos-latest` | **跑起来了**（arm64，macOS 26.7.2，只有 Xcode 26.0 / 26.0.1） |
| `macos-13`、`macos-14-intel`、`macos-15`、`macos-15-intel` | 一直 queued，这账号没有容量 |
| `macos-14`、`macos-14-large` | 秒取消：`The configuration 'macos-14-arm64-us-default' is not supported` |

结论：`runs-on` 只能是 `macos-26`。原来写 `macos-14` 是为了拿 Xcode 15 的 `iig`，
但这个账号上那个标签直接不可用——所以只有一条路：**用 Xcode 26，同时让 `xnu-8796` 能接受新 `iig`**。

## 为什么加了 `0333-iokitcpp-kernel-content.patch`

Xcode 16 起的 `iig` 生成的 DriverKit 代码会写 `IORPC.kernelContent`，而 `xnu-8796` 的
`struct IORPC` 只有四个成员 → `no member named kernelContent in IORPC`，整个 DriverKit 段编不过。
`xnu-10063` 加了同一个成员（`#ifdef KERNEL` 内）。0333 就是把那一处搬过来，
`iokit/IOKit/IORPC.h` 与 `iokit/DriverKit/IORPC.h` 两个副本都改，用户态结构体大小不变。

## 一次成功构建的产物（run 43012468378，11 分钟）

| 产物 | 内容 |
|---|---|
| `xnu-kernel` | `kernel.development.pandora`（16,813,944 字节，Mach-O arm64，sha256 `20749c03…`）+ `kernel.development.pandora.link/`（128 MB：静态归档 `.a`、`.exp`、`.linkarguments`、`.dSYM`）+ KDK 支持库清单 |
| `xnu-buildtree` | `build/xnu.obj`、`fakeroot/usr`、`sdk-kernel-platform/`、打过补丁的 `xnu/` —— 本地增量编译的基础 |
| `xnu-objects` | 1020 个 `.o`、139 个 `.cpo`、`objects.tgz`、`linklibs.tgz`（`libclang_rt.cc_kext*.a`、`libfirehose_kernel.a`） |

## 两个运维上的坑

1. **run 日志下载会被截断**：`GET /actions/runs/{id}/logs` 在这台机器上反复拿到不完整的 zip
   （没有 central directory）。不阻塞：编译命令行可以从 `buildtree` 里的构建目录重新生成，
   链接参数在 `.linkarguments` 里。
2. **`keep_going: true` 要开着**：一次 `make -k` 把所有编译错误报全，省得一轮试一个。

## 本地增量编译还差什么

`wsl/local_build.sh` 依赖 `xnu-cc.py`、`local_relink.py`、`/opt/work/ld64`，这些都是原作者机器上的
（不在仓库里）。本机已有 `ld64.lld`（lld-18）、iPhoneOS16.5 SDK、`fakeroot` 的两个内核库，
缺的是 cctools 的 `ld64`+`strip`（可源码编译）。补齐后就能做单文件重编 + 重链，
`kernel.development.pandora.link/kernel.development.pandora.a` 与 `.linkarguments` 正好是重链的输入。
