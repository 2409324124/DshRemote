# 12 - Windows setup and Android real-device E2E smoke test

验收日期：2026-11-02（Asia/Shanghai）。源码基线：`ae60b31`，加本轮 Android Kotlin 编译修复。

## 1. 验收范围与结论

本轮在一台已有 Windows 11 的开发机上补齐 Android 工具链，完成调试 APK 构建、USB 安装、
小米真机扫码配对、原生 mTLS WebView、公网中继访问及聊天烟雾测试。
不是全新 Windows 镜像验证，不是正式安装器验收，也不是 Android Release/商店发布验收。
Windows Desktop 使用已放置的官方桌面包，启动并加载本地远程访问插件；未重新执行安装向导。

| 项目 | 实际环境 |
|---|---|
| Windows | Windows 11 x64，版本 10.0.26100 |
| DSH Desktop | 文件版本 0.2.1-rc.5 |
| Node / npm | v24.14.0 / 11.9.0 |
| JDK | Eclipse Temurin 17.0.20.1 |
| App / host-plugin | 0.1.14，versionCode 17 / 0.1.16 |
| Expo / React Native | 58.0.14 / 0.88.0 |
| Gradle | 9.4.1，使用生成工程自带 wrapper |
| Android SDK | Platform `platforms;android-37.0`，Build Tools 37.0.0 |
| NDK / CMake | 27.1.12297006 / 3.22.1 |
| 真机 | Xiaomi 17，型号 25113PN0EC，Android 16 |
| APK | `com.amesucre.dshremote`，minSdk 24，targetSdk 36 |
| 公网链路 | WSS 控制面 8443；手机 HTTPS/mTLS 数据面 1443；未改动已有 443 服务 |

## 2. Windows 主机与工具链准备

先按 [部署指南](../../dsh-remoteService/docs/DEPLOY.md) 配置 relay、DNS、主机 TLS 证书及 host token。
本文统一用 `relay.example.com` / `h_example` 占位，不包含本机真实域名、密钥、配对 URI 或 token。

### 2.1 先对齐版本：Expo SDK 不等于 Android SDK

本轮没有升级仓库的 Expo / React Native 来适配电脑，而是补齐电脑缺少的 Android 构建组件。
最初本机已有 Android Platform 35 / Build Tools 35.0.0，但这不足以构建当前依赖生成的工程。

| 层级 | 仓库或依赖实际要求 | 本机最初情况 / 最终处理 |
|---|---|---|
| Expo SDK | `package.json` 声明 `~58.0.14`，实际安装 58.0.14 | 保持仓库版本；58 不是 Android API 编号 |
| React Native | 声明及实际安装均为 0.88.0 | 保持仓库版本，不降级换工具链 |
| Android compileSdk | 37 | 最初只有 Platform 35；最终补装 `platforms;android-37.0` |
| Android targetSdk / minSdk | 36 / 24 | 与 compileSdk 37 是不同字段，不是版本冲突 |
| Build Tools | 37.0.0 | 最初只有 35.0.0；最终补装 37.0.0 |
| NDK / CMake / Gradle | 27.1.12297006 / 3.22.1 / wrapper 9.4.1 | 按生成工程补齐，不能用“已安装 Android SDK”代替逐项核对 |

Android 版本来源可在安装依赖后核对：
`node_modules/react-native/gradle/libs.versions.toml` 中本轮为 `compileSdk=37`、`targetSdk=36`、
`minSdk=24`、`buildTools=37.0.0`、`ndkVersion=27.1.12297006`；Expo autolinking 的 Gradle 插件
读取版本目录并设置根工程属性，生成的 `android/app/build.gradle` 再消费这些属性。
因此不能只看 `app.json` 中显式设置的 `minSdkVersion=24`，就认为安装任意 SDK 版本都够用。

`npm run check:android-env` 目前是 Bash 粗检：主要检查 JDK/SDK 目录/adb 是否存在，
**不校验 compileSdk、Build Tools、NDK、CMake 是否满足当前工程**，也不是 Windows PowerShell 完整预检。
它显示“环境就绪”不等于已通过构建。本 PR 只记录该限制，不扩改环境检查脚本。

### 2.2 首次构建实际卡过的位置

| 卡点 / 表现 | 本轮确认的原因 | 处理方式 |
|---|---|---|
| 配好 `ANDROID_HOME` 后仍需安装 SDK | 本机 35 与生成工程所需 37 / Build Tools 37.0.0 不一致 | 按版本表补齐，不修改依赖版本来绕过 |
| 请求 `platforms;android-37` 找不到包 | 本轮 SDK Manager 的包名实际是 `platforms;android-37.0` | 使用下面列出的完整 package ID；API 数字和安装包名不要混写 |
| Gradle 很久没有任务进展、CPU 很低 | 继承了不可用代理；去掉代理后，直连 `plugins.gradle.org` 的依赖请求仍失败 | 先确认下载连通性，再显式配置 Gradle JVM HTTP/HTTPS 代理；不是 Kotlin / RN 版本不兼容 |
| Gradle wrapper 下载停在 0 字节 | `gradle-9.4.1-bin.zip` 的下载没有取得有效进展 | 使用可用网络/代理；手动下载时先校验完整性，再放到该 wrapper 的缓存目录，不换成其它 Gradle 版本 |
| 自动安装 NDK 看起来卡住 | NDK 下载体积大，受下载链路影响；不是已经开始编译 C++ | 预先安装并确认 27.1.12297006 完整；本轮用官方归档校验后解压补齐，不把 `.partial` 目录当作已安装 |
| 下载结束后仍被 Kotlin / Ninja 阻塞 | 透明背景属性赋值编译错误、Windows 对象文件路径过长 | 按第 3 节修复 setter，并在生成工程中设置 `CMAKE_OBJECT_PATH_MAX=240` |

SDK Manager 代理参数与 Gradle JVM 的代理参数不是同一套；仅设置 PowerShell 的代理环境变量
不代表两者都能下载。首次构建必须有依赖缓存或可用网络；第 3 节的 `--offline` 只适合依赖齐全后的复编。

### 2.3 安装与配置步骤

1. 安装 Windows 版 Node 24、JDK 17、Android SDK Command-line Tools / Platform Tools。
   本轮分别放在 `D:\dev\jdk-17` 和 `D:\Android\Sdk`，仓库使用短路径 `D:\dev\DshRemote`。
2. 安装、启动 DSH Windows Desktop。若使用本地插件，先完全退出 Desktop，在服务目录安装依赖，
   再执行下列命令。`dsh` CLI 必须先可用；安装来源及基础配置沿用部署指南。

```powershell
$Repo = 'D:\dev\DshRemote'
Set-Location "$Repo\dsh-remoteService"
npm ci
dsh plugin --profile desktop add "$Repo\dsh-remoteService\packages\host-plugin"
```

启动 Desktop 后，在远程访问设置中填写 relay URL、域名、hostId、host token、主机证书及私钥路径。
本轮直接使用 1443 数据端口时，还需要配置 `publicPort=1443`，并在云防火墙放行该端口。
主机证书必须覆盖 `<hostId>.<relayDomain>`；不能关闭 TLS 校验代替正确配置。

同一个 `DSH_HOME` / hostId 只运行一个主机实例。不要在 Desktop 运行时另开 `dsh web`。
本轮曾因两个实例反复替换中继控制连接而导致配对网络失败；停止经确认的旧 CLI 后恢复。
不要使用“杀掉全部 Node/Electron”的方式排查，避免影响其它应用。

```powershell
$env:JAVA_HOME = 'D:\dev\jdk-17'
$env:ANDROID_HOME = 'D:\Android\Sdk'
$env:ANDROID_SDK_ROOT = $env:ANDROID_HOME
$env:Path = "$env:JAVA_HOME\bin;$env:ANDROID_HOME\platform-tools;" + $env:Path
$SdkManager = "$env:ANDROID_HOME\cmdline-tools\latest\bin\sdkmanager.bat"

# 按交互提示阅读并接受 SDK license。
& $SdkManager --sdk_root=$env:ANDROID_HOME --licenses
& $SdkManager --sdk_root=$env:ANDROID_HOME 'platform-tools' 'platforms;android-37.0' 'build-tools;37.0.0' 'ndk;27.1.12297006' 'cmake;3.22.1'
```

注意该 SDK 的安装名是 `platforms;android-37.0`，不是 `platforms;android-37`。
下载受网络限制时，可为 SDK Manager 显式添加
`--proxy=http --proxy_host=127.0.0.1 --proxy_port=10808`；无代理环境不要照抄该参数。

## 3. 构建 Android 调试 APK

```powershell
Set-Location "$Repo\dsh-remoteApp"
npm ci
# 第一次生成工程时执行；不要对已有本地设置的工程随意使用 --clean。
& .\node_modules\.bin\expo.cmd prebuild --platform android
```

### 本轮必需的编译修复

`modules/dsh-mtls/android/src/main/java/com/dsh/remote/mtls/DshMtlsWebView.kt` 将
`backgroundColor = 0x00000000` 改为 `setBackgroundColor(0x00000000)`，避免 Kotlin 对继承属性的赋值编译错误。
透明背景语义不变，本轮不调整 App 布局。

Windows 上 RN/CMake 生成的对象文件路径过长会使 Ninja 构建失败。本轮在**生成的、未跟踪的**
`android/app/build.gradle` 的现有 `defaultConfig` 内加入：

```groovy
externalNativeBuild {
    cmake {
        arguments "-DCMAKE_OBJECT_PATH_MAX=240"
    }
}
```

不要另建第二个 `defaultConfig`。`128` 在本机不足以容纳固定目录前缀，最终使用 `240` 构建通过。
`android/` 不提交；重新 prebuild 后检查该设置。仍需保持仓库和 SDK 路径尽可能短。

```powershell
Set-Location "$Repo\dsh-remoteApp\android"

# 先清理当前构建进程继承的代理变量，不修改系统代理设置。
Remove-Item Env:HTTP_PROXY,Env:HTTPS_PROXY,Env:ALL_PROXY,Env:http_proxy,Env:https_proxy,Env:all_proxy -ErrorAction SilentlyContinue

# 本机首次下载依赖时使用的可选代理；网络直连正常时改为 @()。
$ProxyArgs = @('-Dhttp.proxyHost=127.0.0.1', '-Dhttp.proxyPort=10808', '-Dhttps.proxyHost=127.0.0.1', '-Dhttps.proxyPort=10808')
& .\gradlew.bat assembleDebug --no-daemon --console=plain @ProxyArgs
if ($LASTEXITCODE -ne 0) { throw 'Android build failed' }
```

输出：`android/app/build/outputs/apk/debug/app-debug.apk`。本轮离线增量复编日志为：

```text
BUILD SUCCESSFUL in 54s
339 actionable tasks: 22 executed, 317 up-to-date
```

这是已有缓存上的成功轮次，退出码为 0，不是干净环境首次构建耗时。完成依赖下载后可用
`./gradlew.bat assembleDebug --offline --no-daemon --console=plain` 做增量复编。
没有网络、也没有完整缓存时，不能用 `--offline` 完成首次构建。

交付 APK 为 180142683 字节，SHA-256：

```text
BB1C5236CADA33860427987B456FC10C5F08AFE45C9905E679C16BD849B31AF8
```

该摘要对应已经安装并验收的 APK，本次复编后再次核对摘要一致。APK 为 Android debug 签名，
已验证 v2 签名及 DEX 中的原生 mTLS 模块，不作为 Release 包发布，也不上传 APK 到源码仓库。

## 4. USB 安装、Metro 与真机验收

1. 手机开启开发者选项、USB 调试，手动允许本电脑的调试授权。
   小米还需开启“USB 安装”，并由用户完成手机安装确认；`INSTALL_FAILED_USER_RESTRICTED`
   是手机拒绝安装，不代表 APK 编译失败。不要绕过手机安全确认。
2. 用 `adb devices -l` 确认目标处于 `device`，下列 `<SERIAL>` 必须替换为该手机序列号。

```powershell
$Adb = "$env:ANDROID_HOME\platform-tools\adb.exe"
$Serial = '<SERIAL>'
& $Adb devices -l
& $Adb -s $Serial install -r "$Repo\dsh-remoteApp\android\app\build\outputs\apk\debug\app-debug.apk"
& $Adb -s $Serial reverse tcp:8081 tcp:8081
& $Adb -s $Serial shell am start -n com.amesucre.dshremote/.MainActivity
```

另开一个终端启动 Metro，并在调试期间保持运行：

```powershell
Set-Location 'D:\dev\DshRemote\dsh-remoteApp'
& .\node_modules\.bin\expo.cmd start --port 8081 --offline
```

本轮 APK **未内嵌 JS**，通过 USB reverse 从 Metro 加载；不能脱离 Metro 宣称独立运行通过。
Expo `--offline` 只限制开发服务器的外部请求，不会阻断 App 访问公网 relay。
本轮不要同时传 `--offline` 和 `--localhost`。

3. 只在 Desktop 生成新配对 QR，手机扫描、核对主机指纹/SAS，按配置完成桌面确认。
   不使用 POC 占位 CSR 或电脑生成的证书冒充 AndroidKeyStore 真机路径。
4. 配对后检查 App 的原生 mTLS WebView 已加载真正的 DSH 页面，地址为
   `https://<hostId>.relay.example.com:1443/?dsh_mobile=1`，无证书错误。
5. 在手机发送“回复 ok 即可”，确认收到 `ok`，并在 Windows Desktop 看到同一会话。
   不需要执行远程重启，也不应重新扫码来验证一个纯桌面显示问题。

### 已观测到的证据

| 验收项 | 证据与结论 |
|---|---|
| 编译 | `assembleDebug` 返回 `BUILD SUCCESSFUL`，APK 摘要见上 |
| 安装 | USB 安装返回 `Performing Streamed Install` / `Success`；只读包查询确认 0.1.14 / code 17 |
| 真机配对 | 17:15:17 主机审计 `pairing=paired`、`pair-request=paired`；用户确认“这次配对成功了” |
| mTLS / 公网页面 | 17:16:10 审计 `ws-upgrade=ok`；手机截图显示已连接、原生 mTLS 身份与完整 DSH 页面 |
| 聊天 | 用户确认手机烟雾测试通过；桌面实际截图显示同一“确认回复 ok”会话及 `ok` 回复 |
| 私钥边界 | APK 包含 AndroidKeyStore CSR/证书处理模块，使用原生身份；未导出设备私钥、未做硬件 TEE/StrongBox attestation |

脱敏审计摘录（UTC+08:00，不包含设备 ID、资源、密钥或哈希）：

```text
2026-11-02 17:15:17  pairing       paired
2026-11-02 17:15:17  pair-request  paired
2026-11-02 17:16:10  ws-upgrade    ok
```

原始构建日志、审计、手机截图及 APK 在本机保留，不把含个人部署地址/设备身份的截图、
生产配置、证书或聊天数据库提交到 PR。原生 mTLS 成功页截图的 SHA-256 为：

```text
239B0D3470D88E01B8277890401FC737FA8854B2C15F5D7AAFB6866578D9D3AD
```

## 5. 回归与未验收项

移出配对弹窗后续补丁后，2026-11-02 在本 PR 代码范围上重新验证：

| 命令 | 本轮结果 |
|---|---|
| `cd dsh-remoteService; npm test` | 163 项：161 passed、2 skipped、0 failed；两项跳过需 POSIX shell |
| `cd dsh-remoteApp; npm run typecheck` | 通过 |
| `npm run check:native` | 通过，0 warnings；静态桥检查不能替代原生编译 |
| `npm run check:shell` | 跳过：脚本未识别本机 Windows Chrome 路径；不计为浏览器行为验收通过 |
| Android `assembleDebug --offline --no-daemon --console=plain` | 通过：`BUILD SUCCESSFUL in 54s`，339 项任务中 22 项执行、317 项已是最新，退出码 0 |

本次复编后再次确认 APK 的 SHA-256 与第 3 节记录一致。构建验收与第 4 节的真实手机
配对、页面加载、聊天验收分别记录；本轮收口没有重新安装 APK 或扩展远程重启验收范围。

已存在的 Windows 证书工具兼容修复随基线保留，参见
[EC CSR / OpenSSL 排障](08-host-ec-csr-windows.md)。未将主机重启单元测试通过等同于本轮 Android
远程重启真机通过。本轮不验收重启、FCM、iOS、新增权限管理或发布签名。

### 已知问题，独立后续任务

- `FOLLOWUP-PAIRING-UI`：PC 弹窗忽略 `paired` 终态，自动生成下一张 QR，可能在手机已连接后仍显示“等待配对”。
  审计显示成功后约 5 毫秒即创建新会话，属于桌面状态展示问题；现有设备授权及手机聊天已验证可用。
  已编写的修复/测试保存为独立补丁，不属于本 PR。原始 QR 不得作为公开 issue 附件。
- `FOLLOWUP-DEVICE-COUNT`：侧栏“中继在线 · 3 台设备”统计未撤销的配对记录，不是当前在线数量，
  也未排除过期记录。本轮有两条电脑测试身份和一条真实 Android 身份；不在本 PR 调整文案或增加 presence。
- 本轮不修改上述前端行为，不为了让截图更好看而撤销/删除现有设备，也不混入 npm 安装造成的无关锁文件变化。
