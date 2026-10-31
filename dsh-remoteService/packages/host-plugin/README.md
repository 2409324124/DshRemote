# dsh-remote-service（主机插件 · 群体服务版）

> 许可证：**AGPL-3.0-or-later**（完整文本见根目录 [`LICENSE`](../../../LICENSE)）。
>
> 版本 **0.2.2**：本插件只保留群体服务版（A 模式）主机能力（classic/单体版的 mTLS、主机证书、
> 自托管推送已剥离）；**「重启主机 DSH」作为可关闭的「组件」在群体版恢复**（默认关闭）。
> 0.2.2 修复「组件开关保存失败」：历史配置缺 `groupServiceUrl` 时自动从 `group.json` 回填，
> 且纯状态字段（如 `restartHostEnabled`）保存不再重建隧道。
> 旧版配置字段会被忽略，升级后需要重新注册一次。

运行在 **DSH 主机** 上的远程访问插件，提供：

- 向部署方 relay 的**自助注册**（邀请码）与注册信息刷新；
- 主动外连 relay 控制面的**出站隧道**（主机不开放任何入站端口、不持有 TLS 证书）；
- 本机 **loopback 明文 HTTP/WS 代理**：逐请求校验 relay 签名 token + 设备在册后反代到
  `127.0.0.1` 的 DSH Web；
- **群体版设备配对**（二维码 v2 + 一次性密钥 + 桌面确认 + P-256 设备公钥登记）；
- **「重启主机 DSH」组件**（可关闭，默认关闭）：手机经隧道 `POST /__dsh_remote/restart`
  让主机调度 detached helper 重启整个 DSH 客户端（桌面端整组重启 / Node CLI 原样拉起）；
- 桌面 DSH 设置页 / 侧栏入口（注册、状态、配对、设备撤销、审计、组件开关）；
- 本机管理 API（仅本机已认证浏览器可访问）。

relay 的部署（含 TLS / 注册 API / 边缘配置）见 [`../../../server/README.md`](../../../server/README.md)。

---

## 1. 安装到 DSH

插件通过 DSH profile 的本地 link / 包安装：

```bash
# 方式 A：在 DSH 桌面端插件管理里添加本地目录
#   仓库路径：dsh-remoteService/packages/host-plugin

# 方式 B：命令行（路径按实际仓库位置）
dsh plugin --profile desktop add link:/path/to/dshRemote/dsh-remoteService/packages/host-plugin
```

安装后在 DSH 中启用插件，重启 DSH 桌面端/Web 实例使 Host 半与 Client 半加载。

> Host 半是 `index.js`，Client 半是 `client.js`（侧栏入口 + 设置页）。协议见 `packages/protocol`。

---

## 2. 群体服务版（A 模式）工作原理

```text
手机 ──TLS──> relay 边缘（终止手机 TLS、校验 __Host- 短时 token）
                        │
                        └─ 出站隧道（主机主动连 relay 控制面）
                                │
                        主机 loopback 明文代理 127.0.0.1:<groupPlainPort>
                                │  校验：Host == <hostId>.<relayDomain>
                                │        token 用 relay 公钥离线验签、未过期、hostId 匹配
                                │        token 里的 deviceId 仍是本机有效设备
                                ▼
                        DshProxy ──> 127.0.0.1:<DSH 端口>（注入本机 cookie / deny list / WS）
```

- 主机**不终止 TLS**、不持有证书、没有入站端口；relay 只把明文流转发进隧道。
- 设备身份只来自「配对登记的设备公钥 + relay 签名的短时 token」；不信任来源 IP、
  `X-Forwarded-*`、`Host`、`Origin`、自定义头或查询参数。
- 主机每次重新拉起时会先调用 relay `POST /__group/refresh` 同步最新
  `mobilePort` / `controlUrl`（不消耗邀请码），再重建隧道。
- 设备配对/撤销后自动把设备表（deviceId + P-256 公钥）全量 `device.sync` 给 relay。

---

## 3. 配置

DSH 设置 → 远程访问。配置落盘在 `~/.dsh/remote-access/config.json`（0600）：

| 字段 | 说明 |
|---|---|
| `enabled` | 启用远程访问；关闭后隧道断开、loopback 代理停止 |
| `hostName` | 主机友好名称（手机上显示，注册时同步给服务端） |
| `requireApproval` | 新设备配对需要本机确认（默认开启） |
| `groupServiceUrl` | 部署方 relay 的**注册 API** 地址，例如 `https://relay.example.com:19443`（POC 允许 `http://`，需 `groupInsecure`）。隧道 WSS 地址由注册响应 `controlUrl` 下发 |
| `groupInviteCode` | 一次性邀请码；注册成功后自动清空 |
| `groupPlainPort` | loopback 明文代理端口（默认 `0` = 随机；只被本机隧道客户端连接） |
| `groupInsecure` | 仅 POC：允许 http/ws 与自签证书（生产必须 https/wss） |
| `restartHostEnabled` | 「组件」：允许手机重启主机 DSH（**默认关闭**；关闭时 App 隐藏 ⏻、重启接口 501） |
| `denyPrefixes` | 反代 deny list（`/connect`、`/oauth`、`/open-in-app`、`/api/remote-access`、`/__dsh_remote`） |
| `maxRequestBytes` | 远程代理请求体上限（默认 256 MiB） |

注册结果落盘在 `~/.dsh/remote-access/group.json`（0600）：`hostId` / `hostToken` /
`relayDomain` / `mobilePort` / relay token 公钥 / `serviceUrl` / `controlUrl` / `hostName`。
**不要**把 `group.json`、邀请码或配对二维码提交到仓库 / 截图 / 日志里。

### 从单体版（≤0.1.x）升级

classic 配置字段（`relayUrl` / `relayDomain` / `hostId` / `hostToken` / `publicPort` /
`tlsCertFile` / `tlsKeyFile` / `push*` / `apns*` / `groupEnabled` 等）已从 schema 删除，
读取时会被忽略、保存时不会写回。升级后：

1. 在设置页填「服务地址（注册 API 端口）+ 邀请码」→ 点「注册并连接」；
2. 删除不再使用的旧 TLS 证书 / 设备 CA 目录（如 `~/.dsh/remote-access/certs`、
   `~/.dsh/remote-access/device-ca`）与 `push-tokens.json`（可选，插件不再读取）；
3. 旧手机 App 的 mTLS 连接不再可用，需用群体版 App 重新扫码配对。
4. 「允许手机重启主机 DSH」已迁移到「设置 → 远程访问 → 组件」；默认仍为关闭，需要时再打开。
5. 0.2.2 起，若配置里缺 `groupServiceUrl` 但 `group.json` 有注册信息，启动/保存时会自动回填
   （历史配置迁移；不需要重新填服务地址或重新注册）。

---

## 4. 群体版配对

1. 桌面 DSH → 设置 → 远程访问 → 「配对新设备」；
2. 生成 **QR v2**：`dshremote://pair?v=2&host=<hostId>.<relayDomain>&port=<mobilePort>&auth=token&...`
   （端口中 relay 下发；二维码 2 分钟有效，可重新生成）；
3. 手机 App 扫码 → 生成/读取本机 P-256 设备密钥 → 经 `https://<host>:<port>/__dsh_remote/pair`
   提交**公钥**（不再签 CSR / 证书）；
4. `requireApproval` 开启时，手机显示 SAS 短码，桌面端核对后点「允许配对」；
5. 主机把设备公钥写入本机设备表并 `device.sync` 给 relay；手机随后用
   challenge → 签名 → 短时 token 换 `__Host-dsh_session` cookie 访问。

设备可在设置页「已配对设备」中撤销；已撤销设备记录可删除。撤销会同步 relay，原 token 随即失效。

---

## 5. 本机接口

Host 半注册在 DSH 本机 webServer：

| 路径 | 说明 |
|---|---|
| `/api/remote-access/*` | 本机管理 API（仅本机已认证 DSH 浏览器；远程代理 deny 该前缀） |

管理 API：

| 路由 | 说明 |
|---|---|
| `GET /api/remote-access/status` | 群体服务 / 隧道 / 配对 / 设备 / 配置状态 |
| `POST /api/remote-access/config` | 更新群体版配置字段 |
| `POST /api/remote-access/group/register` | `{serviceUrl, inviteCode, hostName?}` 注册并连接（邀请码一次性） |
| `POST /api/remote-access/group/retry` | 重试隧道连接（先刷新注册信息，不消耗邀请码） |
| `POST /api/remote-access/group/reset` | 停止隧道、删除 `group.json`、关闭远程访问 |
| `POST /api/remote-access/pairing/start` | 创建一次性配对会话（QR v2） |
| `POST /api/remote-access/pairing/cancel` | 取消当前配对会话 |
| `POST /api/remote-access/pairing/approve` | `{pairingId, ticket?, approve}` 桌面确认/拒绝 |
| `GET /api/remote-access/pairing/qr.svg` | 当前配对二维码 SVG |
| `GET /api/remote-access/devices` | 设备列表 |
| `POST /api/remote-access/devices/:id/revoke` | 撤销设备 |
| `POST /api/remote-access/devices/:id/delete` | 删除已撤销设备记录 |
| `GET /api/remote-access/audit?limit=N` | 最近审计 |

经隧道（relay 边缘 → 主机 loopback 代理）提供：

| 路径 | 说明 |
|---|---|
| `POST /__dsh_remote/pair` | 手机提交配对公钥（一次性 secret + 主机确认保护） |
| `GET /__dsh_remote/pair/status?ticket=` | 配对确认轮询 |
| `GET /__dsh_remote/status` | 已鉴权设备可读的主机状态（见下） |
| `POST /__dsh_remote/restart` | 远程重启主机 DSH（「组件」开启时；成功 202） |

会话凭证两种等价形式（同一份 relay 签名 token、同一套离线验签 + 设备在册校验）：

- `Cookie: __Host-dsh_session=<token>`（浏览器 WebView / relay 边缘默认）；
- `Authorization: Bearer <token>`（App 的 JS fetch；本轮 relay 边缘以 cookie 为主，
  该形式保留为等价兜底，便于本地调试与后续 relay 支持）。

`GET /__dsh_remote/status` 响应（`ok:true`）：

```json
{
  "ok": true,
  "data": {
    "hostId": "h_ab12cd34",
    "hostName": "<HOST_NAME>",
    "domain": "h_ab12cd34.relay.example.com",
    "auth": "token",
    "relayConnected": true,
    "version": "0.2.2",
    "restart": { "enabled": false },
    "devices": { "total": 1, "active": 1 }
  }
}
```

`POST /__dsh_remote/restart`（仅 POST，body 可为空）：

| 状态 | 响应 | 含义 |
|---|---|---|
| 202 | `{"ok":true,"data":{"restarting":true,"mode":"desktop-app\|node"}}` | 已安排重启；helper 约 1s 后动手，先让 202 返回 |
| 401 | `{"ok":false,"error":{...}}` | 未带有效 token（cookie/Bearer 都没有或验签失败/设备已撤销） |
| 405 | `{"ok":false,"error":{"code":"method-not-allowed",...}}` | 非 POST |
| 501 | `{"ok":false,"error":{"code":"host-restart-disabled",...}}` | 「组件」未开启（`restartHostEnabled !== true`） |
| 501 | `{"ok":false,"error":{"code":"host-restart-unavailable",...}}` | 当前运行环境/插件版本未接入重启调度 |
| 501/500 | `{"ok":false,"error":{"code":"host-restart-unsupported\|host-restart-failed",...}}` | 环境不支持 / 调度 helper 失败 |

审计：`host-restart` 动作，`result=accepted`（`hash` = 重启模式或 `deduped`）或
`result=failed`（`hash` = `disabled` / `not-enabled` / 失败码），`deviceId` 为发起设备。

重启范围（`lib/host-restart.js`）：

- **DSH 桌面端（Electron shell + host 子进程）**：目标是整个应用进程组（shell 为组长），
  helper 直接 SIGKILL shell → 清理残留与旧单实例锁 → `open -n <DSH.app>` 重新拉起（失败再
  兜底直接执行 `.app` 内可执行文件）；不会只杀 host 子进程（那只会弹出崩溃对话框）。
- **Node CLI（`dsh web` 等）**：SIGTERM 本进程后按 `execPath + argv` 原样拉起。
- **Windows**：Desktop 以 shell pid 为 `taskkill` 目标再 `start` 重新拉起（best-effort）。
- 10s 去重窗口（连点/App 重试只调度一次）；`DSH_REMOTE_DISABLE_HOST_RESTART=1` 时一律拒绝
  调度（测试/CI 防误重启）。重启期间连接会短暂断开，随后 App 自动重连。

其余已鉴权请求反代到本机 DSH Web。`/api/remote-access/*` 对远程设备一律拒绝；
`/connect`、`/oauth`、`/open-in-app*`、`/__dsh_remote/*` 等本机敏感路径同样在 deny list 内。

**群体版不提供**：主机 TLS 监听、设备证书/mTLS、APNs/Expo 推送。推送等能力由服务端
（部署方）在后续切片统一提供，主机端不再发送任何通知。

---

## 6. 安全边界

- 设备身份只来自**配对登记的设备公钥 + relay 签名的短时 token**；主机逐请求验签
  并校验 token 里的 deviceId 仍在本机有效设备表。不信任来源 IP / `X-Forwarded-*` /
  `Host` / `Origin` / 自定义头 / 查询参数。
- relay 边缘终止手机 TLS、可见明文（A 模式取舍，风险与责任由部署方承担）；
  主机只做出站隧道，不持有 TLS 私钥。
- DSH Web 始终保持 `127.0.0.1`，插件剥离客户端头并注入本机 session cookie。
- 配对密钥一次性且短时有效；主机侧另有限速；设备可撤销（主机设备表 + relay 会话双保险）。
- 管理 API 只允许本机已认证 DSH 浏览器访问；远程设备一律 403。
- 「重启主机」组件默认关闭；开启后仍要求设备 token 鉴权（cookie/Bearer 等价）且设备未撤销，
  关闭时接口直接 501，不进入调度。

---

## 7. 手机 WebView 的会话优化（Client 半）

只在手机远程 WebView（`?dsh_mobile=1`）下生效，桌面本机不安装：

- **会话热缓存**：最近打开过的 3 个会话额外 retain，切走不销毁、切回 0 快照；
  超过按 LRU 释放；连接断开时只保留当前会话。可用
  `globalThis.__DSH_REMOTE_SESSION_CACHE__ = { max }` 调整。
- **历史分片**：把 `session/follow` 首屏与 `session/page` 翻页请求窗口压小
  （`6/3/1`、`10/5/1`，turn rail 跳转 `30/15/2`），历史交给 DSH 既有「加载更早」按需补齐。
  调试覆盖：`globalThis.__DSH_REMOTE_MOBILE_HISTORY__`。
- **后台恢复重连**：页面隐藏 ≥10s 再恢复时主动重连，刷新会话状态。

细节见 [`docs/diagnostics/04-session-switch-cache.md`](../../../docs/diagnostics/04-session-switch-cache.md)
与 [`09-mobile-history-slicing.md`](../../../docs/diagnostics/09-mobile-history-slicing.md)。

> 新增/修改 WebSocket 反向代理时注意保留 `lib/ws-keepalive.js` 的心跳桥：DSH 的
> `/api/remote.mux` 每 2s Ping，慢速链路上大帧会挤占心跳，代理需要及时代答 Pong，
> 否则 DSH 会误判失联并踢掉 mux。

---

## 8. 排障

| 现象 | 检查 |
|---|---|
| 注册报 `not_found`（控制面只接受 WebSocket upgrade） | 「服务地址」填成了控制面 WSS 端口；要填**注册 API** 地址（如 `https://relay.example.com:19443`） |
| 注册报 `invalid_invite` / `invite_expired` / `host_limit` | 邀请码错误/过期/用尽，或该部署方主机配额已满；找部署方重新生成 |
| 隧道断开 | 服务地址可达？`group.json` 里 `controlUrl` 是否与当前服务端一致？点「重试连接」或重新注册 |
| 手机 401/403 | 设备是否已配对且未撤销？token 是否过期（App 自动续期）？Host 头是否等于 `<hostId>.<relayDomain>`？ |
| 配对二维码过期 | 重新生成；配对密钥 ≤120s |
| 手机打开管理 API 403 | 这是安全边界；远程设备不允许管理本地插件 |
| App 没有「重启主机 ⏻」按钮 / 重启报 `host-restart-disabled` | 「设置 → 远程访问 → 组件」里「允许手机重启主机 DSH」是否开启（默认关闭）；保存后 status 的 `restart.enabled` 应为 true |
| 重启报 `host-restart-unavailable` / `host-restart-unsupported` | 插件版本过旧，或当前运行环境不支持自动重启（如 Linux Electron）；本机手动重启仍可用 |
| 打开大会话「载入历史」很久/失败、画面反复跳 | 插件是否为 ≥0.2.0 且已重启 DSH；手机 WebView 是否加载了历史分片；relay 带宽/线路是否过慢；详见 [`docs/diagnostics/09-mobile-history-slicing.md`](../../../docs/diagnostics/09-mobile-history-slicing.md) |

---

## 9. 相关文档

- [relay / 服务器部署](../../../server/README.md)
- [群体服务设计（A 模式）](../../../docs/group-service-design.zh.md)
- [服务器 / relay 部署](../../../server/README.md)
- [根目录 README](../../../README.md)

---

## 10. 变更记录

- **0.2.2**（修复「允许手机重启主机 DSH」保存失败）
  - 历史配置迁移：`config.groupServiceUrl` 为空但有 `group.json` 注册信息时，启动/保存时
    自动用 `registration.serviceUrl` 回填并落盘（此前会因 `validateConfig` 报
    「groupServiceUrl（服务地址）未配置」，导致任何保存都被 admin 拒绝为 400 invalid-config，
    隧道也无法重建）。
  - 配置保存按需重建运行时：只有 `enabled` / `groupServiceUrl` / `groupInviteCode` /
    `hostName` / `groupPlainPort` / `groupInsecure` / `requireApproval` / `denyPrefixes` /
    `maxRequestBytes` 变化才重建 `GroupHostRuntime`；`restartHostEnabled` 等纯状态字段
    只持久化，保存立即返回、隧道不重连。
  - 非运行时字段的保存不再返回历史遗留校验错误（`updateConfig` 返回 `applied` 标记，
    admin 只在真正重建运行时失败时返回 400）。
  - 停止 loopback 代理时主动断开残留 keep-alive/活跃连接，避免重建等待连接自然结束造成的
    「保存很慢/超时」。
  - 客户端保存失败 toast 带上服务端 `error.message` 与 `error.code`（如 `invalid-config`）。
- **0.2.1**：群体版恢复「重启主机 DSH」可关闭组件（`restartHostEnabled`，默认关闭）、
  cookie/Bearer 等价鉴权、`GET /__dsh_remote/status` 返回 `restart.enabled`、
  `POST /__dsh_remote/restart`（202/401/405/501 + 审计）。
- **0.2.0**：群体服务版（A 模式）主机能力（注册/隧道/loopback 明文代理/群体配对）。
