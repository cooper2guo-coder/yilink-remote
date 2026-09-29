# 公共服务器恢复说明

> 日期：2026-09-28
> 变更：回退 NG1 约束，恢复 RustDesk 官方公共服务器连接支持

## 修改内容

### 1. P9 回退：关于页外链恢复

文件：`flutter/lib/desktop/pages/desktop_setting_page.dart`

恢复前（P9 删除状态）：
```dart
// [YiLink] 纯局域网产品：移除官方网站/隐私声明外链入口。
Container(...)
```

恢复后（原始状态）：
```dart
InkWell(
    onTap: () {
      launchUrlString('https://rustdesk.com/privacy.html');
    },
    child: Text(
      translate('Privacy Statement'),
      style: linkStyle,
    ).marginSymmetric(vertical: 4.0)),
InkWell(
    onTap: () {
      launchUrlString('https://rustdesk.com');
    },
    child: Text(
      translate('Website'),
      style: linkStyle,
    ).marginSymmetric(vertical: 4.0)),
Container(...)
```

### 2. P15 回退：公共服务器引导恢复

文件：`flutter/lib/desktop/pages/connection_page.dart`

恢复前（P15 修改状态）：
```dart
// [YiLink] 纯局域网直连产品：不显示"配置公共/自建服务器"引导（原跳 rustdesk.com/pricing）。
_svcIsUsingPublicServer.value = false;
```

恢复后（原始状态）：
```dart
_svcIsUsingPublicServer.value = await bind.mainIsUsingPublicServer();
```

### 3. apply-branding.ps1 更新

新增 `-EnablePublicServer` 开关：

```powershell
# 默认（局域网直连模式）：隐藏公共服务器引导
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk

# 启用公共服务器：恢复 rustdesk.com 引导与外链
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -EnablePublicServer
```

## 影响范围

| 功能 | 局域网直连模式 | 公共服务器模式 |
|---|---|---|
| 关于页外链 | 隐藏 | 显示 rustdesk.com/privacy |
| 连接页引导 | 隐藏 | 显示"配置公共服务器"卡片 |
| 连接方式 | IP:21118 直连 | ID 直连 + IP:21118 备用 |
| 服务器依赖 | 无 | RustDesk 官方公共服务器 |

## 验证步骤

1. 重新构建：`flutter build windows --release`
2. 启动应用，检查关于页是否显示官网/隐私外链
3. 检查连接页是否显示"配置公共服务器"引导
4. 验证 ID 直连功能（无需输入 IP:端口）

## 验证结果（2026-09-29）

构建成功（`flutter build windows --release`，199.5s，Exit 0），运行验证：

1. **P15 公共服务器引导**：主界面底部状态栏显示"就绪，如果需要更快连接速度，你可以选择自建服务器"（默认纯局域网模式下此行被隐藏）——证据 `evidence/task-8/public-server-main.png`。
2. **P9 关于页外链**：设置→关于页显示"隐私声明"和"网站"两个超链接，标题仍为"关于易连远程 YiLink"、版权仍为"Copyright © 2026 YiLink"——证据 `evidence/task-8/about-final.png`。
3. ID 直连与 IP:21118 直连并存（direct-server 默认 Y 未改动）。

安装验证（2026-09-29）：新版 `yilink-1.5.0-install.exe` 经安装页「同意并安装」装入 `D:\Program Files\YiLink\`（1.5.0+68，构建于 2026-09-29 19:23）；YiLink 服务 RUNNING，TCP 21118 LISTENING，主界面 ID 191 384 162、状态栏显示公共服务器就绪引导——证据 `evidence/task-7/install-final-main.png`。

构建注意事项：本机代理 127.0.0.1:7897 未运行时，不要设置 HTTP(S)_PROXY（否则 pub get 必失败）；GitHub 直连正常时需移除 git 全局 `url.ghfast.top.insteadOf` 配置；使用 `PUB_HOSTED_URL=https://pub.flutter-io.cn` + `FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn` 镜像。
