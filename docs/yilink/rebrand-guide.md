# YiLink 品牌改造指南

> 基线：RustDesk 1.5.0（commit `e9ddbd8f460dac1b30b9815ba324a5dadc8d97b0`）
> 工具：`scripts/apply-branding.ps1`

## 1. 品牌参数

| 参数 | 值 | 说明 |
|---|---|---|
| Brand | `YiLink` | 英文品牌名 |
| BrandCn | `易连远程` | 简体中文品牌名 |
| BrandCnTw | `易連遠端` | 繁体中文品牌名 |
| CopyrightHolder | `YiLink` | 版权主体 |

## 2. 补丁清单（20 项，13 个文件）

| 编号 | 触点 | 文件 | 说明 |
|---|---|---|---|
| P1 | 品牌名根触点 | `libs/hbb_common/src/config.rs:72` | `APP_NAME` 默认 `"RustDesk"` → `"YiLink"` |
| P2 | 局域网直连默认 | `libs/hbb_common/src/config.rs` | `direct-server` 选项出厂默认 `"Y"` |
| P3 | Windows 可执行文件名 | `flutter/windows/CMakeLists.txt` | `BINARY_NAME` rustdesk → yilink |
| P4 | Windows 版本信息 | `flutter/windows/runner/Runner.rc` | 产品名/公司名/版权行品牌化 |
| P5 | 便携自解压包名 | `build.py` | portable packer 输出 exe 名品牌化 |
| P6 | 关于页标题（默认/英文） | `src/lang/template.rs` | 关于对话框标题 YiLink |
| P7 | 关于页标题（简体） | `src/lang/cn.rs` | 同上，简体中文 |
| P8 | 关于页标题（繁体） | `src/lang/tw.rs` | 同上，繁体中文 |
| P9 | 关于页外链 | `flutter/lib/desktop/pages/desktop_setting_page.dart` | 删除官网/隐私外链 |
| P10 | 关于页版权主体 | `flutter/lib/desktop/pages/desktop_setting_page.dart` | 版权行主体 RustDesk → YiLink |
| P11 | pubspec 描述 | `flutter/pubspec.yaml` | description 品牌化 |
| P12 | 桌面 TabBar 标题 | `flutter/lib/desktop/widgets/tabbar_widget.dart` | 硬编码 "RustDesk" → "YiLink" |
| P13 | powered-by 页脚 | `flutter/lib/common.dart` | 隐藏 "Powered by RustDesk" 页脚 |
| P14a | 安装页隐私外链 | `flutter/lib/desktop/pages/install_page.dart` | 隐私政策外链改为静态文本 |
| P14b | 安装页无用 import | `flutter/lib/desktop/pages/install_page.dart` | 删除 url_launcher_string import |
| P15 | 公共服务器引导 | `flutter/lib/desktop/pages/connection_page.dart` | 隐藏 rustdesk.com/pricing 引导卡 |
| P16 | 设置页无用 import | `flutter/lib/desktop/pages/desktop_setting_page.dart` | 删除 url_launcher_string import |
| P17 | runner 回退窗口标题 | `flutter/windows/runner/main.cpp` | `app_name` 回退值 L"RustDesk" → L"YiLink" |
| P18 | 便携打包解释器名 | `build.py` | `pip3`/`python3` → `pip`/`python` |
| P19 | 安装器输出名 | `build.py` | `rustdesk-{version}-install.exe` → `yilink-{version}-install.exe` |
| P20 | packer 版本信息 | `libs/portable/Cargo.toml` | winres 元数据品牌化 |

## 3. 品牌资产（26 件，4 个投放目标）

源图由 `scripts/make_placeholder_icon.py` 程序化生成，`scripts/make_branding_assets.py` 导出：

| 投放目标 | 内容 |
|---|---|
| `flutter/assets/` | icon.png(1024)、icon.ico（多帧 16-256）、logo.png / logo_light.png / logo_dark.png(1024)、mac-icon.png(1024) |
| `flutter/windows/runner/resources/` | app_icon.ico（窗口/任务栏） |
| `res/` | icon.ico(128 多帧)、app_icon.ico(48)、tray-icon.ico(32)、128x128.png、128x128@2x.png(256)、64x64.png、32x32.png |
| macOS（13 件） | 供 iconutil 生成 icns（见 Task 10） |

## 4. 应用补丁

```powershell
# 预演（不写盘）
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -DryRun

# 实跑（幂等）
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk

# 仅改代码不重投资产
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -SkipAssets

# 保留公服引导（不推荐，仅调试用）
powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -SkipDirectDefault
```

## 5. 更换品牌

1. 修改 `scripts/apply-branding.ps1` 顶部参数：
   ```powershell
   param(
       [string]$Brand = 'NewBrand',
       [string]$BrandCn = '新品牌',
       [string]$BrandCnTw = '新品牌',
       [string]$CopyrightHolder = 'NewBrand'
   )
   ```

2. 替换品牌资产源图：
   - `assets/branding/src/icon.png`（1024x1024 主图标）
   - `assets/branding/src/logo.png` / `logo_light.png` / `logo_dark.png`

3. 重跑管线：
   ```powershell
   python scripts/make_branding_assets.py
   powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk
   ```

## 6. 升级上游

1. 拉取新 tag：
   ```bash
   cd third_party/rustdesk
   git fetch --tags
   git checkout 1.5.1  # 示例新版本
   ```

2. 检查补丁冲突：
   ```powershell
   powershell -File scripts/apply-branding.ps1 -RepoRoot D:\Agent\remotedesk -DryRun
   ```

3. 如有冲突，手动调整补丁后重新应用

## 7. 合规检查清单

- [ ] 根目录 `LICENSE` 文件存在且为 AGPL-3.0
- [ ] 关于页保留开源许可入口（AGPL LICENSE 按钮）
- [ ] 未删除任何上游版权声明
- [ ] 修改清单完整记录（本文件）
- [ ] 二开源码在分发时提供对应源码获取方式
