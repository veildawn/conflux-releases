# Conflux 公开发行仓库

[English](README.md)

**Conflux** 团队多租户代理节点管理平台的公开二进制、安装脚本、Docker 镜像与 GPL-3.0 运行时源码合规产物发布仓库。支持五种自建协议（`VLESS+REALITY`、`Hysteria2`、`Trojan`、`Shadowsocks 2022`、`TUIC v5`）、外部订阅聚合与只读共享、6 类主流客户端精确编译、跨主机守恒额度账本（`L = U + R + A`）、Cloudflare DNS 与 ACME 自动化，以及内嵌双语控制台。

## 快速部署

### Linux 控制端安装（systemd，`amd64` / `arm64`）

```sh
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/release-pubkey.pem -o /tmp/conflux-release-pubkey.pem
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/install-server.sh \
  | sudo bash -s -- \
    --public-base-url https://conflux.example.com \
    --database-url 'postgres://conflux:password@127.0.0.1:5432/conflux?sslmode=disable' \
    --release-pubkey-pem /tmp/conflux-release-pubkey.pem
```

### 受管主机 Agent + Runtime 接入（systemd，`amd64` / `arm64`）

在管理后台「主机管理 → 接入 VPS」获取 15 分钟一次性注册令牌后，在目标 VPS 上执行：

```sh
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/release-pubkey.pem -o /tmp/conflux-release-pubkey.pem
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/install-agent.sh \
  | sudo bash -s -- \
    --server https://conflux.example.com \
    --enroll-token '<15分钟注册令牌>' \
    --release-pubkey-pem /tmp/conflux-release-pubkey.pem
```

### Docker Compose 部署（`amd64` / `arm64`）

```sh
mkdir -p conflux && cd conflux
curl -fsSL https://raw.githubusercontent.com/veildawn/conflux-releases/main/deploy/docker-compose.yml -o docker-compose.yml
```

在同目录创建 `.env`（填写 `CONFLUX_PUBLIC_BASE_URL`、`CONFLUX_MASTER_KEY_HEX`（由 `openssl rand -hex 32` 生成）以及 `POSTGRES_PASSWORD`），然后启动：

```sh
docker compose up -d
```

无论是原生 `systemd` 还是 Docker Compose 部署，均支持在控制面板**侧边栏顶部版本徽章**内一键检查更新并安全升级（Docker 模式下无需挂载 `/var/run/docker.sock`，控制端会将通过 Ed25519 验签与 SHA-256 校验的新二进制落盘到持久化卷 `/var/lib/conflux/bin` 并平滑重启接管）。

## 签名校验与 GPL-3.0 源码合规

每个 Release 均包含：
- `conflux-server_<version>_linux_<arch>.tar.gz`（控制端一键自更新压缩包）
- 多架构独立二进制（`conflux-server`、`conflux-agent`、`conflux-runtime`、`conflux-updater`、`conflux-ctl`）
- `checksums.txt` 与 `checksums.txt.sig`（对应 `release-pubkey.pem` 的 Ed25519 签名）
- 各组件 `*.manifest.json` 及汇总 `manifests.json`
- `conflux-runtime-<version>-source.tar.gz`（与所发布 `conflux-runtime` 二进制完全对应的 GPL-3.0-or-later 源码包）
