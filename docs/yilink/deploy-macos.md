# YiLink macOS 部署指南

> 版本：1.5.0（基于 RustDesk 1.5.0）
> 适用：macOS 10.15+（Intel/Apple Silicon）

## 1. 获取安装包

从 GitHub Actions 下载最新构建产物 `YiLink-1.5.0-macos-x86_64.dmg` 或 `YiLink-1.5.0-macos-aarch64.dmg`。

**注意**：macOS 构建需用户 GitHub 账号 fork 仓库并启用 CI，详见 [build-macos.md](build-macos.md)。

## 2. 安装与权限配置

### 2.1 绕过 Gatekeeper

由于应用未签名，首次运行需绕过 Gatekeeper：

```bash
# 方法1：移除隔离属性
xattr -cr /Applications/YiLink.app

# 方法2：系统偏好设置 → 安全性与隐私 → 通用 → 仍要打开
```

### 2.2 授予必要权限

YiLink 需要以下权限才能正常工作：

1. **屏幕录制**（必需）
   - 系统偏好设置 → 安全性与隐私 → 隐私 → 屏幕录制
   - 勾选 YiLink

2. **辅助功能**（必需，用于远程控制）
   - 系统偏好设置 → 安全性与隐私 → 隐私 → 辅助功能
   - 勾选 YiLink

3. **完全磁盘访问**（可选，用于文件传输）
   - 系统偏好设置 → 安全性与隐私 → 隐私 → 完全磁盘访问
   - 勾选 YiLink

### 2.3 启动应用

```bash
open /Applications/YiLink.app
```

或从启动台/应用程序文件夹双击打开。

## 3. 防火墙配置

macOS 内置防火墙默认放行局域网连接。如启用严格模式，需手动放行：

```bash
# 查看防火墙状态
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# 添加 YiLink 例外
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /Applications/YiLink.app
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /Applications/YiLink.app
```

## 4. 连接方法

### 4.1 主控端（控制方）

1. 在"控制远程桌面"输入框输入：`被控IP:21118`
2. 点击"连接"
3. 输入被控端密码

### 4.2 被控端（被控制方）

1. 确保 YiLink 已启动
2. 主界面显示本机 ID 和一次性密码
3. 确保已授予屏幕录制与辅助功能权限

## 5. 跨平台连接

Mac 主控连接 Windows 被控：
- Windows 被控端 IP + `:21118`
- 输入 Windows 端显示的密码

Windows 主控连接 Mac 被控：
- Mac 被控端 IP + `:21118`
- 输入 Mac 端显示的密码

## 6. 常见问题

**Q: 启动后提示"已损坏，无法打开"**
A: 执行 `xattr -cr /Applications/YiLink.app` 移除隔离属性。

**Q: 无法远程控制，只能观看**
A: 检查辅助功能权限是否已授予 YiLink。

**Q: 画面黑屏或卡顿**
A: 检查屏幕录制权限；尝试降低画质设置。

## 7. 卸载

```bash
# 删除应用
rm -rf /Applications/YiLink.app

# 删除配置
rm -rf ~/Library/Preferences/com.yilink.remote.plist
rm -rf ~/Library/Application\ Support/YiLink

# 删除日志
rm -rf ~/Library/Logs/YiLink
```
