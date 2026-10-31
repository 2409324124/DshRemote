# dsh-remote-service（主机插件 · 群体服务版）

> 许可证：**AGPL-3.0-or-later**（完整文本见根目录 [`LICENSE`](../../../LICENSE)）。
>
> 版本 **0.2.0**：本插件已剥离 classic/单体版的 mTLS、主机证书、自托管推送与远程重启，
> **只保留群体服务版（A 模式）主机能力**。旧版配置字段会被忽略，升级后需要重新注册一次。

运行在 **DSH 主机** 上的远程访问插件，提供：

- 向部署方 relay 的**自助注册**（邀请码）与注册信息刷新；
- 主动外连 relay 控制面的**出站隧道**（主机不开放任何入站端口、不持有 TLS 证书）；
- 本机 **loopback 明文 HTTP/WS 代理**：逐请求校验 relay 签名 token + 设备在册后反代到
  `127.0.0.1` 的 DSH Web；
- **群体版设备配对**（二维码 v2 + 一次性密钥 + 桌面确认 + P-256 设备公钥登记）；
- 桌面 DSH 设置页 / 侧栏入口（注册、状态、配对、设备撤销、审计）；
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
| `denyPrefixes` | 反代 deny list（`/connect`、`/oauth`、`/open-in-app`、`/api/remote-access`、`/__dsh_remote`） |
| `maxRequestBytes` | 远程代理请求体上限（默认 256 MiB） |

注册结果落盘在 `~/.dsh/remote-access/group.json`（0600）：`hostId` / `hostToken` /
`relayDomain` / `mobilePort` / relay token 公钥 / `serviceUrl` / `controlUrl` / `hostName`。
**不要**把 `group.json`、邀请码或配对二维码提交到仓库 / 截图 / 日志里。

### 从单体版（≤0.1.x）升级

classic 配置字段（`relayUrl` / `relayDomain` / `hostId` / `hostToken` / `publicPort` /
`tlsCertFile` / `tlsKeyFile` / `push*` / `apns*` / `restartHostEnabled` 等）已从 schema 删除，
读取时会被忽略、保存时不会写回。升级后：

1. 在设置页填「服务地址（注册 API 端口）+ 邀请码」→ 点「注册并连接」；
2. 删除不再使用的旧 TLS 证书 / 设备 CA 目录（如 `~/.dsh/remote-access/certs`、
   `~/.dsh/remote-access/device-ca`）与 `push-tokens.json`（可选，插件不再读取）；
3. 旧手机 App 的 mTLS 连接不再可用，需用群体版 App 重新扫码配对。

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
| `GET /__dsh_remote/status` | 已鉴权设备可读的主机状态（token 模式） |

其余已鉴权请求反代到本机 DSH Web。`/api/remote-access/*` 对远程设备一律拒绝；
`/connect`、`/oauth`、`/open-in-app*`、`/__dsh_remote/*` 等本机敏感路径同样在 deny list 内。

**群体版不提供**：主机 TLS 监听、设备证书/mTLS、APNs/Expo 推送、远程重启主机。
推送等能力由服务端（部署方）在后续切片统一提供，主机端不再发送任何通知。

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
| 打开大会话「载入历史」很久/失败、画面反复跳 | 插件是否为 ≥0.2.0 且已重启 DSH；手机 WebView 是否加载了历史分片；relay 带宽/线路是否过慢；详见 [`docs/diagnostics/09-mobile-history-slicing.md`](../../../docs/diagnostics/09-mobile-history-slicing.md) |

---

## 9. 相关文档

- [relay / 服务器部署](../../../server/README.md)
- [群体服务设计（A 模式）](../../../docs/group-service-design.zh.md)
- [服务器 / relay 部署](../../../server/README.md)
- [根目录 README](../../../README.md)
