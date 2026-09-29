# YiLink Windows 部署指南

> 版本：1.5.0（基于 RustDesk 1.5.0）
> 适用：Windows 10/11 x64

## 1. 安装方式

### 方式一：安装版（推荐）

下载 `dist/yilink-1.5.0-install.exe`，双击运行：

1. 安装程序请求管理员权限（UAC），点击"是"
2. 默认安装到 `C:\Program Files\YiLink\`（或自定义路径）
3. 自动创建：
   - 开始菜单快捷方式 `YiLink\YiLink.lnk`
   - 桌面快捷方式 `YiLink.lnk`
   - 开机自启项（HKCU Run 注册表）
   - 防火墙入站规则 `YiLink LAN Direct (TCP 21118)`
4. 安装完成后自动启动主界面

### 方式二：便携版

下载 `dist/YiLink-1.5.0-windows-x86_64-portable.zip`，解压后运行 `yilink.exe`。

**注意**：便携版不会自动创建防火墙规则，需手动添加：
```powershell
# 管理员 PowerShell
New-NetFirewallRule -DisplayName "YiLink LAN Direct (TCP 21118)" -Direction Inbound -LocalPort 21118 -Protocol TCP -Action Allow
```

## 2. 首次启动配置

### 2.1 界面概览

启动后显示主界面：
- **你的桌面**：本机 ID（如 `191 384 162`）和一次性密码（每次重启变化）
- **控制远程桌面**：输入对方 ID 或 IP:21118 连接

### 2.2 设置永久密码

方式一：UI 设置
1. 点击主界面右上角菜单 → 设置 → 安全
2. 设置"永久密码"

方式二：配置文件
编辑 `%APPDATA%\YiLink\config\YiLink.toml`：
```toml
password = 'YourPassword'
```

### 2.3 开机自启

安装版默认已添加开机自启。验证：
```powershell
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "YiLink"
```

## 3. 连接方法

### 3.1 主控端（控制方）

1. 在"控制远程桌面"输入框输入：`被控IP:21118`
2. 点击"连接"
3. 输入被控端密码（一次性密码或永久密码）

### 3.2 被控端（被控制方）

1. 确保 YiLink 已启动并监听 21118 端口
2. 主界面显示本机 ID 和一次性密码
3. 防火墙规则已放行 TCP 21118

## 4. 防火墙配置

安装版自动创建规则。手动验证：
```powershell
Get-NetFirewallRule -DisplayName "*YiLink*"
```

手动添加（如便携版）：
```powershell
New-NetFirewallRule -DisplayName "YiLink LAN Direct (TCP 21118)" -Direction Inbound -LocalPort 21118 -Protocol TCP -Action Allow
```

## 5. 卸载

### 安装版

1. 开始菜单 → `YiLink\Uninstall YiLink.lnk`
2. 或运行 `D:\Program Files\YiLink\YiLink.exe --uninstall`（需管理员）

卸载后清理：
- 安装目录自动删除
- 防火墙规则手动删除：
  ```powershell
  Get-NetFirewallRule -DisplayName "*YiLink*" | Remove-NetFirewallRule
  ```

### 便携版

直接删除解压目录即可。

## 6. 常见问题

**Q: 连接提示"密码错误"**
A: 检查被控端密码是否输入正确（一次性密码每次重启变化，永久密码需在设置中配置）。

**Q: 连接超时**
A: 检查被控端防火墙是否放行 TCP 21118，两台设备是否在同一局域网。

**Q: 远程画面卡顿**
A: 局域网带宽不足或被控端硬件性能限制，尝试降低画质设置。

## 7. 日志与排查

日志位置：`%APPDATA%\YiLink\log\`

关键日志：
- `YiLink.log`：主程序日志
- `YiLink_service.log`：服务日志（安装版）
