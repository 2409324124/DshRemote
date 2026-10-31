# dsh-remoteService（群体服务版 · 服务端 monorepo）

> 许可证：**AGPL-3.0-or-later**（完整文本见根目录 [`LICENSE`](../LICENSE)）。

群体服务版（A 模式）的服务端实现：**relay（终止手机 TLS + 边缘鉴权 + 主机隧道）+ 主机插件
（DSH 主机，出站隧道 + loopback 明文代理）+ 共享协议**。本分支不包含 classic 单机版
（mTLS 盲中继）功能。

配套手机 App：[`../dsh-remoteApp`](../dsh-remoteApp)；公网部署：[`../server`](../server)；
设计与决策：[`../docs/group-service-design.zh.md`](../docs/group-service-design.zh.md)。

## 包结构

```
packages/protocol/     # 群体版协议：QR v2、边缘鉴权常量、控制消息、二进制流帧、
                       # challenge/token 签名与校验、cookie 解析
packages/host-plugin/  # DSH 主机插件：群体注册/刷新、控制面 WSS、设备同步、
                       # loopback 明文代理（HTTP + WebSocket upgrade）、群体配对、桌面设置 UI
packages/relay/        # 群体版 relay：控制面 WSS、注册 API、部署方后台 /__admin、
                       # 边缘 TLS（通配证书）、challenge→token 鉴权、主机隧道
```

## 快速导航

| 内容 | 文档 |
|---|---|
| 部署 relay（证书/端口/nginx/后台/邀请码） | [`../server/README.md`](../server/README.md) |
| 主机插件安装/配置 | [`packages/host-plugin/README.md`](packages/host-plugin/README.md) |
| relay 配置与运维 | [`packages/relay/README.md`](packages/relay/README.md) |
| 手机 App | [`../dsh-remoteApp/README.md`](../dsh-remoteApp/README.md) |
| 落地问题与根因记录 | [`../docs/diagnostics/12-group-service-issues.md`](../docs/diagnostics/12-group-service-issues.md) |

## 开发与测试

```bash
cd dsh-remoteService
npm install
npm test                    # 全部单元/集成测试
npm run validate:artifacts  # 部署产物静态校验（shell/Node 语法、JSON、relay dev compose）
```

## 安全边界（摘要）

- 设备身份只来自「配对注册的 P-256 设备公钥 + relay 签发的短时 token（15 分钟，native 续期）」；
  主机逐请求复验 token 并检查设备在册。
- relay 在 A 模式下终止手机 TLS（部署方可见明文，责任由部署方承担），是身份信任方；
  请部署在自己信任的服务器上。
- DSH Web 始终保持 `127.0.0.1` 回环，不暴露公网、不用 `--trusted-host`。
- 主机明文代理保留与 classic 一致的 deny list（`/connect`、`/oauth`、`/open-in-app*`、
  `/api/remote-access`、`/__dsh_remote/*` 等）。
- 撤销双保险：主机设备表删除 + relay 会话失效；控制连接断链立即生效。
- 邀请制注册（邀请码 + 限速 + 配额），不开放公开注册。
