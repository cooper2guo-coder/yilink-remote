# YiLink（易连远程）Windows 构建指南

> 基线：RustDesk **1.5.0**（commit `e9ddbd8f460dac1b30b9815ba324a5dadc8d97b0`）
> 构建主机：Windows 11 x64，非管理员账户（VS 安装需一次 UAC）
> 工具链全部按上游 1.5.0 tag 自带 CI（`.github/workflows/ci.yml`）锚定，勿随意升级。

## 1. 工具链版本（实际安装记录）

| 组件 | 版本 | 位置 | 说明 |
|---|---|---|---|
| VS 2022 Build Tools | 17.14.41（MSVC 14.44.35207） | `C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools` | VCTools x64 + Win11 SDK 22621 + CMake/Ninja 组件 |
| CMake | 3.31.6-msvc6 | VS 自带 `...\CommonExtensions\...\CMake\CMake\bin` | 已注册用户 PATH；vcpkg 实际使用自带的 4.4.0 |
| Ninja | 1.12.1 | VS 自带 `...\CommonExtensions\...\CMake\Ninja` | 已注册用户 PATH |
| Rust（默认） | stable 1.98.1 msvc | `third_party\rust\cargo` | 经清华/rsproxy 镜像安装 |
| Rust（构建用） | **1.75.0 msvc** | 同上，`third_party\rustdesk` 目录 `rustup override` | CI 锚点；仓库无 rust-toolchain.toml |
| LLVM/Clang | **15.0.6** | `third_party\llvm` | `LIBCLANG_PATH` 指向 bin；rust-bindgen 依赖 |
| Flutter | **3.24.5**（Dart 3.5.4） | `third_party\flutter-sdk` | 需替换 RustDesk 定制 engine + dropdown 补丁（见第 4 节） |
| NASM | 2.16.03 | `third_party\nasm` | aom 汇编依赖 |
| Python | 3.12.10（Pillow 12.3.0） | 系统 Python + `third_party\python-pkgs` | build.py 与品牌资产脚本 |
| vcpkg | commit `9e593bb18ea69cc5095e012465dcd675a822ed0d`（2026.07） | `third_party\vcpkg` | 上游 tag 内置 CI 锚点；vcpkg-tool 2026-07-27 bootstrap |
| Git | 2.54.0 | 系统 | |

用户级持久环境变量（`[Environment]::SetEnvironmentVariable(...,'User')`）：
`CARGO_HOME`、`RUSTUP_HOME`、`LIBCLANG_PATH`、`VCPKG_ROOT`、`PUB_CACHE=third_party\pub-cache`，
PATH 追加 rust/cargo/bin、nasm、flutter-sdk/bin、vcpkg、llvm/bin、VS cmake/ninja。

## 2. 一键搭建（幂等，可分阶段断点续跑）

```powershell
# 普通用户工具（无 UAC，落 third_party/）
powershell -ExecutionPolicy Bypass -File scripts\setup-windows.ps1 -Stage Rust,Nasm,Llvm,Flutter,VcpkgClone -GithubProxy https://ghfast.top
# VS Build Tools（会弹一次 UAC）
powershell -ExecutionPolicy Bypass -File scripts\setup-windows.ps1 -Stage VsBuildTools
# vcpkg 静态依赖（manifest 模式，最耗时：下载+编译约 1-3 小时）
powershell -ExecutionPolicy Bypass -File scripts\setup-windows.ps1 -Stage VcpkgLibs -GithubProxy https://ghfast.top
```

环境自检（新开终端后应 11/11）：

```powershell
powershell -ExecutionPolicy Bypass -File scripts\check-env.ps1
# vcpkg 依赖核对（在 third_party\rustdesk 下执行）
vcpkg list # 应含 libvpx / libyuv / opus / aom / ffmpeg 等 x64-windows-static
```

## 3. vcpkg 浅克隆注意事项（重要）

vcpkg 为浅克隆（`--depth 1`），而 rustdesk `vcpkg.json` 的 overrides 固定了
`amd-amf@1.4.35`、`ffnvcodec@12.1.14.0` 两个历史版本；vcpkg 解析 versions 时需要
历史 git tree，浅库会报 `read-tree failed / failed to unpack tree object`。

**不要 `git fetch --unshallow`**（全历史 1.5GB+）。本项目采用 overlay port 方案：
已将这两个版本的 port 文件（来自上游 bump commit `be966f38` / `3c58f930`）放入
`third_party/rustdesk/res/vcpkg/{amd-amf,ffnvcodec}/`（`vcpkg.json` 本就声明该目录为
overlay-ports）。overlay 同名 port 直接覆盖 registry 版本解析，不再触碰历史 tree。

## 4. 基线构建（未改版源码先验证）

```powershell
# 1) pin Rust 1.75 + flutter precache + 定制 engine 覆盖 + dropdown 补丁
powershell -ExecutionPolicy Bypass -File scripts\prepare-build.ps1 -RepoRoot D:\Agent\remotedesk -GithubProxy https://ghfast.top
# 2) 构建 portable（在 third_party\rustdesk 目录）
python build.py --portable --flutter --skip-portable-pack --hwcodec --vram
```

产物：`flutter\build\windows\x64\runner\Release\rustdesk.exe`（品牌化后为 `yilink.exe`）。

定制 engine 固定来源：`https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip`
（60.4MB，资产冻结于 2024-12-01，与 1.5.0 / Flutter 3.24.5 同期）。

## 5. 品牌化与直连默认

```powershell
# 先准备资产（AI 通道不可用时用程序化占位图标）
$env:PYTHONPATH='D:\Agent\remotedesk\third_party\python-pkgs'
python scripts\make_placeholder_icon.py     # 可选：占位源图
python scripts\make_branding_assets.py      # 导出 26 个资产到 assets/branding/dist
# DryRun 校验 16 个补丁全部唯一匹配后再实跑
powershell -File scripts\apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -DryRun
powershell -File scripts\apply-branding.ps1 -RepoRoot D:\Agent\remotedesk
```

补丁清单（P1-P16）逐项登记于 `docs/REBRAND-PATCHES.md`（Task 5 应用时生成）。
核心：P1 `APP_NAME=YiLink`（配置目录/服务名/窗口名联动）、P2 `direct-server` 出厂默认 `"Y"`
（被控端监听 TCP **21118**，主控用 `IP:21118` + 永久密码直连，无需公网服务器）。

## 6. 本机网络与排障备忘

- 本地代理 `127.0.0.1:7897`：curl/schannel 与 vcpkg 自动走它直连 GitHub。
- 出口带宽限速约 60KB/s：**多个 GitHub 大流量严禁并行**（会 connection reset 35/56），严格串行。
- 慢速代理下大文件（PowerShell Core、engine）若连接挂死（.part 0 字节不增长），
  用 `curl -L -C - --retry 30 --retry-all-errors --speed-time 20 --speed-limit 2048`
  手动预下载到 vcpkg downloads / engine 缓存目录，哈希匹配后工具会跳过下载。
- 含中文的 `.ps1` 必须保存为 **UTF-8 with BOM**，否则 PowerShell 5.1 按 GBK 解析导致引号不闭合。
- Flutter/cargo 写 AppData 会被部分沙箱拦截：构建命令在沙箱外终端执行；`PUB_CACHE` 已指向工作区。
- NSIS 安装器（LLVM）后台静默安装可能被 UAC 取消，需在可交互会话执行并点一次"是"。
