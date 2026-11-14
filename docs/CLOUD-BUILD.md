# Signdock 原生 iOS App · 云编译手册

> 这份文档随仓库提交到 GitHub，是「打包方法」的**权威可照抄版本**。
> 沙箱会重置、会话会压缩、AI 会换，但只要仓库在，这份方法就丢不了。
> 最后更新：2026-11-14

## 0. 一句话

把网页签名控制台的所有功能，用**全新原生 SwiftUI 工程**重做一遍，通过 **GitHub Actions 的 macOS runner** 云编译出真机 `.ipa`，用用户个人证书签名后装到 iPad mini 5。**不是网页套壳**——套壳做不了 Apple ID 签名。

---

## 1. 为什么必须是原生 App（不是套壳）

用户原话：「App 是一个全新的，只不过只是页面一样，不能是网页套壳，不然他也只是个网页还是不能制作 Apple ID 签名」。

Apple ID 签名需要：iOS Keychain 存凭证、Anisette 设备数据、本地网络/VPN、与 Apple 私有接口通信、直接安装到主屏幕。WKWebView / PWA 壳一个都做不到。所以页面可以照着网页版长一样，但代码必须是纯 Swift 编译的原生工程。

---

## 2. 仓库与凭据

| 项 | 值 |
|---|---|
| 仓库 | `hjm98-rgb/signdock-native`（public，main，2026-11-14 创建） |
| 本地 | `/workspace/signdock-native` |
| 账号 | `hjm98-rgb`，free 计划，token scopes 含 `repo` `workflow` |
| Token | `/root/.config/signdock/gh_token`（600 权限）。读取：`TOK=$(cat /root/.config/signdock/gh_token)` |
| 推送 | `git -c http.extraHeader="AUTHORIZATION: bearer $TOK" push origin main` |

**铁律**：token 只在本地 600 权限文件里，绝不写进任何提交文件、绝不回显到对话、绝不硬编码。
公开仓库 Actions 免费额度足够跑几十次（free 账号私有仓库额度更少）。

用户主设备：**iPad mini 5**，有**个人 p12 证书 + mobileprovision + 证书密码**。
涉及 Apple ID 登录 / 2FA / 证书信任**必须暂停让用户手动处理**。

---

## 3. 需要的 GitHub Secrets（仓库 Settings → Secrets → Actions）

| Secret | 说明 |
|---|---|
| `P12_BASE64` | 个人开发证书 `.p12` 的 base64（`base64 -i cert.p12 | pbcopy`） |
| `P12_PASSWORD` | 证书密码 |
| `MOBILEPROVISION_BASE64` | 描述文件 base64 |
| `KEYCHAIN_PASSWORD` | CI 临时钥匙串密码，任意随机串 |
| `TEAM_ID` | Apple Developer Team ID |

没配证书 secrets 时，CI 产出**未签名 IPA**（仍可下载，再由用户用自己的证书签名），**不要假装签了名**。

---

## 4. 目录结构

```
signdock-native/
├── .github/workflows/build.yml   # 云编译流水线
├── .gitignore
├── project.yml                    # xcodegen 工程定义
├── ExportOptions.plist            # 导出签名 IPA 用
├── docs/CLOUD-BUILD.md            # 本手册
├── README.md
└── Signdock/                      # 纯 Swift 源码（原生，非套壳）
    ├── SigndockApp.swift          # @main
    ├── ContentView.swift          # 根导航 / 侧栏，9 个视图
    ├── Theme.swift                # 深色主题，对齐网页版
    ├── Models/                    # 数据模型 + 数据空间隔离存储
    │   └── SpaceStore.swift       # FileManager，Documents/spaces/<spaceId>/
    ├── Networking/
    │   └── APIClient.swift        # URLSession 对接 Flask 真实 API
    ├── Services/
    │   ├── AppleIDSigner.swift    # Apple ID 签名（Keychain/2FA/7天/3个/权限）
    │   └── SignService.swift      # 证书签名 + 三引擎
    ├── Views/                     # 9 个原生视图
    │   ├── DashboardView.swift
    │   ├── AppsView.swift
    │   ├── CertsView.swift
    │   ├── EnginesView.swift
    │   ├── SignTasksView.swift
    │   ├── OTAView.swift
    │   ├── BrowserView.swift      # 能操作（GitHub 限定）
    │   ├── MakerView.swift        # 制作应用 + 免责协议门禁
    │   └── AppleIDView.swift      # Apple ID 签名
    └── Info.plist
```

---

## 5. `project.yml`（xcodegen）

```yaml
name: Signdock
options:
  bundleIdPrefix: com.signdock
  deploymentTarget:
    iOS: "16.0"
  xcodeVersion: "16"
  createIntermediateGroups: true
targets:
  Signdock:
    type: application
    platform: iOS
    sources:
      - path: Signdock
    info:
      path: Signdock/Info.plist
      properties:
        CFBundleDisplayName: Signdock
        CFBundleName: Signdock
        CFBundleIdentifier: com.signdock.app
        CFBundleShortVersionString: "1.0"
        CFBundleVersion: "1"
        LSRequiresIPhoneOS: true
        UILaunchScreen: {}
        ITSAppUsesNonExemptEncryption: false
        UIRequiredDeviceCapabilities: [arm64]
        UISupportedInterfaceOrientations:
          - UIInterfaceOrientationPortrait
          - UIInterfaceOrientationLandscapeLeft
          - UIInterfaceOrientationLandscapeRight
        NSAppTransportSecurity:
          NSAllowsArbitraryLoads: false
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.signdock.app
        SWIFT_VERSION: "5.0"
        TARGETED_DEVICE_FAMILY: "1,2"
        CODE_SIGN_STYLE: Manual
        DEVELOPMENT_TEAM: ""
      configs:
        Release:
          SWIFT_OPTIMIZATION_LEVEL: "-O"
schemes:
  Signdock:
    build:
      targets:
        Signdock: all
    archive:
      config: Release
```

不引入 CocoaPods / SPM 外网依赖，减少云编译失败点。

---

## 6. `ExportOptions.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key><string>development</string>
  <key>signingStyle</key><string>manual</string>
  <key>teamID</key><string>$(TEAM_ID)</string>
  <key>provisioningProfiles</key>
  <dict><key>com.signdock.app</key><string>Signdock Development</string></dict>
  <key>stripSwiftSymbols</key><true/>
  <key>compileBitcode</key><false/>
</dict>
</plist>
```

`method` 按证书类型选 `development`（个人开发者）或 `ad-hoc`。`$(TEAM_ID)` 在 CI 里用 `PlistBuddy` 或 `sed` 从 secret 替换。

---

## 7. `.github/workflows/build.yml`

```yaml
name: Build Signdock IPA
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: macos-14
    timeout-minutes: 40
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: |
          ls /Applications | grep Xcode
          sudo xcode-select -s "$(ls -d /Applications/Xcode_*.app | head -1)"
          xcodebuild -version

      - name: Install xcodegen
        run: brew install xcodegen

      - name: Generate Xcode project
        run: xcodegen generate

      - name: Import signing certificate (optional)
        env:
          P12_BASE64: ${{ secrets.P12_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          MP_BASE64: ${{ secrets.MOBILEPROVISION_BASE64 }}
          KC_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
          TEAM_ID: ${{ secrets.TEAM_ID }}
        run: |
          if [ -z "$P12_BASE64" ] || [ -z "$MP_BASE64" ]; then
            echo "NO_CERT: 产出未签名 IPA"; echo "SIGN_MODE=unsigned" >> $GITHUB_ENV; exit 0
          fi
          echo "SIGN_MODE=signed" >> $GITHUB_ENV
          KC="$RUNNER_TEMP/build.keychain-db"
          security create-keychain -p "$KC_PASSWORD" "$KC"
          security set-keychain-settings -lut 21600 "$KC"
          security unlock-keychain -p "$KC_PASSWORD" "$KC"
          echo "$P12_BASE64" | base64 --decode > "$RUNNER_TEMP/cert.p12"
          security import "$RUNNER_TEMP/cert.p12" -k "$KC" -P "$P12_PASSWORD" \
            -T /usr/bin/codesign -A
          security set-key-partition-list -S apple-tool:,apple: -s -k "$KC_PASSWORD" "$KC"
          security list-keychains -d user -s "$KC" $(security list-keychains -d user | tr -d '"')
          mkdir -p "$HOME/Library/MobileDevice/Provisioning Profiles"
          echo "$MP_BASE64" | base64 --decode > "$HOME/Library/MobileDevice/Provisioning Profiles/signdock.mobileprovision"
          /usr/libexec/PlistBuddy -c "Set :teamID $TEAM_ID" ExportOptions.plist

      - name: Archive
        run: |
          COMMON="-project Signdock.xcodeproj -scheme Signdock -configuration Release \
                  -destination generic/platform=iOS -archivePath build/Signdock.xcarchive"
          if [ "$SIGN_MODE" = "signed" ]; then
            xcodebuild $COMMON CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM="$TEAM_ID" clean archive
          else
            xcodebuild $COMMON CODE_SIGNING_ALLOWED=NO clean archive
          fi

      - name: Export IPA
        run: |
          mkdir -p build/ipa
          xcodebuild -exportArchive -archivePath build/Signdock.xcarchive \
            -exportPath build/ipa -exportOptionsPlist ExportOptions.plist \
            -allowProvisioningUpdates || {
              echo "带签名导出失败，回退为未签名 IPA"
              rm -rf build/ipa && mkdir -p build/ipa/Payload
              cp -R build/Signdock.xcarchive/Products/Applications/Signdock.app build/ipa/Payload/
              (cd build/ipa && zip -qry Signdock-unsigned.ipa Payload)
            }
          ls -lh build/ipa/*.ipa

      - name: Verify IPA structure
        run: |
          IPA=$(ls build/ipa/*.ipa | head -1)
          unzip -l "$IPA" | grep -E 'Payload/.*\.app/Info.plist' || exit 1

      - uses: actions/upload-artifact@v4
        with:
          name: Signdock-IPA
          path: build/ipa/*.ipa
          if-no-files-found: error
```

---

## 8. 本地验证 / 手动触发

```bash
# 本地（Linux 无法编译 iOS，只做结构与文档校验）
cd /workspace/signdock-native
git add -A && git commit -m "cloud build: workflow + project + docs"
TOK=$(cat /root/.config/signdock/gh_token)
git -c http.extraHeader="AUTHORIZATION: bearer $TOK" push origin main

# 查看 Actions 结果
curl -s -H "Authorization: token $TOK" \
  https://api.github.com/repos/hjm98-rgb/signdock-native/actions/runs?per_page=3 \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print([(r['name'],r['status'],r['conclusion']) for r in d['workflow_runs']])"

# 手动触发
curl -s -X POST -H "Authorization: token $TOK" \
  https://api.github.com/repos/hjm98-rgb/signdock-native/actions/workflows/build.yml/dispatches \
  -d '{"ref":"main"}'
```

编译报错时：读 Actions 日志 → 改 `project.yml` / Swift 源码 → 重新 commit + push，循环到绿。

---

## 9. 九个原生视图的功能要求

| 视图 | 必须真实做到 |
|---|---|
| 总览 | 统计本数据空间的应用/证书/任务数，引擎状态 |
| 应用库 | `.fileImporter` 从文件 App 导入 `.ipa`/`.p12`/`.mobileprovision`；**不要设死 UTI 过滤**（iOS 上会把自定义扩展名变灰） |
| 证书 | 存 p12 + 密码到本空间，Keychain 保护，绝不上传/共享 |
| 签名引擎 | 三引擎可选：原版 zsign（默认）/ Feather / Anvil，切换持久化 |
| 签名任务 | 选 IPA + 证书 + 引擎 → 调真实 `/api/sign-now` → 显示真签名结果，失败真报错 |
| 无线安装 | manifest.plist + `itms-services://` + 二维码（证书签名模式） |
| 能操作 | 只允许 GitHub 相关链接的简易浏览器；下载的 `.ipa` 入本空间应用库；强白名单逐跳校验 |
| 制作应用 | 开源母版 + 资源替换重打包；**免责协议硬门禁**（不主动勾选就锁定，换空间重签） |
| Apple ID 签名 | 见下节 |

数据隔离：`FileManager`，每空间一个目录 `Documents/spaces/<spaceId>/`；切换空间重新校验协议门禁；服务端零留存。

网络层：`APIClient` 用 `URLSession` 对接现有 Flask 真实 API，baseURL 默认 `https://a1f475e54f1b1229a.app.workbuddy.host`，同源请求带 `sharecode`。

---

## 10. Apple ID 签名（真实边界，别做假按钮）

两种签名模式：**证书签名**（已有，走 p12 + OTA）+ **Apple ID 签名**（新增，直接装主屏幕）。

- 输入 Apple ID 登录；**登录与 2FA 必须由用户手动完成**，绝不自动代登录、绝不把密码发到服务端。Apple 登录要 2FA + Anisette + 私有接口，服务端代登录会被拦且等于索取用户密码。
- 登录后选应用库里的 IPA → 点签名 → 用 Apple ID 签出的证书签名 → **直接安装到主屏幕**（不依赖 itms-services OTA）。
- **7 天有效期**：免费 Apple ID 证书 7 天过期，界面明确告知「7 天后需重新签名」。
- **最多 3 个 App**：免费账号主屏幕同时最多 3 个自签 App，做成真实计数与拦截，超限提示先删旧的。
- **权限按钮**：Apple ID 签名页加「权限」按钮，点进去列出可申请 capabilities（App Groups / iCloud / Push / Associated Domains / Keychain Sharing / HealthKit / SiriKit…），可「全部申请」，但必须写明：
  - 免费个人 Apple ID 只开放极少 capabilities；
  - 多数需要付费开发者账号 + App ID 显式开启 + 描述文件重新生成；
  - **Apple 是否批准未知**，界面说明「需要什么条件」「哪些能用/不能用」，绝不假装申请成功。

真实实现参考（同账号现成开源工程，clone 复用别从零造私有协议）：
- `hjm98-rgb/sidestore-zh-hans`：完整 Swift 工程，真做了 Apple ID 签名（2FA / Anisette / 证书签发）
- `hjm98-rgb/localdevvpn-32fix`：纯 SwiftUI App 结构
- `hjm98-rgb/eversign`：签名相关
- 合规第三方：AltStore / SideStore 系、紫水晶启动器 Amethyst（用户认可的开源第三方启动器）

---

## 11. 反复踩过的坑（别再犯）

1. 别把 token 写进提交文件、别在对话回显。
2. 别做网页套壳 / 假按钮 / 假进度条 / 假成功 / 共享证书。
3. 别放开「能操作」到任意网站，只 GitHub（用户否决过 Gitee 扩容和「放开全部网站」）。
4. 大文件别下载完再二次上传回服务端（会 `ERR_CONNECTION_RESET`），用流式 + 一次性票据。
5. 协议弹层别嵌在带 `transform` 动画的容器里（iOS `fixed` 定位会错位），遮罩别盖住顶栏把用户困住。
6. 切换数据空间有竞态，要用作废世代或重新校验门禁。
7. 涉及登录 / 证书信任 / 2FA 一律暂停，交用户手动。

---

## 12. 验收清单

- [ ] `xcodegen generate` 成功生成 `.xcodeproj`
- [ ] GitHub Actions 编译绿，`Signdock-IPA` artifact 可下载
- [ ] `unzip -l *.ipa` 确认 `Payload/Signdock.app/Info.plist` + Mach-O 存在
- [ ] 9 个视图在模拟器/真机可打开，无崩溃
- [ ] 网页版全套回归脚本不回退（`tools/verify_*.js`）
- [ ] iPad mini 5 真机装一次（涉及登录/信任证书，需用户手动）
