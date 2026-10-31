# DSH Remote · 群体服务版（A 模式）状态

快照：2026-11-01 · 分支 **`feat/group-service`** · 本分支**只包含群体服务功能**，
不含 `main` 的 classic/单体版（mTLS 盲中继、主机证书、APNs 本机推送、远程重启主机）。

## 当前版本

| 组件 | 版本 | 状态 |
|---|---|---|
| relay | **0.3.0**（镜像 `dsh-remote-relay:0.3.0-group`） | 只保留群体版：控制面 WSS + 注册 API + 部署方后台 + 边缘 TLS/鉴权/隧道 |
| host-plugin | **0.2.1** | 只保留群体版：注册/隧道/loopback 明文代理（HTTP+WS）/群体配对/设置 UI；推送已删除；**远程重启主机恢复为可选组件（默认关闭）** |
| protocol | **0.2.0** | 只保留群体版协议常量/QR v2/帧/签名校验 |
| App | **0.1.2（iOS build 8 / Android versionCode 8）** | 只有群体 token 模式；mTLS/推送/通知 UI 已删除；**App 锁触发/位置已修**，**远程重启主机 UI 已恢复**；待真机验收 |

## 验收状态

- 群体服务版曾于 **2026-11-01** 用 App 0.1.0 (build 6) + host-plugin 0.1.23 + relay 0.2.13
  完成真机验收（配对→立刻进入无 401→手机布局/会话/实时消息正常；nginx 前置链路重配对正常）。
- **本轮「剥离 classic」重构后**：App 0.1.1 (build 7) 已构建并安装到真机，**待用户验收**
  （见 [`dsh-remoteApp/docs/ACCEPTANCE.md`](dsh-remoteApp/docs/ACCEPTANCE.md)）；
  验收通过后再提交 App 改动。

## 本轮（2026-11-01）剥离内容

- **host-plugin**：删除 classic TLS/mTLS/证书/CSR/CA、classic 配对、APNs/Expo 推送、
  `session/event` 推送监听与 `/__dsh_remote/{audit,devices,push-token}` 等 classic 接口；
  **远程重启主机保留为群体版可选组件**（0.2.1 恢复：`/__dsh_remote/status|restart` 走群体隧道，
  cookie/Bearer 鉴权，默认关闭，需在主机「设置 → 远程访问 → 组件」开启）；配置只剩群体字段。
- **relay**：删除 classic 数据面 TLS 透传（443 盲转发）、classic HTTP 面（301/admin/push/devices）、
  relay push、tokens.json/主机 token CLI、classic 配置；`mode` 只接受 group，缺省即群体版；
  边缘默认开启（显式关闭才 warn）。
- **App**：删除 iOS/Android 原生 mTLS（CSR/identity/network/mTLS WebView）、classic 配对与证书存储、
  原生 WebView 分支、APNs/通知/应用内通知卡片、通知设置与证书诊断；保留群体配对/token 会话/
  主机列表/设备与安全/RN WebView + 手机壳/内嵌浏览器；**远程重启 UI 恢复**（Bearer/cookie 原生请求），
  **App 锁只在冷启动/打开主机连接时触发并移到设置页**。
- **server/**：删除 classic compose/env/nginx 模板与 host 证书脚本，只保留群体版
  `docker-compose.group.yml` + `.env.group.example` + group nginx 配置。
- **protocol / docs / scripts**：清理 classic 协议符号、classic POC/部署脚本与经典文档；
  根 README/STATUS/server README/包 README 全部改为群体版定位。

## 关键修复（本轮真机踩坑，详见 diagnostics）

- 配对后立刻 401、点重试又能进：relay 单请求隧道 + iOS keep-alive 复用连接，pairing 后的 challenge
  被透传到主机明文代理。修复：隧道化请求强制 `Connection: close`（Upgrade 除外）。
- 页面能开但「暂无会话/一直重连」：主机明文代理未代理 WebSocket Upgrade；已补 upgrade 代理。
- 跳内嵌浏览器/手机布局丢失：隧道 URL 带端口被误判外链；改为 hostname 匹配。
- 设备「从未连接」：群体版鉴权成功时更新 lastSeen。
- 其他：多 challenge 并存、重连不清设备表、旧设备失效提示、错误响应诊断。

完整记录：[`docs/diagnostics/12-group-service-issues.md`](docs/diagnostics/12-group-service-issues.md)。

## 部署入口

- relay 部署（证书/端口/nginx/后台/邀请码/排障）：[`server/README.md`](server/README.md)
- 主机插件：[`dsh-remoteService/packages/host-plugin/README.md`](dsh-remoteService/packages/host-plugin/README.md)
- App：[`dsh-remoteApp/README.md`](dsh-remoteApp/README.md)
- 设计/安全决策：[`docs/group-service-design.zh.md`](docs/group-service-design.zh.md)

## 已知限制

- Android 未构建/未真机（本机无 Android SDK）；仅静态检查。
- 服务端推送（APNs）未实现（设计上后续走服务端统一通道）；本分支 App 无任何本机推送能力。
- iOS 原生无法把 App 注入的会话 cookie 设为 HttpOnly（依赖 relay `Set-Cookie`，见设计文档）。
- 协议/配置仍会演进，仅限小范围邀请制使用。
