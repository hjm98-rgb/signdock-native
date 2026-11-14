# Signdock（原生 iOS App）

把网页签名控制台的全部功能，用**全新原生 SwiftUI 工程**重做一遍，通过 **GitHub Actions 的 macOS runner** 云编译出可直接安装的真机 `.ipa`。

> **不是网页套壳。** 页面和网页版一致，但代码是纯 Swift 原生编译。原因：Apple ID 签名需要 Keychain、Anisette、本地网络与直接安装到主屏幕，WKWebView / PWA 壳做不到。

## 状态

- [x] 全新仓库创建（`hjm98-rgb/signdock-native`，public，main）
- [x] 云编译手册 `docs/CLOUD-BUILD.md`（含可照抄的 workflow / project.yml / ExportOptions.plist）
- [ ] SwiftUI 原生工程 `Signdock/`（9 个视图）
- [ ] Apple ID 签名原生模块
- [ ] 首次 CI 编译通过、产出可安装 IPA

## 快速开始

完整方法见 **[docs/CLOUD-BUILD.md](docs/CLOUD-BUILD.md)** —— 包含账号/凭据、GitHub Secrets、`project.yml`、`ExportOptions.plist`、`build.yml` 全文、九个视图的功能要求、Apple ID 签名的真实边界、验收清单。

最小流程：

```bash
brew install xcodegen
xcodegen generate          # project.yml -> Signdock.xcodeproj
# push 到 main 即触发 .github/workflows/build.yml
# Actions 产出 Signdock-IPA artifact
```

## 两种签名模式

| 模式 | 说明 |
|---|---|
| 证书签名 | 个人 p12 + mobileprovision，走真实三引擎（zsign / Feather / Anvil）+ OTA 安装 |
| Apple ID 签名 | 手动登录 Apple ID + 2FA，直接装到主屏幕；**7 天有效期**、**最多 3 个 App**、带权限（capabilities）申请页 |

涉及 Apple ID 登录 / 2FA / 证书信任一律由用户手动完成，不自动代登录、不上传密码。

## 安全与合规

- 凭据只在用户本机 / GitHub Secrets，**绝不写进仓库**。
- 数据按数据空间隔离，只存本机，服务端零留存。
- 「能操作」浏览器只允许 GitHub 相关链接，不放开任意网站。
- 仅用于用户自有权利或明确开源且允许衍生的应用；制作应用带免责协议硬门禁。
