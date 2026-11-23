# Conflux Releases

[简体中文](README.zh.md)

Public downloads, deployment templates, CI, and releases for **Conflux**, a multi-tenant proxy node management platform with four built-in protocols and optional Snell v6, external subscription sharing, six-client subscription compilation, conserved traffic quotas (`L = U + R + A`), DNS/ACME automation, and a bilingual console.

The current stable release is [v0.0.19](https://github.com/veildawn/conflux-releases/releases/tag/v0.0.19). See [all releases](https://github.com/veildawn/conflux-releases/releases) for release history. Application source is maintained in the private `veildawn/conflux-service` repository; all GitHub Actions run in this public repository. Actions are disabled in the private repository.

Clients connect to the selected node, which forwards traffic to the destination. The control server distributes subscriptions, configuration, and quota grants; ordinary proxy payload does not pass through it. Explicit node chaining adds the configured relay hops. The built-in protocols are VLESS + REALITY, Hysteria2, Trojan, and Shadowsocks 2022. Linux hosts can also enable direct Snell v6. External subscriptions also support TUIC.

## Downloads

| Artifact | Platforms |
| --- | --- |
| `conflux-server`, `conflux-agent`, `conflux-runtime`, `conflux-updater`, `conflux-ctl` | Linux `amd64` / `arm64` |
| `conflux-ctl` | macOS (`darwin`) `amd64` / `arm64` |
| `ghcr.io/veildawn/conflux-release:0.0.19` | Linux `amd64` / `arm64` container image |
| `conflux-runtime-0.0.19-source.tar.gz` | Source for the MPL-2.0 runtime embedding `Xray-core 26.11.9`, dependency lockfiles, build instructions, and license notice |

The [GHCR image](https://github.com/veildawn/conflux-releases/pkgs/container/conflux-release) is public and can be pulled without signing in. The package name is **`conflux-release`** (singular); this repository is **`conflux-releases`** (plural). Pin a version or digest for a reproducible deployment; `latest` is a mutable tag.

## Installation

The examples below install **0.0.19** explicitly. Native server and Agent installers support Debian 12/13 and Ubuntu 22.04/24.04/26.04 with systemd, on `amd64` or `arm64`. They require root, local storage, `curl`, OpenSSL with Ed25519 support, `sha256sum`, and Python 3. The standalone CLI supports Linux and macOS.

### Download and Verify Installers

Run in a new working directory. Download the checksum signature and installers from the same release, then verify them before execution:

```sh
set -eu
RELEASE_VERSION=0.0.19
RELEASE_URL="https://github.com/veildawn/conflux-releases/releases/download/v${RELEASE_VERSION}"
for file in release-pubkey.pem checksums.txt checksums.txt.sig install-server.sh install-agent.sh install-ctl.sh; do
  curl -fsSL "$RELEASE_URL/$file" -o "$file"
done

# SHA-256 fingerprint of the DER-encoded SPKI public key.
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

The fingerprint above and [release-pubkey.pem](release-pubkey.pem) are the published trust reference. Keep a trusted copy for future verification; fetching a key and signature from the same download alone does not establish trust.

### Linux Control Server

From the verified download directory, supply an existing PostgreSQL database and your external HTTPS URL:

```sh
sudo bash ./install-server.sh --version 0.0.19 \
  --public-base-url https://conflux.example.com \
  --database-url 'postgres://conflux:REPLACE_WITH_PASSWORD@127.0.0.1:5432/conflux?sslmode=disable' \
  --release-pubkey-pem "$PWD/release-pubkey.pem"
sudo systemctl status conflux-server
sudo journalctl -u conflux-server -n 50 --no-pager
```

Native installation listens on `127.0.0.1:8080`. Configure an HTTPS reverse proxy with WebSocket support for the public URL. On a fresh database, the server log contains a single-use setup token valid for 30 minutes; open the console, complete setup, and enroll administrator MFA. Preserve `/etc/conflux` (including master/signing keys), `/var/lib/conflux`, and PostgreSQL backups.

### Managed Host Agent and Runtime

Create a 15-minute enrollment token in **Hosts → Connect VPS**. Save it to a root-readable `0600` file on the target host, then run the verified installer there:

```sh
sudo bash ./install-agent.sh --version 0.0.19 \
  --server https://conflux.example.com \
  --enroll-token-file /root/conflux-enroll-token \
  --release-pubkey-pem "$PWD/release-pubkey.pem"
sudo systemctl status conflux-agent conflux-runtime
```

Keep the SQLite WAL ledger on local storage. Installers change firewall rules only when `--configure-firewall` is requested; open the configured proxy ports in host and cloud firewalls as needed.

### Snell v6

On Linux amd64 and arm64, the agent installer accepts `--with-snell-v6`. Hosts running `v0.0.18` or later can also enable or uninstall the component online without SSH via **Hosts** (single-host drawer or multi-select batch actions), **Nodes** (inline enable CTA during creation), or `conflux-ctl host component`. Hosts download the upstream `sing-box 1.14.4` release directly over HTTPS and verify its archive and executable SHA-256 digests without relaying through the control plane, while `conflux-runtime` activates or removes `proto_snell_v6_v1` live without restarting active Xray tunnels. Existing hosts on `v0.0.17` or earlier unlock online component management after a standard online upgrade to `v0.0.18`, or can still install manually via:

```sh
sudo /usr/local/bin/conflux-runtime snell-component --install
```

In **Nodes**, create a Snell v6 instance on a host reporting the component. Allow its entire TCP port range in the host firewall, cloud firewall and any port mapping. The first port serves protocol probes. Each active member credential gets its own port and PSK, including both credentials during a planned rotation. Subscriptions publish a member port after an authenticated host check. The service forwards TCP and UDP over the TCP listener and uses the existing quota ledger and outbound policy.

Managed nodes support direct connections in `default` and `unshaped` modes. Subscriptions can export Surge, Mihomo, sing-box and v2rayN configurations. v2rayN uses a custom sing-box outbound. Traffic forwarding was tested with Mihomo and sing-box. Surge Mac passed configuration validation. Native Surge traffic and Shadowrocket device compatibility are pending verification. External sources with a userkey export to sing-box and v2rayN. Only external sources can use `unsafe-raw`, which sends application data in plaintext.

### Remote CLI

From the verified download directory:

```sh
sh ./install-ctl.sh --version 0.0.19 --install-dir "$HOME/.local/bin"
export PATH="$HOME/.local/bin:$PATH"
conflux-ctl version

# In Account & Security → API Access Tokens, create a scoped token and download it.
conflux-ctl config set --server https://conflux.example.com --token-file "$HOME/Downloads/conflux-api-token.txt" --profile default
conflux-ctl whoami
conflux-ctl status
conflux-ctl ops alerts --me
```

Cluster administrators can use `conflux-ctl ops tasks` and `conflux-ctl ops alerts` for cluster-wide activity. Run `conflux-ctl skill rules` for instructions on binding and ordering subscription rules and previewing changes across all six clients.

The native server and Agent installers verify the Ed25519 signatures and SHA-256 digests in the binary manifests. When downloading from GitHub Releases, the CLI installer requires OpenSSL with Ed25519 support and verifies the signature on `checksums.txt` before checking the artifact's SHA-256 digest. For artifacts served by a control plane, it checks the SHA-256 digest against the `X-Artifact-SHA256` response header.

### Docker Compose

Download the [Compose template](deploy/docker-compose.yml) into a deployment directory:

```sh
mkdir -p conflux && cd conflux
curl -fsSL https://raw.githubusercontent.com/veildawn/conflux-releases/main/deploy/docker-compose.yml -o docker-compose.yml
```

Create a `0600` `.env` file alongside it, replacing both secret placeholders with separate values generated once by `openssl rand -hex 32`:

```dotenv
CONFLUX_PUBLIC_BASE_URL=https://conflux.example.com
CONFLUX_MASTER_KEY_HEX=REPLACE_WITH_PERSISTENT_64_HEX_CHARACTERS
POSTGRES_PASSWORD=REPLACE_WITH_A_RANDOM_PASSWORD
CONFLUX_SERVER_IMAGE=ghcr.io/veildawn/conflux-release:0.0.19
```

```sh
chmod 600 .env
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=50 conflux-server
```

The default binding is `127.0.0.1:8080`; configure an HTTPS/WebSocket reverse proxy. PostgreSQL uses the `pgdata` volume and server state uses `conflux-data` at `/var/lib/conflux`. Back up these volumes and `.env`, and reuse the master key on every restart. Complete first-run setup using the token in the server log.

Native and Compose deployments support in-app updates through the sidebar version badge. Updates verify signed checksums and binary digests; Docker stores the replacement binary in its persistent data volume and does not require mounting `/var/run/docker.sock`.

## Domains with manually configured DNS

In **Domains & Certificates**, choose **Use existing domain**, select the target host, and enter the domain that already points to it. Cloudflare credentials and managed DNS bindings are optional. You continue to maintain the A / AAAA records; the certificate appears under **TLS certificates** and renews automatically.

Leave validation on **Automatic (recommended)**. The server uses DNS-01 only for a matching Cloudflare-managed domain; otherwise it selects HTTP-01 or TLS-ALPN-01 based on available ports. Ensure every A / AAAA record points to the target host and that the selected validation port is publicly reachable: TCP 80 for HTTP-01 or TCP 443 for TLS-ALPN-01. For supported native Nginx, Apache, and Caddy deployments, the Agent automatically prepares the HTTP-01 challenge route while preserving the website, validates the change, and cleans it up afterward. Caddy requires its local admin API. Containerized services, custom configuration locations, and unsupported listeners may still require operator intervention; the console reports when automatic handling is unavailable.

## Creating protocols on hosts with existing websites

Since v0.0.7, the new-protocol form defaults to the most recently attached host domain, including manually configured domains whose certificate is still being issued. You can override the connection address. TLS protocols still require a valid certificate for that address; REALITY uses a separately validated handshake target and SNI.

After upgrading the control plane and node components, use the deployment-environment check in the protocol form to inspect DNS, TCP/UDP listeners, and REALITY TLS compatibility. Creation checks the environment again. An explicit port that overlaps another managed instance is rejected. Automatic port selection can choose a free candidate and preserve existing web services. The console shows the detected conflict and adjusted endpoint. A free local port still needs to be reachable through your firewall or NAT; post-creation probes report reachability. Older Agents must be upgraded to provide these checks.

## User network probes

Conflux v0.0.4 and later check your current network automatically while the admin console's Nodes page is visible. Upgrade the control plane, node agent and runtime together. Open a remote node to see its endpoint and separate TCP or UDP results. **Check again** starts both the browser and server checks.

The browser uses WebRTC on the node's configured service port. VLESS + REALITY and Trojan use TCP, Hysteria2 uses UDP, and Shadowsocks 2022 uses the transports enabled on the instance. With port hopping, the result covers the one entrance shown. A transport is reachable when the node receives its authenticated response. After ten seconds without a receipt, that transport is unconfirmed. Results stay fresh for three minutes. Scheduled server health checks continue when the console is closed.

For the published v0.0.3 release, follow the [v0.0.3 device probe instructions](https://github.com/veildawn/conflux-releases/blob/4d899cc537b925014a043e8e6e54fcbc9729ef86/README.md#user-network-probes).

## Release Contents and Runtime Source

Each release includes:

- Standalone Linux binaries for all five components and macOS CLI binaries.
- `conflux-server_<version>_linux_<arch>.tar.gz` containing `conflux-server` and `conflux-ctl` for self-update.
- `install-server.sh`, `install-agent.sh`, `install-ctl.sh`, and `release-pubkey.pem`.
- `conflux-snell-component-<version>-linux-<arch>.json` with the upstream version and SHA-256 pins for the optional Snell component on each Linux architecture. The signed checksums cover both files.
- Signed `checksums.txt`, its base64 Ed25519 signature `checksums.txt.sig`, and the identical `SHA256SUMS.txt` compatibility alias. The checksum list covers binaries, archives, installers, and the public key.
- Ten Linux component manifests, a container-image digest manifest, and combined `manifests.json`, signed with the `conflux-release-manifest-v1` domain. macOS CLI assets are covered by signed checksums.
- `conflux-runtime-<version>-source.tar.gz` from the same source commit as the runtime binary. It contains the runtime module's README, license notice, and pinned dependencies. See [LICENSE](LICENSE).

To build the published runtime source, extract its archive, use the Go version declared in its `go.mod`, and run `go test ./...` and `go build ./cmd/conflux-runtime`. The public repository itself does not contain the application source tree.

## Maintainer Workflow

| Workflow | Trigger and purpose |
| --- | --- |
| [Verify Release Repo](.github/workflows/ci.yml) | Public `main` pushes and pull requests; checks the public key and Compose file |
| [Source CI](.github/workflows/source-ci.yml) | Manual `source_ref` or `repository_dispatch` type `source-ci`; verifies private source, frontend, both Go modules with race detection, and cross-builds the Linux components |
| [Release](.github/workflows/release.yml) | Manual version/source/channel inputs, reusable workflow call, or public `v*` tag push; builds, signs, and publishes artifacts and the image |
| [Release request](.github/workflows/release-request.yml) | Push changing `.github/release-requests/*.json`; invokes the reusable release workflow |

Source pushes alone do not start Source CI. From an authorized **private source checkout**, after committing and pushing:

```sh
SOURCE_SHA="$(git rev-parse HEAD)"
gh workflow run source-ci.yml --repo veildawn/conflux-releases -f source_ref="$SOURCE_SHA"
gh run list --repo veildawn/conflux-releases --workflow source-ci.yml --limit 5
```

After that SHA passes CI, use the next unused version, create its tag in the source repository, and dispatch the public release:

```sh
RELEASE_TAG=vX.Y.Z  # Replace with the next unused version.
git tag -a "$RELEASE_TAG" "$SOURCE_SHA" -m "Conflux $RELEASE_TAG"
git push origin "$RELEASE_TAG"
gh workflow run release.yml --repo veildawn/conflux-releases \
  -f version="$RELEASE_TAG" -f source_ref="$SOURCE_SHA" -f channel=stable
```

Use `channel=prerelease` for prereleases and optionally supply `changelog`. The release workflow resolves the source ref to a full SHA; prefer supplying that SHA directly. The source tag identifies the application commit, while the public release tag belongs to this distribution repository. A public tag-triggered build requires the matching tag in the private source repository. Keep existing versions intact and ship fixes under a new version.

Required Actions secrets are `SOURCE_REPO_SSH_KEY` (read-only deploy key for the private source repository), `RELEASE_PUBLIC_KEY` (base64 32-byte Ed25519 public key), and `RELEASE_SIGNING_KEY` (base64 64-byte Ed25519 private key). The release workflow also supports `SOURCE_REPO_TOKEN` as an alternative; Source CI uses the SSH key. Package publishing uses the workflow's `GITHUB_TOKEN` with `packages: write`. Keep the checked-in public key, installer key, and signing identity aligned.

Before declaring publication complete, check the public Actions result, release assets, signatures and SHA-256 digests, runtime source correspondence, and anonymous image access for both supported architectures. Follow [AGENTS.md](AGENTS.md) when maintaining workflows or documentation, and keep this README and [README.zh.md](README.zh.md) synchronized.
