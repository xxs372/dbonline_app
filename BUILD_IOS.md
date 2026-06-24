# DB Online iOS 构建指南

DB Online React Native 项目编译为 iOS App（.ipa 文件）的完整指南。

> **当前环境**: Windows（无 macOS），无法原生编译 iOS。下文提供所有可行的构建方案。

---

## 项目信息

| 项目 | 值 |
|---|---|
| 应用名称 | DB Online |
| Bundle ID | com.dbonline.app |
| 最低 iOS 版本 | 15.1 |
| 当前版本 | 1.0 (1) |
| 框架 | React Native 0.86.0 |
| 包管理器 | pnpm 9.15.9 |

---

## 方案一：GitHub Actions 构建无签名 IPA（推荐，无需 Apple 开发者账号）

仓库已内置 `ios-unsigned-ipa.yml` 工作流，可在 macOS GitHub Runner 上自动编译出无签名的 `.ipa` 文件。

### 步骤

1. **将代码推送到你的 GitHub 仓库**

   ```bash
   # 在本地准备好代码后
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/你的用户名/dbonline_app.git
   git push -u origin main
   ```

2. **触发构建**

   - **自动触发**: 推送到 `main` 分支时会自动触发构建
   - **手动触发**: 在 GitHub 仓库页面点击 `Actions` → `Build Unsigned iOS IPA` → `Run workflow`

3. **下载 IPA**

   构建完成后，在 Actions 运行页面底部找到 `Artifacts` 区域，下载 `DBOnline-unsigned-ios.zip`。
   解压后得到 `DBOnline-unsigned.ipa`。

### 安装到 iPhone

无签名 IPA 需要通过侧载方式安装：

- **AltStore** (推荐): [altstore.io](https://altstore.io) — 免费，需 Apple ID，每 7 天刷新
- **SideStore**: [sidestore.io](https://sidestore.io) — AltStore 的开源替代
- **TrollStore**: 需要 iOS 14.0–17.0 越狱兼容设备，安装后永不过期
- **Sideloadly**: [sideloadly.io](https://sideloadly.io) — Windows 工具

---

## 方案二：GitHub Actions 构建签名 IPA（需要 Apple Developer 账号）

仓库已内置 `ios-signed-ipa.yml` 工作流，使用你的 Apple 开发者证书签名，生成可直接分发或上架的 IPA。

### 前置条件

1. **Apple Developer Program** 账号 ($99/年)
2. 在 Apple Developer Center 创建 **Distribution Certificate** 和 **App ID** (`com.dbonline.app`)
3. 创建对应的 **Provisioning Profile** (App Store / Ad Hoc)

### 设置 GitHub Secrets

在 GitHub 仓库 → Settings → Secrets and variables → Actions 中添加以下密钥：

| Secret 名称 | 说明 |
|---|---|
| `IOS_BUILD_CERTIFICATE_BASE64` | 分发证书 (.p12) 的 Base64 编码 |
| `IOS_P12_PASSWORD` | p12 证书密码 |
| `IOS_BUILD_PROVISION_PROFILE_BASE64` | 描述文件 (.mobileprovision) 的 Base64 编码 |
| `IOS_KEYCHAIN_PASSWORD` | 临时 Keychain 密码（可以是任意随机字符串） |
| `IOS_TEAM_ID` | Apple Developer Team ID（可在 developer.apple.com 查看） |

### 导出证书和描述文件

```bash
# 1. 从钥匙串导出证书
security export -k login.keychain -t identities -f pkcs12 -o cert.p12

# 2. Base64 编码
base64 -i cert.p12 -o cert.base64

# 3. 描述文件同理
base64 -i profile.mobileprovision -o profile.base64
```

### 触发构建

1. 在 GitHub → Actions → `Build Signed iOS IPA` → `Run workflow`
2. 选择导出方式 (`app-store` 或 `ad-hoc`)
3. 点击运行，等待构建完成
4. 下载 Artifacts 中的 `DBOnline-signed-ios` → 获取 `.ipa` 文件

---

## 方案三：在 macOS 上本地编译（推荐开发者）

### 环境要求

- macOS 15 (Sequoia) 或更高
- Xcode 16+
- Node.js 24+
- pnpm 9.15.9
- Ruby 3.3.11 + CocoaPods 1.16.2

### 构建步骤

```bash
# 1. 克隆代码
git clone https://github.com/你的用户名/dbonline_app.git
cd dbonline_app

# 2. 安装 JS 依赖
pnpm install --frozen-lockfile --config.node-linker=hoisted --config.package-import-method=copy

# 3. 安装 CocoaPods 依赖
cd ios
bundle install
bundle exec pod install --clean-install
cd ..

# 4. 在 Xcode 中打开项目
open ios/DBOnline.xcworkspace

# 5. 在 Xcode 中选择目标设备为 "Any iOS Device"
# 6. Product → Archive → Distribute App
```

### 命令行 Xcode 构建

```bash
# 无签名构建 (用于侧载测试)
xcodebuild \
  -workspace ios/DBOnline.xcworkspace \
  -scheme DBOnline \
  -configuration Release \
  -sdk iphoneos \
  -derivedDataPath build/DerivedData \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGN_IDENTITY="" \
  build

# 打包为 IPA
mkdir -p build/Payload
cp -R "build/DerivedData/Build/Products/Release-iphoneos/DBOnline.app" build/Payload/
(cd build && zip -qry DBOnline-unsigned.ipa Payload)
```

---

## 方案四：迁移至 Expo + EAS Build（无需 macOS）

如果你没有 macOS 也不想用 GitHub Actions，可以将项目迁移到 Expo 并使用 EAS Build 的云服务编译 IPA。

### 迁移步骤

```bash
# 1. 安装 Expo CLI
npx expo install expo

# 2. 安装开发客户端
npx expo install expo-dev-client

# 3. 安装 EAS CLI
npm install -g eas-cli

# 4. 登录 Expo 账号
eas login

# 5. 初始化 EAS 配置
eas init

# 6. 构建 iOS
eas build --platform ios
```

EAS Build 会自动在云端 macOS 机器上编译 IPA 并返回下载链接。
需要 Expo 账号（免费）和 Apple Developer 账号（用于签名）。

---

## 常见问题

### Q: 无签名 IPA 能用吗？
A: 不能直接安装。需要通过 AltStore、SideStore、TrollStore 等侧载工具安装。
侧载的 IPA 在没有开发者账号的情况下每 7 天需要刷新一次。

### Q: 一定要 Apple Developer 账号吗？
A: 如果是侧载（AltStore 等），不需要付费账号，免费 Apple ID 即可。
如果要上架 App Store 或通过 TestFlight 分发，需要 $99/年的 Apple Developer Program。

### Q: 能在 Windows 上编译 iOS App 吗？
A: 不能原生编译。但可以通过以下方式间接获得 IPA：
   - 使用 GitHub Actions（本仓库已配置好）
   - 迁移到 Expo 后使用 EAS Build 云服务
   - 使用第三方 CI 服务（Codemagic、Bitrise 等）

### Q: 项目中的 Bundle ID 是什么？
A: `com.dbonline.app`。如果你要上传到 App Store 或被其他服务管理，
   需要在 `ios/DBOnline.xcodeproj/project.pbxproj` 中修改 `PRODUCT_BUNDLE_IDENTIFIER`。
