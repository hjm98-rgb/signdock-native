# 仓库记忆：Signdock 原生 App

> 此文件随仓库提交，clone 到任何机器都会自动加载进 CodeBuddy 上下文。
> **权威手册：`docs/CLOUD-BUILD.md`**（含完整 workflow / project.yml / ExportOptions.plist）。

## 这个项目是什么
把网页签名控制台的全部功能，用**全新原生 SwiftUI 工程**重做，借助 **GitHub Actions macOS runner** 云编译真机 `.ipa`。
**绝不是网页套壳** —— 套壳做不了 Apple ID 签名（需 Keychain / Anisette / 本地网络 / 直接装主屏幕）。

## 每轮开工前必读
1. `docs/CLOUD-BUILD.md`：完整打包方法、GitHub Secrets、9 个视图要求、Apple ID 签名边界、验收清单。
2. 用户级记忆 `~/.codebuddy/CODEBUDDY.md` 与 `~/.codebuddy/rules/signdock-cloud-build.md`：账号、凭据位置、全局红线。

## 不可动摇的红线
- 凭据只在本地 600 权限文件 / GitHub Secrets，**绝不写进本仓库、绝不回显**。
- 涉及 Apple ID 登录 / 2FA / 证书信任**必须暂停让用户手动处理**，不自动代登录、不上传密码。
- 不做假按钮 / 假进度条 / 假成功 / 共享证书 / 任意网站浏览器。
- 数据按数据空间隔离，只存本机，服务端零留存。
- 「能操作」浏览器只允许 GitHub 相关链接。
- Apple ID 签名：7 天有效期、最多 3 个 App、权限申请需说明 Apple 侧条件与不确定性。

## 常用命令
```bash
xcodegen generate                     # project.yml -> Signdock.xcodeproj
# push 到 main 触发 .github/workflows/build.yml，产物 Signdock-IPA artifact
```
GitHub token 位置见用户级记忆（不写入任何提交文件）。
