# DSH Remote · 群体服务版 relay 部署手册

本目录用于在公网 VPS 上部署**群体服务版（A 模式）** relay。
适用版本：relay **0.3.0-group**（本分支只提供群体版，不包含 classic 单机版）。

> **安全前提（务必先读）**：A 模式下 relay 在边缘**终止手机 TLS**，部署方可以看到会话明文
> 与元数据，风险与责任由部署方承担；relay 同时也是设备身份的信任方，请部署在自己信任的
> 服务器上。设计细节见 [`../docs/group-service-design.zh.md`](../docs/group-service-design.zh.md)。

## 1. 组成与端口

```
手机 App ──https──► relay 边缘 :443（*.<relayDomain> 通配证书，终止 TLS）
                     │  challenge/token 鉴权
                     ▼
主机插件 ──wss──► 控制面 :8443（主机出站隧道；主机插件主动连 relay）
                     │
部署方 ──https──► 注册 API :9443（主机自助注册/刷新）
                 ──https──► 后台 :9444（/__admin，只绑回环）
```

容器内固定端口：edge=`443`、控制面 WSS=`8443`、注册 API=`9443`、后台=`9444`、healthz=`80`。
宿主机映射与公网端口在 `.env.group.example` 中配置，原则只有两类：

- **★ 公网入口**（要放行/由 nginx 转发）：`RELAY_GROUP_MOBILE_PORT`（手机访问）、
  `RELAY_GROUP_PUBLIC_CONTROL_PORT`（主机 WSS）、`RELAY_GROUP_API_HOST_PORT`（注册 API）；
- **内部/回环**（不要对公网放行）：admin、healthz，以及 nginx 前置形态下的 edge/control 映射。

> `RELAY_GROUP_MOBILE_PORT` / `RELAY_GROUP_PUBLIC_CONTROL_PORT` 必须等于**手机/主机实际连接的
> 公网端口**（nginx listen 端口），不是容器端口、也不是宿主机映射端口。

## 2. 前置条件

- 一台 Linux VPS（x86_64），Docker + Compose v2；
- 域名：`relay.example.com` 与 `*.relay.example.com` 都解析到该 VPS；
- 两张证书（生产必须公网 CA，手机才信任；自签只适合 POC）：

  | 目录（`data/certs/` 下） | 证书 | 覆盖域名 | 用途 |
  |---|---|---|---|
  | `<RELAY_DOMAIN>/` | 控制面证书 | `<RELAY_DOMAIN>`（apex） | 主机 WSS、注册 API、后台 |
  | `wildcard/` | 边缘通配证书 | `*.<RELAY_DOMAIN>` | 手机访问 `<hostId>.<RELAY_DOMAIN>`，relay 终止 TLS |

  - `fullchain.pem`（证书+中间链）+ `privkey.pem`（私钥）；私钥 `600`、目录 `700`；
  - 通配证书只能 DNS-01 签发（`*.domain` 不能用 HTTP-01），例如
    `acme.sh --issue --dns dns_xxx -d '<RELAY_DOMAIN>' -d '*.<RELAY_DOMAIN>'`；
  - 也可以申请一张同时含 apex + `*.<domain>` 的 SAN 证书，复制到两个目录。

## 3. 部署（两种形态）

### 3.1 准备变量与证书

```bash
cd server
cp .env.group.example .env.group      # 至少改 RELAY_DOMAIN；端口/镜像/admin token 按需改
mkdir -p data/certs/<RELAY_DOMAIN> data/certs/wildcard
# 把两张证书的 fullchain.pem / privkey.pem 放进对应目录
chown -R 100:101 data                 # 容器内 relay 用户 UID/GID
```

`docker compose` 默认只读 `.env`；文件叫 `.env.group` 时必须显式 `--env-file .env.group`
（或改名为 `.env`）。**首次启动会根据 `.env.group` 自动生成 `data/config/relay.json`**
（已存在不覆盖；高级项可手改，字段参考 `group/relay.group.example.json`）。

### 3.2 启动 relay

```bash
# A. 本机构建（开发机/Apple Silicon 记得指定 linux/amd64）
DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose --env-file .env.group -f docker-compose.group.yml up -d --build

# B. 服务器离线镜像（推荐生产）
docker load -i dsh-relay-0.3.0-group-linux-amd64.tar
docker compose --env-file .env.group -f docker-compose.group.yml up -d --no-build
# 校验离线包：sha256 应为
# 206208af1a28a4d0e78139216ebbffd431f3485efab7f7d4bc79c17518691ce0
```

生产服务器建议整理成标准目录（Compose 不关心文件名，只关心 `-f` 与 env 文件）：

```text
/opt/dsh-group/
├── docker-compose.yml      # = 仓库 server/docker-compose.group.yml
├── .env                    # = 仓库 server/.env.group.example（改端口/镜像/域名）
└── data/                   # 挂载进容器：config/relay.json、certs/、group.json、admin-token…
```

```bash
cd /opt/dsh-group
chmod 600 .env
docker compose config          # 确认变量插值
docker compose up -d --no-build
docker compose ps && docker compose logs -f relay
```

### 3.3 nginx 前置（推荐）

relay 自己终止手机 TLS，nginx **只做 L4 SNI 透传**，不要做 HTTPS 终止/HTTP 反代
（否则边缘证书与设备 cookie 都会出问题）。示例见
[`nginx/group-nginx-stream.conf`](nginx/group-nginx-stream.conf)：把公网 `443 → 127.0.0.1:<edge映射>`、
公网 `8443 → 127.0.0.1:<control映射>`，按 SNI 分流；然后在 `.env.group` 里把
`RELAY_GROUP_EDGE_BIND` / `RELAY_GROUP_CONTROL_BIND` 改为 `127.0.0.1`。

只有一台 relay 时也可直连（不装 nginx）：把 edge/control 映射到公网端口，并让
`RELAY_GROUP_MOBILE_PORT` / `RELAY_GROUP_PUBLIC_CONTROL_PORT` 等于这两个公网端口。

**注册 API（19443）** 既可以直接对公网，也可以让 nginx 终止 TLS 后反代到
`https://127.0.0.1:19443`：此时把 `RELAY_GROUP_API_HOST_BIND=127.0.0.1`，并且**必须**把
`Host` 设为 `<RELAY_DOMAIN>`（注册响应里的 `controlUrl` 用请求 Host 生成，Host 错了主机回连
隧道会失败）；参考 [`nginx/group-api-http.conf`](nginx/group-api-http.conf)。注意反代后注册
限速按 127.0.0.1 共享桶，必要时调大 `registration.maxAttemptsPerMinute`。

**后台（9444）** 默认只绑 `127.0.0.1:19444`，不要暴露公网：用 SSH 隧道
（`ssh -L 19444:127.0.0.1:19444 <VPS>`）或按
[`nginx/group-admin-http.conf`](nginx/group-admin-http.conf) 反代到独立 admin 域名 + basic auth。

## 4. 部署方后台与邀请码

- 后台地址：`https://127.0.0.1:19444/__admin/`（经 SSH 隧道/nginx 反代访问）；
- admin token 首次启动自动生成到 `data/admin-token`（0600），也可显式设 `RELAY_ADMIN_TOKEN`：

  ```bash
  docker exec dsh-relay-group node packages/relay/bin/relay.js admin-token --config /data/config/relay.json
  ```

- 生成邀请码（后台「生成邀请码」或 CLI）：

  ```bash
  docker exec dsh-relay-group node packages/relay/bin/relay.js invite create --note "friend A" --config /data/config/relay.json
  ```

  复制给群成员的是**邀请码本身（43 字符）**，不是列表里的 `inv_xxx` ID；
  后台列表里可随时复制；旧版本（哈希存储）的邀请显示「旧邀请码（不可显示）」，需重新生成。

## 5. 主机与 App

1. DSH 主机安装本分支 host-plugin（**0.2.4**）；DSH「设置 → 远程访问 → 群体服务」填：
   - 服务地址：`https://<relayDomain>:19443`（注册 API，不是控制面端口）；
   - 邀请码：后台生成的邀请码；
   - 点击「注册并连接」；成功后显示 hostId、隧道地址、连接状态。
2. 手机 App（**0.1.1**，iOS build 7）扫码配对：DSH「生成配对二维码」→ App 扫码 →
   桌面确认 → 进入主机。
3. 主机侧设备列表可随时撤销设备（relay 会话即时失效）；设备 lastSeen 在群体版鉴权成功时更新。

> 从旧版本升级：服务地址以前可能填的是控制面端口，请改成注册 API 地址并重新注册一次；
> 改过 `RELAY_GROUP_MOBILE_PORT` / `RELAY_GROUP_PUBLIC_CONTROL_PORT` 后，主机插件「重试连接」
> 会自动刷新注册信息（`POST /__group/refresh`），必要时重置注册。

## 6. 验证与排障

### 6.1 基础验证

```bash
# relay 日志（正常应有 host online / device.sync / edge: request 等）
docker logs -f dsh-relay-group

# healthz
curl -sk https://127.0.0.1:<health端口>/healthz

# 注册 API（无请求体应返回 400 JSON，而不是连接失败）
curl -sk -X POST https://<relayDomain>:19443/__group/register -H 'content-type: application/json' -d '{}'

# 边缘端口 TLS 证书（应返回通配证书）
openssl s_client -connect <relayDomain>:<mobilePort> -servername <hostId>.<relayDomain> </dev/null 2>/dev/null | openssl x509 -noout -subject
```

### 6.2 常见问题

| 现象 | 排查 |
|---|---|
| 容器反复重启、日志 `cannot read config file ... ENOENT` | 镜像启动会从 env 自动生成 `data/config/relay.json`；检查 `data/` 挂载与权限（`chown 100:101`） |
| 容器报「缺少 RELAY_DOMAIN」 | `docker compose` 没读到 env 文件：带 `--env-file .env.group`，或改名为 `.env`；`docker compose ... config | grep RELAY_DOMAIN` 自检 |
| 改了 env 但端口/镜像没变 | 需要 `up -d --force-recreate`（`restart` 不更新端口映射）；`docker inspect dsh-relay-group --format '{{.Config.Image}}'` 确认镜像 tag |
| 宿主端口仍是 0.0.0.0 | compose 文件缺少 `*_HOST_BIND` 变量行（用仓库最新 compose），或未重建容器 |
| 主机日志 `relay control socket error` | `RELAY_GROUP_PUBLIC_CONTROL_PORT` 必须等于主机实际连接的 nginx listen 端口；改后主机「重试连接」或重置注册 |
| 手机 App 配对后一直 401 | 先看 relay 日志是否有对应 `edge: request POST /__group/auth/...`；若 App 错误页响应体是 `{"ok":false,"error":"..."}`（字符串 error），说明请求被透传到主机，检查 relay 版本（≥0.3.0）与是否有多实例/代理复用连接 |
| App 能开页面但「暂无会话/一直重连」 | host-plugin 需 ≥0.2.0（loopback 明文代理已代理 WebSocket Upgrade）；确认 DSH 已重启加载新插件 |
| 设备列表「从未连接」 | host-plugin ≥0.2.0 在群体鉴权成功时更新 lastSeen；重启 DSH 后应正常 |
| 后台打不开 | 后台只绑回环：用 SSH 隧道或 nginx 反代；不要直接把 19444 暴露公网 |
| 手机/主机关联到旧端口 | `RELAY_GROUP_MOBILE_PORT`/`PUBLIC_CONTROL_PORT` 配错或未重建；主机「重置注册」重新拿二维码 |

### 6.3 与其它服务同机共存

如果这台机器已有别的网站/relay 占用 443/8443/1443/18443/18080，把 group 的内部端口
（`RELAY_GROUP_EDGE_HOST_PORT` / `RELAY_GROUP_CONTROL_HOST_PORT` / `RELAY_GROUP_HEALTH_HOST_PORT`）
改成 16xxx/18xxx 等空闲端口，公网端口仍按 nginx 实际 listen 填；nginx 的 `stream {}` 全局只有一份，
把 group 的 SNI 段合并进现有 stream 配置即可。group 必须有**自己的 relay 域名**与通配证书。

## 7. 升级、备份与安全

### 7.1 从旧版本升级（classic/单体版 → 群体版）

本分支已删除 classic 单机版；如果服务器上还跑着旧 relay（0.2.x / classic 镜像）：

1. `docker load` 新镜像 `dsh-remote-relay:0.3.0-group`，改 `.env` 的 `DSH_RELAY_IMAGE`，`up -d --force-recreate`；
2. 旧 `data/config/relay.json` 会被保留（entrypoint 不覆盖）：里面的 classic 字段
   （`mode`/`dataPort`/`tokensFile`/`hostTokens`/`push`）在新版会被忽略并打 warn；建议对照
   `group/relay.group.example.json` 手动精简一次，或备份后删除让 entrypoint 重新生成；
3. `data/group.json`、`signing-key.pem`、`admin-token`、证书可沿用（主机注册/会话不丢）；
4. 主机插件升级到 **0.2.0** 后需重启 DSH；旧 classic 配置字段会被丢弃，群体服务地址/邀请码保留，
   必要时「重置注册」重新注册；App 升级到 **0.1.1** 后需重新扫码配对（旧 classic/单体连接会被丢弃）；
5. 升级后首次验证：relay `/healthz` 200、主机注册成功、App 配对→进入正常、前后台不再收到任何推送。

- **升级**：`docker load` 新镜像 → 改 `.env` 的 `DSH_RELAY_IMAGE` → `up -d --force-recreate`；
  改过镜像 tag/端口后必须重建容器。
- **备份**：整个 `data/`（`group.json` 主机/邀请码、`signing-key.pem` 会话 token 签名私钥、
  `admin-token`、证书）。`group.json` 含明文邀请码、`signing-key.pem`/`admin-token` 泄露等于
  身份泄露，**不要入库/上传/外发**。
- **安全**：边缘与注册 API 必须对群成员网络可达；后台只回环；admin token 只给部署方；
  不使用自签证书对外（手机会拒绝）；relay 机器被攻破等同于部署方被冒充，务必做好主机安全。

## 8. 相关文档

- 设计与安全决策：[`../docs/group-service-design.zh.md`](../docs/group-service-design.zh.md)
- 落地问题与根因记录：[`../docs/diagnostics/12-group-service-issues.md`](../docs/diagnostics/12-group-service-issues.md)
- 主机插件：[`../dsh-remoteService/packages/host-plugin/README.md`](../dsh-remoteService/packages/host-plugin/README.md)
- 手机 App：[`../dsh-remoteApp/README.md`](../dsh-remoteApp/README.md)
