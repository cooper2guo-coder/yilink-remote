# YiLink macOS 构建指南

> 版本：1.5.0（基于 RustDesk 1.5.0，分支 `yilink-brand-1.5.0`）
> CI 方案：GitHub Actions `yilink-macos.yml`（本地为 Windows，无法交叉编译 macOS，故云构建为主）
> 部署与权限细节见 [deploy-macos.md](deploy-macos.md)，Windows 端见 [build-windows.md](build-windows.md)

## 1. 构建方式概览

| 方式 | 适用场景 | 产物 |
|---|---|---|
| GitHub Actions 云构建（推荐） | 无 Mac 设备、正式出包 | `yilink-macos-<arch>` artifact 内含 dmg |
| 本地 Mac 构建 | 有 Mac 设备、调试 | `flutter/build/macos/Build/Products/Release/RustDesk.app` |

> **已验证构建（2026-09-30）**：[Run 36586195219](https://github.com/cooper2guo-coder/yilink-remote/actions/runs/36586195219) `completed / success`，双架构均绿。产物已下载至仓库 `dist/`：`yilink-1.5.0-macos-aarch64.dmg`（29.9 MB）、`yilink-1.5.0-macos-x86_64.dmg`（36.2 MB）。

## 2. CI 云构建（推荐）

### 2.1 前置条件

1. GitHub 账号
2. 将本仓库（含 `.github/workflows/yilink-macos.yml` 与 `.github/workflows/bridge.yml`）推送到 GitHub，
   分支 `yilink-brand-1.5.0`。品牌补丁与 workflow 均已随源码提交，**无需再手工打补丁**
3. 不需要任何 secrets：未配置签名证书时自动跳过签名/公证步骤，产出 ad-hoc 签名包（见第 4 节）

### 2.2 触发构建

1. 进入 GitHub 仓库页面 → **Settings** → **Actions** → **General**：
   - **Actions permissions** 选择 "Allow all actions and reusable workflows"
2. 顶部 **Actions** 标签页 → 左侧选择 **Build YiLink macOS**
3. 点击 **Run workflow**：
   - **Branch**: 选择 `yilink-brand-1.5.0`
   - **upload-tag**: 默认 `yilink-test`（预留参数，当前不影响产物）
4. 点击 **Run workflow** 开始构建

构建包含两个 job，串行依赖：

| Job | Runner | 耗时参考 |
|---|---|---|
| `generate-bridge` | ubuntu-22.04（生成 flutter-rust-bridge 桥接代码） | 约 15~25 分钟 |
| `build-for-macOS`（矩阵 ×2） | macos-15-intel（x86_64）/ macos-14（aarch64） | 约 40~90 分钟 |

> 首次构建 vcpkg 依赖（ffmpeg/hwcodec 等）无缓存，耗时偏长；后续命中 Actions 缓存会明显加快。

### 2.3 下载产物

构建成功后，在该次运行页面底部 **Artifacts** 区下载：

| Artifact | 内含 dmg | 适用 |
|---|---|---|
| `yilink-macos-x86_64` | `yilink-1.5.0-macos-x86_64.dmg` | Intel Mac |
| `yilink-macos-aarch64` | `yilink-1.5.0-macos-aarch64.dmg` | Apple Silicon (M 系列) |

说明：

- artifact 由 `actions/upload-artifact` 无条件上传，不依赖 tag/release；zip 解压即得 dmg
- dmg 内应用包名当前仍为 `RustDesk.app`（macOS 端 `PRODUCT_NAME` 品牌化尚未应用），
  dmg 外层文件名已改为 `yilink-*`，不影响功能验证
- Artifacts 默认保留 90 天；如需长期分发，建议发布 Release 或自行归档

## 3. 本地 Mac 构建（备选）

### 3.1 环境要求

- macOS 12+（Apple Silicon 产物目标系统最低 12.3）
- Xcode 15+（或至少 Xcode Command Line Tools）
- **Rust 1.81**（macOS 端依赖 `cidre`，必须 ≥1.81，与 CI 的 `MAC_RUST_VERSION` 一致）
- **Flutter 3.24.5**（与 CI 的 `FLUTTER_VERSION` 一致）
- vcpkg（与 CI 同一 commit：`9e593bb18ea69cc5095e012465dcd675a822ed0d`）

### 3.2 环境准备

```bash
# Xcode 命令行工具
xcode-select --install

# Rust 1.81
rustup toolchain install 1.81.0
rustup default 1.81.0

# Flutter 3.24.5
git clone https://github.com/flutter/flutter.git -b 3.24.5 --depth 1 ~/flutter
export PATH="$HOME/flutter/bin:$PATH"
flutter doctor

# vcpkg（与 CI 同 commit）
git clone https://github.com/microsoft/vcpkg.git ~/vcpkg
cd ~/vcpkg && git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d && ./bootstrap-vcpkg.sh && cd -
export VCPKG_ROOT=~/vcpkg
```

### 3.3 构建步骤

```bash
git clone -b yilink-brand-1.5.0 https://github.com/<YOUR_ACCOUNT>/<YOUR_REPO>.git
cd <YOUR_REPO>
git submodule update --init --recursive

# 与 CI 相同的构建参数
python3 build.py --flutter --hwcodec --unix-file-copy-paste
```

### 3.4 产物

应用位于 `flutter/build/macos/Build/Products/Release/RustDesk.app`，
可直接运行验证；如需分发可自行用 `create-dmg` 打包。

## 4. 签名现状与 Gatekeeper

### 4.1 当前状态：ad-hoc 签名（无 Apple Developer 证书）

- CI 与本地构建均使用 Xcode 默认的 **ad-hoc 签名**（`CODE_SIGN_IDENTITY = "-"`）
- 未配置签名 secrets 时，`yilink-macos.yml` 中的证书导入、rcodesign、公证步骤
  （均带 `if: env.MACOS_P12_BASE64 != null && env.MACOS_P12_BASE64 != ''` 条件）会自动跳过
- 分发后首次运行会被 Gatekeeper 拦截，需手动绕过

### 4.2 用户侧绕过 Gatekeeper

```bash
# 方法 1：移除隔离属性后正常打开
xattr -cr /Applications/YiLink.app

# 方法 2：右键点击应用 → 打开（弹窗中再点"打开"）

# 方法 3：系统设置 → 隐私与安全性 → 底部"仍要打开"
```

### 4.3 正式签名（可选，生产分发）

如需消除 Gatekeeper 拦截，在仓库 **Settings → Secrets and variables → Actions** 配置以下
secrets 后重新 Run workflow，CI 会自动走证书签名 + 公证 + staple 流程（workflow 无需改动）：

| Secret | 说明 |
|---|---|
| `MACOS_P12_BASE64` | Developer ID Application 证书 .p12 的 base64 |
| `MACOS_P12_PASSWORD` | .p12 导出密码 |
| `MACOS_CODESIGN_IDENTITY` | 签名身份，如 `Developer ID Application: XXX` |
| `MACOS_NOTARIZE_JSON` | App Store Connect API key（JSON，供 rcodesign 公证） |

> 生成 .p12 base64：`base64 -i developerid.p12 | pbcopy`（Mac）或
> `certutil -encode developerid.p12 out.txt`（Windows，取 out.txt 内容体）。

## 5. 运行时权限授予

安装后需按 [deploy-macos.md](deploy-macos.md) 第 2 节引导用户授予：

- **屏幕录制**（被控必需）
- **辅助功能**（远程控制必需）
- 完全磁盘访问（可选，文件传输）

## 6. 验证清单

- [ ] CI 两个 job 全绿，Artifacts 中出现 `yilink-macos-x86_64` 与 `yilink-macos-aarch64`
- [ ] dmg 解压安装后应用可启动（执行过 `xattr -cr` 或右键打开）
- [ ] 屏幕录制、辅助功能权限引导正常
- [ ] Mac 主控 ↔ Windows 被控互连正常
