# Conflux Releases

[简体中文](README.zh.md)

Public release distribution repository for **Conflux** — a private multi-tenant proxy node management platform combining self-hosted multi-protocol dataplane orchestration (`VLESS+REALITY`, `Hysteria2`, `Trojan`, `Shadowsocks 2022`, `TUIC v5`), external subscription aggregation and sharing, deterministic 6-client compilation, conserved member traffic quota accounting (`L = U + R + A`), Cloudflare DNS & ACME automation, and an embedded bilingual console.

## Quick Install

### Linux Control Server (systemd, `amd64` / `arm64`)

```sh
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/release-pubkey.pem -o /tmp/conflux-release-pubkey.pem
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/install-server.sh \
  | sudo bash -s -- \
    --public-base-url https://conflux.example.com \
    --database-url 'postgres://conflux:password@127.0.0.1:5432/conflux?sslmode=disable' \
    --release-pubkey-pem /tmp/conflux-release-pubkey.pem
```

### Managed Host Agent + Runtime (systemd, `amd64` / `arm64`)

Generate an enrollment token in the Admin Console (`Hosts -> Connect VPS`), then run on the target VPS:

```sh
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/release-pubkey.pem -o /tmp/conflux-release-pubkey.pem
curl -fsSL https://github.com/veildawn/conflux-releases/releases/latest/download/install-agent.sh \
  | sudo bash -s -- \
    --server https://conflux.example.com \
    --enroll-token '<15m-enroll-token>' \
    --release-pubkey-pem /tmp/conflux-release-pubkey.pem
```

### Docker Compose (`amd64` / `arm64`)

```sh
mkdir -p conflux && cd conflux
curl -fsSL https://raw.githubusercontent.com/veildawn/conflux-releases/main/deploy/docker-compose.yml -o docker-compose.yml
```

Create `.env` with `CONFLUX_PUBLIC_BASE_URL`, `CONFLUX_MASTER_KEY_HEX` (`openssl rand -hex 32`), and `POSTGRES_PASSWORD`, then start:

```sh
docker compose up -d
```

Both native `systemd` and Docker Compose deployments support **one-click in-app updates** from the sidebar version badge (`conflux-server` downloads, verifies the Ed25519 signature + SHA-256 digest, stages the verified binary onto the persistent `/var/lib/conflux` volume under Docker without requiring `/var/run/docker.sock`, and gracefully restarts).

## Release Verification & GPL-3.0 Compliance

Every release publishes:
- `conflux-server_<version>_linux_<arch>.tar.gz` (in-app self-update archive)
- Standalone verified binaries (`conflux-server`, `conflux-agent`, `conflux-runtime`, `conflux-updater`, `conflux-ctl`)
- `checksums.txt` + `checksums.txt.sig` (Ed25519 signature verified by `release-pubkey.pem`)
- Per-component `*.manifest.json` + combined `manifests.json` (Ed25519 domain-signed `conflux-release-manifest-v1`)
- `conflux-runtime-<version>-source.tar.gz` (complete GPL-3.0-or-later corresponding source for `conflux-runtime`)
