# DSH Remote App 真机验收手册

本手册用于在**有 Xcode/iPhone 或 Android SDK/设备**的机器上完成 App 的真机验收。
本仓库开发机没有这些环境，因此 App 目前只通过了 TS/协议/原生源码一致性检查；
真机验收必须按本手册执行并记录结果。

---

## 0.1.7（主机内设置入口调整 + iOS 底部边缘手势 defer）真机验收记录 — ✅ 2026-11-01 用户验收通过

> 来源：用户反馈
> ①「App 进入主机后右上角设置进入的是系统设置，正常应该是进入该链接（该主机）的设置；
>    在外面进入的才是系统设置」；
> ②「系统设置中最底部的『设备与安全』应该去掉，从每个主机里去看」；
> ③「左下角的挂件拖动：此前可以随意拖动，现在从屏幕最底边起手拖会直接把 App 切到后台」。
>
> 0.1.7 一并包含两批改动（0.1.6 build 12 只是①的中间构建，未交付验收）：
>
> - 进入主机后，顶栏 ⚙ → 打开**这台主机**的「设备与安全」（连接设置）；返回回到主机页面
>   （WebView 常驻，不重新加载）；App「系统设置」只从主机列表（外面）右上角 ⚙ 进入，
>   底部「设备与安全」卡片删除；主机列表卡片上的 ⚙ 仍打开该主机的「设备与安全」。
> - iOS：进入主机页面时 **defer 底部（Home 指示条）的「上滑回主屏」系统手势**。页面里的
>   可拖动悬浮挂件贴到屏幕最底边后，从底边起手拖动不再被系统手势抢走；离开主机页面恢复
>   系统默认。代价：主机页面里从最底边上滑回主屏需连划两次（第一次只高亮 Home 指示条）。
> - 该手势行为做成了 App 设置里的开关（「主机页面手势 → 挂件贴底拖动」，**默认开启**）：
>   想换回一次上滑回主屏的用户可以关掉（详见下方验收项）。
>
> **以下结果尚未由用户确认，不得勾选。**

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.7（buildNumber 14 / Android versionCode 14） |
| 构建 | iOS Release（手动签名），`expo prebuild -p ios --no-install` + `pod install` + `xcodebuild` |
| 构建结果 | `** BUILD SUCCEEDED **`（Xcode Release，手动签名；bundle 版本读到 0.1.7 / 14）；`devicectl` 安装并启动成功 |
| 安装 | 已覆盖安装并启动到用户验收用 iPhone（UDID/序列号不入库；设备侧显示 0.1.7 / 14） |
| 构建目录 | `/tmp/dsh-ios-group-gesture14`（本机临时目录，不入库） |
| 原生模块 | 新增 `DSHMTLSBottomEdgeGestures.swift`（iOS 底部边缘手势 defer）+ JS bridge `setBottomEdgeGestureDeferral`（Android no-op） |
| 静态检查 | `npx tsc --noEmit`、`npm run check:native`、`npm run check:shell` 均通过（59 项 shell 断言） |

### 本版本改动

1. **导航入口（纯 JS/TS）**：
   - `App.tsx`：新增 `devicesReturnRoute`（'home' | 'webview'）——「设备与安全」按进入来源返回；
     WebView 顶栏 ⚙ 改接 `handleOpenConnectionSettings`（→ devices，返回 webview）。
   - `WebViewScreen.tsx`：`onOpenSettings` → `onOpenConnectionSettings`，注释/无障碍文案改「连接设置」；
     新增底部手势 defer 的 effect（页面可见时开、离开时关）。
   - `SettingsScreen.tsx`：删除底部「设备与安全」卡片与 `connection` / `onOpenDevices` props；
     App 锁说明改为「在该主机卡片 ⚙（或进入主机后顶栏 ⚙）打开设备与安全，再开启面容开关」。
2. **App 设置里的手势开关（纯 JS/TS）**：
   - `src/lib/settings.ts`（新）：`bottomEdgeGestureDeferral` 偏好（SecureStore，默认开；
     关闭时写 `'0'`）。
   - `SettingsScreen.tsx`：新增「主机页面手势 → 挂件贴底拖动」开关（仅 iOS 展示），
     说明开启/关闭各自的取舍。
   - `WebViewScreen.tsx`：每次进入主机页面时读取该偏好再决定是否 defer；离开/锁屏恢复默认。
3. **iOS 底部边缘手势 defer（原生）**：
   - `DSHMTLSBottomEdgeGestures.swift`（新）：运行时替换 `UIViewController` 的
     `preferredScreenEdgesDeferringSystemGestures` 基类实现（公开 API，进程内生效）：
     开启返回 `.bottom`，关闭返回 `.none`（系统默认）；切换时通知系统刷新。
   - `DSHMTLSModule.swift`：新增 `AsyncFunction("setBottomEdgeGestureDeferral")`。
   - `DshMtlsModule.kt`：同名 no-op（Android 无等价接口），保持 JS↔原生导出一致。
   - `modules/dsh-mtls/src/index.ts`：新增 `setBottomEdgeGestureDeferral(enabled)` bridge。
   - `scripts/check-native-sources.mjs`：新增方法/文件/API 断言。
4. 版本号 0.1.7 / buildNumber 14 / versionCode 14 同步（`package.json`、`app.json`、
   `modules/dsh-mtls/{package.json,ios/DSHMTLS.podspec,android/build.gradle}`、`package-lock.json`）。

### 验收项（2026-11-01 用户实测确认通过）

- [x] 覆盖安装 0.1.7 (build 14) 后 App 正常启动；已有配对记录 / App 锁 / 单主机面容开关保持（无需重新扫码）。
- [x] **进入主机后**点顶栏 ⚙ → 打开这台主机的「设备与安全」（不是 App 系统设置）；左上返回回到主机页面（页面不重新加载、会话与滚动位置保留）。
- [x] 从**主机列表**（外面）右上角 ⚙ 进入 App「系统设置」：有版本 / App 锁 / 「主机页面手势 → 挂件贴底拖动」/ 清理缓存，底部**没有**「设备与安全」入口。
- [x] 主机卡片上的 ⚙ 仍打开该主机的「设备与安全」，返回回到主机列表。
- [x] **挂件拖动**：把悬浮挂件拖到屏幕最底边后，仍能按住它自由拖动，App 不会被切到后台；一次上滑只高亮 Home 指示条、再上滑一次才回主屏。
- [x] 「挂件贴底拖动」开关**默认开启**；**关闭**后重新进入主机，从最底边上滑恢复**一次**回主屏（挂件贴底时从底边起手拖可能被系统手势抢走）；再打开后恢复两次上滑 + 挂件贴底可拖（开关在下次进入主机时生效）。
- [x] 离开主机页面（回主机列表/设置页）后，底部上滑恢复为一次回主屏。
- [x] 「设备与安全」页功能不回归：主机信息 / 设备密钥 label / 「打开此连接时要求面容」开关 / 解除本机配对（解除后回到列表或扫码页）。
- [x] 其它功能不回归：重启主机 ⏻ 显示/重启/自动恢复、WS 断线自动重连、外链内嵌浏览器、清理缓存、App 锁冷启动。

> 用户 **2026-11-01** 实测确认通过：主机内 ⚙ → 本主机「设备与安全」、系统设置入口/无设备与安全、挂件贴底拖动、开关关闭后一次上滑均按预期。

### 回归命令（本机）

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 14）见本文件「构建与安装（iOS）」
```

---

## 0.1.5（WS 批量断开去抖 + 自动重连修复）真机验收记录 — ✅ 2026-11-01 用户验收通过

> 来源：用户实测（App 0.1.4 build 10 + host 0.2.3）反馈——主机保存「允许手机重启主机 DSH」
> 后 App 连接断开但**不自动重连**，显示「与主机的实时连接已断开，自动重连未恢复，请点击刷新重试」，
> 必须手动刷新。根因：host 保存时会断开 Upgrade/WebSocket，`mobileWebShell` 对每个 WS 各上报一次
> `dsh-ws-close`，同一瞬间多个 close 把 60s/2 次的自动恢复预算一次耗尽。0.1.5 在 App 侧做去抖，
> 即使 host 侧（0.2.4）行为未变也不会再被一波 close 打爆。
> **以下结果尚未由用户确认，不得勾选。**

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.5（buildNumber 11 / Android versionCode 11） |
| 构建 | iOS Release（手动签名），`expo prebuild -p ios --no-install` + `pod install` + `xcodebuild` |
| 构建结果 | `** BUILD SUCCEEDED **`（Xcode Release，手动签名；bundle 版本读到 0.1.5 / 11）；`devicectl` 安装并启动成功，进程存活 |
| 安装 | `devicectl` 覆盖安装并启动到用户验收用 iPhone（UDID/序列号不入库） |
| 构建目录 | `/tmp/dsh-ios-group-clean11`（本机临时目录，不入库） |
| 原生模块 | 与 0.1.4 相同（本版本无原生改动） |
| 静态检查 | `npx tsc --noEmit`、`npm run check:native`、`npm run check:shell` 均通过（另用状态机模拟验证了「一波 5 个 close 只恢复一次 / reload 后仍断才累加预算 / 2 次后报错 / ws-open 重置」） |
| entitlement | 构建产物无 `aps-environment`（App 仍不具备推送能力） |

### 本版本改动（WebSocket 自动恢复）

1. **同一波 close 去抖**：`dsh-ws-close` 处理增加两层去抖——已有排队中的恢复（`pending`）时忽略
   后续 close；2s 内的多个 close 只算一次恢复尝试。host 保存设置同时断多个 socket 时只触发一次恢复。
2. **预算按「reload 后仍失败」累加**：只有「自动 reload 完成后再次 close」才消耗 60s 窗口内的
   2 次预算（`reloadedSinceClose`）；首次 close 只尝试恢复、不计数。
3. **预算重置**：`dsh-ws-open`（实时连接真正恢复）或普通页面加载成功（首次进入 / 用户刷新 /
   清理缓存 / 重启恢复）时清零；自动恢复 reload 的加载完成不清零（用于判断是否真的连续失败）。
4. **进行中不再报错**：预算未耗尽时显示警告色提示「与主机的实时连接已断开，正在自动重连…」
   （带 spinner、不显示「重试」按钮）；只有预算确实耗尽（连续 2 次自动 reload 后仍 close）才显示
   原来的错误文案与「重试」按钮。
5. 自动恢复仍走现有 `reload()`（先确保 group session 有效，必要时续期 token）；恢复成功后由
   `dsh-ws-open` / 页面加载完成**立即刷新一次组件状态**，0.1.4 的 10s 组件轮询兜底保留。
6. 保留 0.1.2–0.1.4 全部能力：重启主机 ⏻（二次确认/202 轮询/恢复自动 reload）、App 锁
   （全局冷启动 + 单连接开关）、无推送。

### 待用户确认的验收项（未勾选）

- [x] 覆盖安装 0.1.5 (build 11) 后 App 正常启动；0.1.4 的配对记录与连接开关保持（无需重新扫码）。
- [x] **主机保存「允许手机重启主机 DSH」（组件开关）后**：App 短暂断线后**自动恢复/重新加载**，
      不出现「自动重连未恢复，请点击刷新重试」错误，无需手动刷新。
- [x] 主机保存组件开关后，App 顶栏 ⏻ 在保存后自动出现/消失（10s 内，或重连后立即刷新）。
- [x] 主机短暂重启/断网恢复后，App 能自动重新加载并恢复实时流；不弹错误。
- [x] 若主机确实长时间不可达：最多自动尝试 2 次后显示「自动重连未恢复，请点击刷新重试」+
      「重试」按钮；点「重试」可恢复。
- [x] 重启主机 ⏻ 功能不回归：组件开启时出现、二次确认、主机恢复后页面自动重载。
- [x] App 锁不回归：冷启动解锁；单连接「打开此连接时要求面容」按设置生效。
- [x] 前后台仍**不收到任何推送**；设置页没有通知/推送项。
- [x] 群体版既有功能不回归：配对（含 SAS）、会话/消息/WebSocket、token 续期、设备撤销提示、
      外链内嵌浏览器、清理缓存。

### 回归命令（本机）

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 11）见本文件「构建与安装（iOS）」
```

### 备注 / 未决

- host 侧（0.2.4）会把「保存组件」改成只断主 mux；App 本轮去抖确保即使 host 一次断多个 socket
  也不会再耗光恢复预算（客户端自保）。
- relay 0.3.0 不重新部署；Android 本机无 SDK，未构建（仅静态检查）。
- 「页面加载成功重置预算」对普通加载成立；自动恢复 reload 的加载完成保留预算，等 `dsh-ws-open`
  或下一次 close 判断是否真正恢复（避免主机一直断开时无限 reload）。

---

> 结论：**用户 2026-11-01 确认验收完毕**（主机保存组件后 App 自动恢复并刷新 ⏻，无需手动刷新/重试）。


## 0.1.4（组件开关动态刷新）真机验收记录 — 待用户验收

> 来源：用户反馈「单机版有的组件开关动态刷新没了」——Host 修改「允许手机重启主机 DSH」后，
> App 顶栏 ⏻ 不会动态出现/消失，需要退出主机再进。0.1.4 新增进入主机后每 **10s** 拉一次
> `/__dsh_remote/status`（仅在页面可见且非重启中时），保存后最多 10s 内自动更新 ⏻。
> **以下结果尚未由用户确认，不得勾选。**

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.4（buildNumber 10 / Android versionCode 10） |
| 构建 | iOS Release（手动签名）；构建目录 `/tmp/dsh-ios-group-clean10` |
| 安装 | 已覆盖安装并启动（UDID 不入库） |
| 前置 | host-plugin ≥ **0.2.2**（组件保存已修复）；relay 0.3.0 无需重部署 |
| entitlement | 无 `aps-environment`（仍无推送能力） |

### 待用户确认

- [ ] 主机开启「允许手机重启主机 DSH」并保存后，**不退出主机**，App 顶栏 ⏻ 在约 10s 内自动出现；
- [ ] 主机关闭该组件并保存后，⏻ 在约 10s 内自动消失（不会误点）；
- [ ] 重启功能本身仍正常（点 ⏻ → 确认 → 主机重启 → 自动恢复）；
- [ ] 回归：配对/进入/手机布局/会话/消息/外链正常；App 锁行为（全局冷启动 + 单连接可选）不变；无任何推送。

---

## 0.1.3（App 锁语义调整：全局只冷启动 + 单连接可选面容）真机验收记录 — 待用户验收

> 来源：用户对 0.1.2 的 App 锁语义提出调整——顶层 App 锁不应导致每次打开连接都要求面容；
> 改为「全局 App 锁只负责冷启动解锁」，并给每台连接增加独立的「打开此连接时要求面容」开关。
> **以下结果尚未由用户确认，不得勾选**；用户验收通过后再补充结论与日期。
>
> 前置：主机端需更新到 **host-plugin ≥ 0.2.2**（0.2.2 修复组件开关保存问题；
> 远程重启自 0.2.1 起提供）。App 侧不依赖该保存修复，但重启 ⏻ 需主机插件已更新且
> 在主机「设置 → 远程访问 → 组件」开启「允许手机重启主机 DSH」。relay 本轮不重新部署（0.3.0 不变）。

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.3（buildNumber 9 / Android versionCode 9） |
| 构建 | iOS Release（手动签名），`expo prebuild -p ios --no-install` + `pod install` + `xcodebuild` |
| 构建结果 | `** BUILD SUCCEEDED **`（Xcode Release，手动签名；bundle 版本读到 0.1.3 / 9）；`devicectl` 安装并启动成功，进程存活 |
| 安装 | `devicectl` 覆盖安装并启动到用户验收用 iPhone（UDID/序列号不入库） |
| 构建目录 | `/tmp/dsh-ios-group-clean9`（本机临时目录，不入库） |
| 原生模块 | 与 0.1.2 相同（含 `groupHttpFetch`：带 `__Host-dsh_session` cookie 的 HTTPS 请求）；本版本无原生改动 |
| 静态检查 | `npx tsc --noEmit`、`npm run check:native`、`npm run check:shell` 均通过 |
| entitlement | 构建产物无 `aps-environment`（App 仍不具备推送能力） |

### 本版本改动（App 锁语义）

1. **全局 App 锁 = 只负责冷启动解锁**（设置页「App 锁（冷启动解锁）」）：
   开启后仅在 App 冷启动首次进入时要求 Face ID / Touch ID；打开主机连接、同步信息、
   心跳、切换会话、回前台均不再要求识别（不再对所有连接生效）。
2. **单连接开关**：`StoredConnection` 新增可选 `requireBiometricOnOpen`（旧记录没有该字段
   = false，加载时只保留 true，迁移安全）。「设备与安全」页（DevicesScreen）为当前主机显示
   「打开此连接时要求面容」开关；全局 App 锁关闭时开关禁用并提示
   「需先在「设置」中开启 App 锁，此开关才会生效」。
3. `App.tsx`：打开连接（主页点开 / 配对完成进入）时只有
   **全局 App 锁开启 && 该连接 `requireBiometricOnOpen === true`** 才 `setLocked(true)`；
   解锁成功后不会被系统 active 回调重新上锁（仍不监听 AppState）。
4. 保留 0.1.2 的「重启主机 DSH」UI（顶栏 ⏻ / 二次确认 / 202 后轮询探活 / 恢复自动 reload）
   与无推送状态；原生 `groupHttpFetch` 与 group token 会话逻辑未改。

### 待用户确认的验收项（未勾选）

- [ ] 覆盖安装 0.1.3 (build 9) 后 App 正常启动；0.1.2 的配对记录可继续使用（无需重新扫码）。
- [ ] **全局 App 锁开启时**：App 冷启动首次进入要求 Face ID / Touch ID；解锁后进入主机列表。
- [ ] **打开任意主机（未单独开启开关）不再要求面容**（即使全局 App 锁开启）。
- [ ] 「设备与安全」页有「打开此连接时要求面容」开关（在当前主机信息下方）。
- [ ] 打开该主机的「打开此连接时要求面容」后：从主机列表打开这台主机要求一次面容；
      打开其它主机不要求（其它主机不受影响）。
- [ ] **全局 App 锁关闭时**：设备页该开关禁用并提示「需先在「设置」中开启 App 锁」；
      即使此前记录为开启，打开连接也不要求面容。
- [ ] 冷启动解锁一次后，进入主机后的同步信息 / 心跳 / 切换会话 / 回前台均**不再**弹面容。
- [ ] **重启 ⏻**：主机开启组件后顶栏出现 ⏻，点击可重启主机并在恢复后自动重载（0.1.2 行为不回归）。
- [ ] 前后台仍**不收到任何推送**；设置页没有通知/推送项。
- [ ] 群体版既有功能不回归：配对（含 SAS）、会话/消息/WebSocket、token 续期、设备撤销提示、
      外链内嵌浏览器、清理缓存。

### 回归命令（本机）

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 9）见本文件「构建与安装（iOS）」
```

### 备注 / 未决

- host 端「组件开关保存」bug 由 host-fix 在 host-plugin 0.2.2 修复（与 App 无关）；
  用户验收重启 ⏻ 前需先更新主机插件，否则组件即使打开也可能保存不上。
- relay 0.3.0 本轮不重新部署；status/restart 走原生 `groupHttpFetch` 的显式 cookie，不依赖 relay Bearer。
- Android 本机无 SDK，未构建（仅静态检查）。

---

## 0.1.2（App 锁触发时机/位置修复 + 恢复群体版「重启主机 DSH」）真机验收记录 — 待用户验收

> 来源：用户对 0.1.1 真机验收反馈两件事——
> A) 开启 App 锁后，进入 App 每次「同步信息 / 心跳」都会强制面容识别；App 锁开关应放到 App 设置页。
> B) 「重启主机 DSH」不属于单体版功能，群体版需要加回。
> **以下结果尚未由用户确认，不得勾选**；用户验收通过后再补充结论与日期。
>
> 前置：主机端需升级到 **host-plugin ≥ 0.2.1**（恢复 `/__dsh_remote/status` 的
> `restart.enabled` 与 `/__dsh_remote/restart`；`GET status` / `POST restart` 均按
> `__Host-dsh_session` cookie 或 Bearer token 鉴权）。relay 本轮**不重新部署**（0.3.0 不变）。

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.2（buildNumber 8 / Android versionCode 8） |
| 构建 | iOS Release（手动签名），`expo prebuild -p ios --no-install` + `pod install` + `xcodebuild` |
| 构建结果 | `** BUILD SUCCEEDED **`（Xcode Release，手动签名；bundle 版本读到 0.1.2 / 8） |
| 安装 | 已用 `devicectl` 覆盖安装并启动到用户验收用 iPhone（UDID/序列号不入库）；启动后进程存活 |
| 构建目录 | `/tmp/dsh-ios-group-clean8`（本机临时目录，不入库） |
| 原生模块 | `modules/dsh-mtls` 新增 `groupHttpFetch`：带 `__Host-dsh_session` cookie（+ Bearer）的 HTTPS 请求，供主机 status/restart 内部接口使用；不使用设备证书 / mTLS |
| 静态检查 | `npx tsc --noEmit`、`npm run check:native`、`npm run check:shell` 均通过 |
| entitlement | 构建产物无 `aps-environment`（App 仍不具备推送能力） |

### 本版本改动

1. **App 锁**：开关从「设备与安全」页移到 **App 设置页**（版本卡片之后）。
   锁屏只在①冷启动（开关开启时）与②打开主机连接（主页点开主机 / 配对成功进入 WebView）时触发；
   **删除**了 App.tsx 里两个 `AppState === 'active'` 监听（原先每次回前台/解锁回调都会重新上锁，
   是反复弹面容的根因）。进入主机后的同步信息、心跳、切会话、回前台都**不再**要求生物识别。
2. **重启主机 DSH**：进入主机后通过原生 `groupHttpFetch` 请求
   `GET /__dsh_remote/status`（带会话 cookie），仅当返回 `data.restart.enabled === true`
   时在顶栏显示 ⏻；点击 → 二次确认 → `POST /__dsh_remote/restart` → 202 后显示「重启中」
   并轮询 status 探活（每 1.5s、单次 3s 超时、最多 30s）→ 主机恢复后自动重新加载 WebView。
   失败按 401/405/501（`host-restart-disabled` / `host-restart-unavailable`）/502/503 给可读提示，
   文案不再出现「设备证书 / mTLS / 证书已失效」。
3. 版本线：App **0.1.2**、iOS **buildNumber 8**、Android **versionCode 8**
   （`app.json` / `package.json` / `modules/dsh-mtls/android/build.gradle` 同步）。

### 待用户确认的验收项（未勾选）

- [ ] 覆盖安装 0.1.2 (build 8) 后 App 正常启动；0.1.1 的群体版配对记录可继续使用，无需重新扫码。
- [ ] **App 锁开关在「App 设置」页**（版本卡片之后），「设备与安全」页不再有 App 锁开关。
- [ ] 开启 App 锁后：App 冷启动首次进入要求 Face ID/Touch ID；从主页点开主机连接要求一次 Face ID/Touch ID。
- [ ] **进入主机后不再反复弹面容**：同步信息 / 心跳 / 切换会话 / 切后台再回前台 / 解锁弹窗收起后，
      均不再出现面容识别；解锁成功后界面保持解锁状态。
- [ ] 关闭 App 锁后不再要求识别；开关状态重启 App 后保持。
- [ ] 主机「远程访问 → 组件」开关**关闭**时，进入主机顶栏**没有** ⏻（只有返回/设置/刷新）。
- [ ] 主机打开「允许手机重启主机 DSH」后进入主机，顶栏出现 ⏻。
- [ ] 点 ⏻ → 二次确认 → 主机 DSH 重启；App 显示「重启中」，主机恢复后页面**自动重新加载**并恢复实时连接。
- [ ] 组件未开启 / 主机不支持时点重启的提示可读（如「主机未允许手机重启 DSH…」），
      且不出现「设备证书 / mTLS」措辞。
- [ ] **前后台均仍不收到任何推送**；设置页没有通知/推送项。
- [ ] 群体版既有功能不回归：配对（含 SAS 确认）、会话列表/消息收发、WebSocket 实时流、
      切后台回前台 token 续期、设备撤销提示、外链在 App 内嵌浏览器打开、清理缓存。

### 回归命令（本机）

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 8）见本文件「构建与安装（iOS）」
```

### 备注 / 未决

- relay 边缘（0.3.0，本轮不重部署）对普通隧道流量**只校验 `__Host-dsh_session` cookie**；
  App 的 JS `fetch` 拿不到 WebView 的 cookie store，因此 status/restart 改走原生
  `groupHttpFetch`（显式 cookie + Bearer）。若未来 relay 支持 Bearer，可去掉 cookie 兼容。
- 若主机端尚未升级到 host-plugin ≥ 0.2.1：status 响应没有 `restart` 字段 → ⏻ 不显示（预期）。
- Android 本机无 SDK，未构建（与 0.1.1 相同，仅静态检查）。

---

## 0.1.1（剥离 classic / 只保留群体版）真机验收记录 — 待用户验收

> 来源：App 已删除全部 classic/单体 mTLS 模式与 APNs 推送 / 应用内通知 / 远程重启主机，
> 只保留群体版（v2 token 配对 + P-256 设备密钥 + 短时 token）。
> **以下结果尚未由用户确认，不得勾选**；用户验收通过后再补充结论与日期。
>
> 前置：服务端需先升级到 **relay 0.3.0 / host-plugin 0.2.0**（本分支只含群体版）；
> 旧 App 连接记录会被丢弃，需重新扫码配对。

| 项目 | 值 |
|---|---|
| App 版本 | 0.1.1（buildNumber 7 / Android versionCode 7） |
| 构建 | iOS Release（手动签名），`expo prebuild -p ios --no-install` + `pod install` + `xcodebuild` |
| 安装 | 覆盖安装到用户验收用 iPhone（`devicectl`；UDID/序列号不入库） |
| 构建目录 | `/tmp/dsh-ios-group-clean`（本机临时目录，不入库） |
| 原生模块 | `modules/dsh-mtls` 只含群体版：P-256 设备密钥 / 签名 / `__Host-` cookie / 缓存清理 |
| 静态检查 | `npx tsc --noEmit`、`npm run check:native`、`npm run check:shell` 均通过 |
| entitlement | 构建产物无 `aps-environment`（App 不再具备推送能力） |

### 待用户确认的验收项（未勾选）

- [ ] 覆盖安装 0.1.1 (build 7) 后 App 正常启动；0.1.0 的群体版配对记录可继续使用（升级迁移），
      classic 版遗留记录被丢弃。
- [ ] 扫码/粘贴 v2 token 二维码完成群体版配对（桌面确认 + SAS 一致）。
- [ ] 进入主机后 DSH 手机布局正常；会话列表 / 消息收发 / WebSocket 实时流正常。
- [ ] 切后台再回前台、长时间停留后仍能正常访问（token 续期）。
- [ ] 主机侧撤销设备后，重新进入提示「设备已在主机侧失效」并引导重新配对。
- [ ] **前后台均不再收到任何推送**；设置页没有通知/推送项。
- [ ] 主界面顶栏没有「重启主机」按钮（只有返回 / 设置 / 刷新）。
- [ ] 设置页只有版本 / 清理缓存 / 设备与安全，无 classic/mTLS/证书项。
- [ ] 「设备与安全」页显示 token 认证、主机 ID、端口、P-256 设备密钥 label、App 锁开关；
      无证书 / mTLS 诊断 / 证书导入项。
- [ ] 外链在 App 内嵌浏览器打开；解除本机配对后可重新扫码配对。

### 回归命令（本机）

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 7）见 dsh-remoteApp/README.md「构建与安装」
```

---

## 群体服务版（A 模式 / token 鉴权）真机验收记录 — 2026-11-01

> 分支 `feat/group-service`：relay 终止 TLS、主机出站隧道、设备 P-256 密钥 + 短时 token
> （无客户端证书）。**用户于 2026-11-01 确认通过**（原话要点：切回 nginx 前置链路后重新配对，
> 可正常进入界面）。以下标识全部脱敏。

| 项目 | 值 |
|---|---|
| App | 0.1.0（群体版起始版本线）buildNumber 6（iOS Release，真机安装） |
| host-plugin | 0.1.23 |
| relay | 0.2.13（验收镜像）；0.2.14 仅日志降噪，功能一致 |
| 设备 | iPhone 真机（iOS 真机系统，Secure Enclave P-256）；UDID/序列号不入库 |
| 网络 | 5G 蜂窝 / Wi-Fi 均验证；公网 relay + 部署方 nginx 前置链路 |

### 验收项

| 项目 | 结果 | 说明 |
|---|---|---|
| 扫码配对（v2 token 模式） | ✅ | relay `challenge issued` + `token issued`，主机设备表新增设备 |
| 配对后立刻进入 | ✅ | 无 401/404；token 一次签发，页面直接加载 |
| DSH 手机布局 | ✅ | 左侧抽屉覆盖式（不挤压正文）、点会话放大、返回正常 |
| 页面内导航 | ✅ | 不再跳到 App 内嵌浏览器（隧道地址带端口时也识别为隧道内） |
| WebSocket 实时通道 | ✅ | `/api/remote.mux` 经隧道升级成功，可收发流式消息 |
| 配对后「重试」可用 | ✅ | 修复前「重试才能进」的连接复用问题已消除 |
| 设备「最近连接」 | ✅ | 群体版鉴权成功即更新 `lastSeen`，管理台不再显示「从未连接」 |
| 停用 nginx 直连对照 | ✅ | 直连 edge 端口与 nginx SNI 前置两种形态均可用 |

### 本分支修复的问题（对应现象）

1. **配对后立刻 401、点重试又能进**：relay 边缘隧道只处理连接上第一个请求，iOS keep-alive
   复用连接导致 pairing 之后的 challenge 被透传到主机明文代理。修复：relay 对隧道化请求强制
   `Connection: close`（WebSocket Upgrade 除外）。
2. **页面能开但「暂无会话 / 一直重新连接中」**：主机 loopback 明文代理未处理 WebSocket 升级。
   修复：host-plugin 增加 upgrade 代理（同一套 token+设备校验后交给 `DshProxy.handleUpgrade`）。
3. **跳内嵌浏览器 / 手机布局丢失**：隧道地址带显式端口时被误判为外链。修复：隧道 URL 按
   hostname（忽略端口）匹配。
4. **设备失效后无限 401/重试**：relay 返回 `unknown_device` 时 App 给出「设备已在主机侧失效，
   请删除本机连接重新扫码配对」的明确提示，并只对刚配对窗口做一次短暂重试。
5. **challenge 并发/重复导致 token 401**：relay 允许同一设备多个未消费 challenge（互不覆盖）。
6. **配对后 redirect 的旧连接**：relay 在主机控制连接替换/短暂重连时不再清空设备表。

### 回归命令（本机）

```bash
cd dsh-remoteService && npm test            # 195/195
cd dsh-remoteService && npm run validate:artifacts
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
# iOS Release 构建（buildNumber 6）
xcodebuild -workspace DSHRemote.xcworkspace -scheme DSHRemote -configuration Release \
  -destination 'generic/platform=iOS' build
```

---

---

## 构建与安装（iOS）

```bash
cd dsh-remoteApp
npm install
npx expo prebuild -p ios --no-install
cd ios && CP_HOME_DIR="$PWD/.pods-home" CP_CACHE_DIR="$PWD/.pods-cache" pod install
cd ..
xcodebuild -workspace ios/DSHRemote.xcworkspace -scheme DSHRemote -configuration Release \
  -destination 'generic/platform=iOS' -derivedDataPath /tmp/dsh-ios-build \
  CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM=<TEAM_ID> \
  PROVISIONING_PROFILE_SPECIFIER=<PROFILE_UUID> \
  CODE_SIGN_IDENTITY='Apple Development: <NAME> (<CERT_ID>)' build
xcrun devicectl device install app --device <IPHONE_UDID> /tmp/dsh-ios-build/Build/Products/Release-iphoneos/DSHRemote.app
xcrun devicectl device process launch --device <IPHONE_UDID> <BUNDLE_ID>
```

> 签名材料、Team ID、描述文件 UUID、UDID、bundle id 走本机参数/环境变量，不入库。

Android（当前开发机无 SDK）：需要 Android SDK + JDK 17+，`npx expo prebuild -p android` 后
`npx expo run:android --variant release`；同一份 JS/原生 group 模块，验收项同 iOS。

## 回归命令

```bash
cd dsh-remoteApp && npx tsc --noEmit
cd dsh-remoteApp && npm run check:native
cd dsh-remoteApp && npm run check:shell
cd dsh-remoteApp && npx expo config --type public
```

## 已知限制

- Android 未构建/未真机（仅静态检查）。
- 群体版暂无本机推送能力（App 已移除 APNs/通知框架与代码；设计上后续走服务端统一推送）。
- iOS 原生无法程序化设置 HttpOnly cookie（依赖 relay `Set-Cookie`；App 注入的会话 cookie 为 Secure/SameSite=Strict/host-only）。
- 旧 classic/单体版连接记录会在升级后直接丢弃（本分支只支持群体版）。

## 证据与脱敏

截图/日志放 `dsh-remoteApp/docs/evidence/`（可选）；提交前按根目录 `AGENTS.md`「三」脱敏
（域名、hostId、设备 ID、UDID、序列号、token、cookie、二维码一律替换为占位符）。
