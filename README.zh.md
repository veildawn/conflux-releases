# Conflux 公开发行仓库

[English](README.md)

**Conflux** 的公开下载、部署模板、CI 与版本发布仓库。Conflux 是多租户代理节点管理平台，支持五种内置协议、外部订阅共享、六类客户端订阅编译、守恒流量额度（`L = U + R + A`）、DNS/ACME 自动化与双语控制台。

当前稳定版本为 [v0.0.28](https://github.com/veildawn/conflux-releases/releases/tag/v0.0.28)，历史版本见[发布列表](https://github.com/veildawn/conflux-releases/releases)。应用源码在私有仓库 `veildawn/conflux-service` 中维护；**所有 GitHub Actions 均在本公开仓库运行**，私有仓库的 Actions 已关闭。

客户端直接连接所选节点，由节点向目标网站转发流量。控制服务器负责订阅、配置与额度分发，普通代理数据不经过控制服务器；显式配置节点链路时才会增加指定的中转节点。五种内置协议为 VLESS + REALITY、Hysteria2、Trojan、Shadowsocks 2022 和 Snell v6，外部订阅也支持 TUIC。

## 下载

| 产物 | 支持平台 |
| --- | --- |
| `conflux-server`、`conflux-agent`、`conflux-runtime`、`conflux-updater`、`conflux-ctl` | Linux `amd64` / `arm64` |
| `conflux-agent`、`conflux-ctl` | macOS（`darwin`）`amd64` / `arm64` |
| `conflux-agent`（`monitor_only`） | Windows `amd64` / `arm64` |
| `ghcr.io/veildawn/conflux-release:0.0.28` | Linux `amd64` / `arm64` 容器镜像 |
| `conflux-runtime-0.0.28-source.tar.gz` | 嵌入 `Xray-core 26.11.9` 与内置 Snell v6 的 GPL-3.0-or-later 运行时对应源码、依赖锁定文件、构建说明和许可证声明 |

[GHCR 镜像](https://github.com/veildawn/conflux-releases/pkgs/container/conflux-release)已公开，无需登录即可拉取。镜像包名为 **`conflux-release`**（单数），仓库名为 **`conflux-releases`**（复数）。部署时固定版本或镜像摘要，便于复现；`latest` 是可变标签。

## 安装

以下示例明确安装 **0.0.28**。原生控制端与 Agent 安装器支持使用 systemd 的 Debian 12/13、Ubuntu 22.04/24.04/26.04，以及 `amd64` / `arm64` 架构；需要 root、本地存储、`curl`、支持 Ed25519 的 OpenSSL、`sha256sum` 和 Python 3。`monitor_only` 监控模式 Agent 还支持 Linux、macOS、Windows 与容器部署；独立 CLI 支持 Linux 和 macOS。

### 下载并校验安装脚本

在新的工作目录中执行。校验清单、签名与脚本必须来自同一版本，验签成功后再执行安装器：

```sh
set -eu
RELEASE_VERSION=0.0.28
RELEASE_URL="https://github.com/veildawn/conflux-releases/releases/download/v${RELEASE_VERSION}"
for file in release-pubkey.pem checksums.txt checksums.txt.sig install-server.sh install-agent.sh install-ctl.sh; do
  curl -fsSL "$RELEASE_URL/$file" -o "$file"
done

# DER 编码 SPKI 公钥的 SHA-256 指纹。
EXPECTED_KEY_SHA256=5e7b7b3c6bf91b15c5a7afcf93fe9904ab329d958d63fba5816566e2bab48ea7
ACTUAL_KEY_SHA256="$(openssl pkey -pubin -in release-pubkey.pem -outform DER | openssl dgst -sha256 | awk '{print $NF}')"
test "$ACTUAL_KEY_SHA256" = "$EXPECTED_KEY_SHA256"
openssl base64 -d -A -in checksums.txt.sig -out checksums.txt.sig.bin
openssl pkeyutl -verify -pubin -inkey release-pubkey.pem -rawin \
  -in checksums.txt -sigfile checksums.txt.sig.bin
python3 - <<'PY'
import hashlib
from pathlib import Path
sums = dict(line.split()[::-1] for line in Path("checksums.txt").read_text().splitlines())
for name in ("release-pubkey.pem", "install-server.sh", "install-agent.sh", "install-ctl.sh"):
    actual = hashlib.sha256(Path(name).read_bytes()).hexdigest()
    if actual != sums[name]:
        raise SystemExit("SHA-256 mismatch: " + name)
print("Installer signatures and checksums verified")
PY
```

以上指纹及仓库中的 [release-pubkey.pem](release-pubkey.pem) 是公开的信任参考。保留可信副本用于后续校验；仅从同一个下载地址同时获取公钥和签名，并不能独立建立信任。

### Linux 控制端

在已校验的下载目录中执行，填写已有 PostgreSQL 数据库连接和对外 HTTPS 地址：

```sh
sudo bash ./install-server.sh --version 0.0.28 \
  --public-base-url https://conflux.example.com \
  --database-url 'postgres://conflux:REPLACE_WITH_PASSWORD@127.0.0.1:5432/conflux?sslmode=disable' \
  --release-pubkey-pem "$PWD/release-pubkey.pem"
sudo systemctl status conflux-server
sudo journalctl -u conflux-server -n 50 --no-pager
```

原生安装默认监听 `127.0.0.1:8080`，需要为公开地址配置支持 WebSocket 的 HTTPS 反向代理。使用新数据库启动时，日志会输出有效期 30 分钟的一次性初始化令牌；打开控制台完成初始化，并为管理员启用 MFA。备份 `/etc/conflux`（包括主密钥和签名密钥）、`/var/lib/conflux` 以及 PostgreSQL 数据。

### 受管主机 Agent 与 Runtime

在管理后台「主机管理 → 接入 VPS」生成 15 分钟有效的注册令牌。在目标主机上将令牌存入 root 可读、权限为 `0600` 的文件，再运行已校验的安装器：

```sh
sudo bash ./install-agent.sh --version 0.0.28 \
  --server https://conflux.example.com \
  --enroll-token-file /root/conflux-enroll-token \
  --release-pubkey-pem "$PWD/release-pubkey.pem"
sudo systemctl status conflux-agent conflux-runtime
```

SQLite WAL 账本必须保存在本地存储。只有显式传入 `--configure-firewall` 时安装器才会修改防火墙规则；请按实际协议端口配置主机防火墙和云安全组。

### Snell v6

自 **v0.0.20** 起，Snell v6 已直接内置于 `conflux-runtime`（自维护实现，无需下载外部 `sing-box` 或启动独立子进程；安装器的 `--with-snell-v6` 参数保留为兼容空操作）。升级至内置版本的主机将自动上报 `proto_snell_v6_v1`，在「主机管理」中展示为「已内置」，无需额外下载或卸载组件；此前采用可选组件模式的 `v0.0.18`–`v0.0.19` 节点在灰度滚动升级期间仍可平滑兼容，完成常规升级后即自动切换为内置能力。

在「节点」中选择已内置（或仍上报该能力）的主机，即可直接创建 Snell v6 实例。主机防火墙、云安全组和 NAT 映射都需要放行整个 TCP 端口范围。第一个端口用于协议探测，每份有效成员凭据独占一个端口和 PSK，计划轮换期间的新旧凭据各占一个。主机完成带认证的探测后，订阅才会发布成员端口。TCP 和 UDP 流量均通过 TCP 监听端口转发，并沿用原有配额账本与出站策略。

自建节点支持 `default` 和 `unshaped` 两种直连模式。可导出 Surge、Mihomo、sing-box 和 v2rayN 配置，其中 v2rayN 使用 sing-box 自定义出站。Mihomo 和 sing-box 已通过实际流量测试，Surge Mac 已通过配置校验。原生 Surge 实流量和 Shadowrocket 真机互通仍待验证。含 userkey 的外部节点可导出到 sing-box 和 v2rayN。外部来源还可显式使用 `unsafe-raw`，该模式以明文传输应用数据。

### 远程 CLI

在已校验的下载目录中执行：

```sh
sh ./install-ctl.sh --version 0.0.28 --install-dir "$HOME/.local/bin"
export PATH="$HOME/.local/bin:$PATH"
conflux-ctl version

# 在控制台「账户与安全 → API 访问令牌」选择所需权限，创建并下载令牌。
conflux-ctl config set --server https://conflux.example.com --token-file "$HOME/Downloads/conflux-api-token.txt" --profile default
conflux-ctl whoami
conflux-ctl status
conflux-ctl ops alerts --me
```

集群管理员可通过 `conflux-ctl ops tasks` 和 `conflux-ctl ops alerts` 查看集群任务和告警。运行 `conflux-ctl skill rules` 可查看如何绑定订阅规则、调整顺序，并预览六种客户端的结果。

原生控制端和 Agent 安装器会验证二进制 manifest 的 Ed25519 签名和 SHA-256 摘要。从 GitHub Releases 下载时，CLI 安装器需要支持 Ed25519 的 OpenSSL，先验证 `checksums.txt` 的签名，再核对产物的 SHA-256 摘要。产物由控制端提供时，安装器在核对 `X-Artifact-SHA256` 响应头摘要的同时还会验证 `.manifest.json` 伴随清单的 Ed25519 签名（不可用时自动回退至经签名的 GitHub Releases 产物）。

### Docker Compose

将 [Compose 模板](deploy/docker-compose.yml)下载到部署目录：

```sh
mkdir -p conflux && cd conflux
curl -fsSL https://raw.githubusercontent.com/veildawn/conflux-releases/main/deploy/docker-compose.yml -o docker-compose.yml
```

在同目录创建权限为 `0600` 的 `.env`。以下两个密钥占位值应分别使用 `openssl rand -hex 32` 生成一次并保存：

```dotenv
CONFLUX_PUBLIC_BASE_URL=https://conflux.example.com
CONFLUX_MASTER_KEY_HEX=REPLACE_WITH_PERSISTENT_64_HEX_CHARACTERS
POSTGRES_PASSWORD=REPLACE_WITH_A_RANDOM_PASSWORD
CONFLUX_SERVER_IMAGE=ghcr.io/veildawn/conflux-release:0.0.28
```

```sh
chmod 600 .env
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=50 conflux-server
```

默认绑定 `127.0.0.1:8080`，需要配置 HTTPS/WebSocket 反向代理。PostgreSQL 数据位于 `pgdata` 卷，服务端状态位于挂载到 `/var/lib/conflux` 的 `conflux-data` 卷。备份数据卷和 `.env`，每次重启复用同一主密钥。使用服务端日志中的令牌完成首次初始化。

原生和 Compose 部署均支持通过侧边栏版本徽章执行应用内升级。升级会验证已签名的校验清单和二进制摘要；Docker 模式将替换二进制保存在持久化数据卷中，无需挂载 `/var/run/docker.sock`。

## 使用手动解析的域名

在**域名与证书**页面点击**使用已解析域名**，选择目标主机并填写已指向该主机的域名。无需先接入 Cloudflare 凭据或创建托管 DNS 绑定。A / AAAA 记录仍由你维护；证书会出现在 **TLS 证书**列表中，并自动续期。

验证方式保留**自动选择（推荐）**。服务器仅对匹配的 Cloudflare 托管域名使用 DNS-01，其他情况按实际端口选择 HTTP-01 或 TLS-ALPN-01。请确保所有 A / AAAA 记录均指向目标主机，且所选验证方式需要的端口可从公网访问：HTTP-01 使用 TCP 80，TLS-ALPN-01 使用 TCP 443。对于受支持的原生 Nginx、Apache 和 Caddy 部署，Agent 会在保留现有网站的情况下自动准备 HTTP-01 验证路径、检查变更并在验证后清理。Caddy 需要启用本地管理 API。容器化服务、自定义配置路径及不支持的监听服务仍可能需要管理员处理；控制台会说明无法自动处理的情况。

## 在已有网站的主机上创建协议

自 v0.0.7 起，新建协议表单默认使用该主机最近绑定的域名，包括证书仍在申请中的手动解析域名，也允许自行修改连接地址。TLS 协议仍需该地址对应的有效证书；REALITY 使用独立验证的握手目标与 SNI。

升级控制端及节点组件后，可在协议表单中检查部署环境，查看 DNS、TCP/UDP 监听状态及 REALITY TLS 兼容性。创建时会再次检查环境。手动指定的端口与其他受管实例冲突时，系统会拒绝创建。使用自动选端口时，系统可选择空闲候选端口，保留现有网站服务。控制台会展示检测到的冲突和调整后的连接地址。主机本地端口空闲仍需防火墙或 NAT 放行，创建后的探测会检查实际可达性。旧版 Agent 需要升级才能提供这些检查。

## 用户网络探测

v0.0.4 及后续版本会在管理控制台的节点页可见时自动检测当前网络。请一并升级控制端、节点 Agent 和 Runtime。在远程节点详情中可以看到实际检测的入口和各传输协议的结果。点击 **重新检测** 可同时刷新浏览器和服务端的检测结果。

浏览器通过 WebRTC 检测节点的业务端口。VLESS + REALITY 和 Trojan 检测 TCP，Hysteria2 检测 UDP，Shadowsocks 2022 按实例启用的传输协议检测。启用端口跳跃时，结果仅覆盖页面显示的一个入口。节点收到经认证的回应后，对应协议显示「可达」。10 秒内未收到回执则显示「超时，未确认」。结果 3 分钟内有效。关闭页面后，服务端仍会定时检查节点健康。

已发布的 v0.0.3 使用随版本提供的工具，操作见 [v0.0.3 用户网络探测说明](https://github.com/veildawn/conflux-releases/blob/4d899cc537b925014a043e8e6e54fcbc9729ef86/README.zh.md#用户网络探测)。

## 发布产物与运行时源码

每个版本包含：

- 五个组件的 Linux 双架构独立二进制，macOS 双架构 CLI 与监控模式 Agent，以及 Windows 双架构监控模式 Agent。
- `conflux-server_<version>_linux_<arch>.tar.gz`，包含供自更新使用的 `conflux-server` 和 `conflux-ctl`。
- `install-server.sh`、`install-agent.sh`、`install-agent.ps1`、`install-ctl.sh` 与 `release-pubkey.pem`。
- `checksums.txt`、其 base64 编码的 Ed25519 签名 `checksums.txt.sig`，以及内容相同的兼容文件 `SHA256SUMS.txt`。校验清单覆盖二进制、压缩包、安装器和公钥。
- 十份 Linux 组件 manifest、一份镜像摘要 manifest 及汇总 `manifests.json`，使用 `conflux-release-manifest-v1` 域签名。macOS 与 Windows 二进制由已签名的校验清单覆盖。
- 与 runtime 二进制来自同一源码提交的 `conflux-runtime-<version>-source.tar.gz`（整体 `GPL-3.0-or-later`，含 `Xray-core` `MPL-2.0` 与内置 Snell v6 `GPL-3.0-or-later`），包含运行时模块的 README、许可证声明、适配清单及固定依赖。参见 [LICENSE](LICENSE)。

如需构建发布的运行时源码，解压源码包，使用其中 `go.mod` 声明的 Go 版本，再执行 `go test ./...` 和 `go build ./cmd/conflux-runtime`。本公开仓库本身不包含应用源码树。

## 维护与发布流程

| 流水线 | 触发方式与用途 |
| --- | --- |
| [Verify Release Repo](.github/workflows/ci.yml) | 公开仓库 `main` 分支推送及 PR；检查公钥和 Compose 文件 |
| [Source CI](.github/workflows/source-ci.yml) | 手动指定 `source_ref`，或 `source-ci` 类型的 `repository_dispatch`；验证私有源码、前端、两个 Go 模块的竞态测试，并交叉构建 Linux、macOS 与 Windows 组件 |
| [Release](.github/workflows/release.yml) | 手动指定版本、源码和通道，可复用工作流调用，或公开仓库 `v*` 标签推送；构建、签名并发布产物与镜像 |
| [Release request](.github/workflows/release-request.yml) | 推送变更 `.github/release-requests/*.json`；调用可复用发布流水线 |

仅推送源码不会自动启动 Source CI。在有权限的**私有源码仓库工作目录**中，提交并推送后执行：

```sh
SOURCE_SHA="$(git rev-parse HEAD)"
gh workflow run source-ci.yml --repo veildawn/conflux-releases -f source_ref="$SOURCE_SHA"
gh run list --repo veildawn/conflux-releases --workflow source-ci.yml --limit 5
```

该提交通过 CI 后，选择尚未使用的版本，在源码仓库创建对应标签，再触发公开仓库发布：

```sh
RELEASE_TAG=vX.Y.Z  # 替换为下一个尚未使用的版本。
git tag -a "$RELEASE_TAG" "$SOURCE_SHA" -m "Conflux $RELEASE_TAG"
git push origin "$RELEASE_TAG"
gh workflow run release.yml --repo veildawn/conflux-releases \
  -f version="$RELEASE_TAG" -f source_ref="$SOURCE_SHA" -f channel=stable
```

预发布使用 `channel=prerelease`，可选填写 `changelog`。发布流水线会将源码引用解析成完整 SHA，建议直接传入完整 SHA。源码标签标识应用提交，公开发布标签属于本发行仓库，两者的提交 SHA 不同。通过公开仓库标签触发构建时，私有源码仓库必须已有同名标签。保留已发布版本，修复应使用新版本号。

必需的 Actions secrets 为 `SOURCE_REPO_SSH_KEY`（私有源码仓库的只读 deploy key）、`RELEASE_PUBLIC_KEY`（base64 编码的 32 字节 Ed25519 公钥）和 `RELEASE_SIGNING_KEY`（base64 编码的 64 字节 Ed25519 私钥）。发布流水线还支持以 `SOURCE_REPO_TOKEN` 替代源码访问方式；Source CI 使用 SSH key。镜像发布使用具有 `packages: write` 权限的工作流 `GITHUB_TOKEN`。仓库公钥、安装器公钥和签名身份必须保持一致。

宣告发布完成前，应确认公开 Actions 成功、发布文件齐全、签名与 SHA-256 正确、运行时源码与二进制对应，并验证两个架构的镜像均可匿名访问。维护工作流或文档时遵循 [AGENTS.md](AGENTS.md)，并同步更新本文件与 [README.md](README.md)。
