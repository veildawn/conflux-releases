# Conflux Releases

[简体中文](README.zh.md)

Public downloads, deployment templates, CI, and releases for **Conflux**, a multi-tenant proxy node management platform with five managed protocols, external subscription sharing, six-client subscription compilation, conserved traffic quotas (`L = U + R + A`), DNS/ACME automation, and a bilingual console.

The first stable release is [v0.0.1](https://github.com/veildawn/conflux-releases/releases/tag/v0.0.1). See [all releases](https://github.com/veildawn/conflux-releases/releases) for subsequent versions. Application source is maintained in the private `veildawn/conflux-service` repository; all GitHub Actions run in this public repository. Actions are disabled in the private repository.

## Downloads

| Artifact | Platforms |
| --- | --- |
| `conflux-server`, `conflux-agent`, `conflux-runtime`, `conflux-updater`, `conflux-ctl` | Linux `amd64` / `arm64` |
| `conflux-ctl` | macOS (`darwin`) `amd64` / `arm64` |
| `ghcr.io/veildawn/conflux-release:0.0.1` | Linux `amd64` / `arm64` container image |
| `conflux-runtime-0.0.1-source.tar.gz` | Corresponding runtime source, dependency lockfiles, build instructions, and license notice |

The [GHCR image](https://github.com/veildawn/conflux-releases/pkgs/container/conflux-release) is public and can be pulled without signing in. The package name is **`conflux-release`** (singular); this repository is **`conflux-releases`** (plural). Pin a version or digest for a reproducible deployment; `latest` is a mutable tag.

## Installation

The examples below install **0.0.1** explicitly. Native server and Agent installers support Debian 12/13 and Ubuntu 22.04/24.04/26.04 with systemd, on `amd64` or `arm64`. They require root, local storage, `curl`, OpenSSL with Ed25519 support, `sha256sum`, and Python 3. The standalone CLI supports Linux and macOS.

### Download and Verify Installers

Run in a new working directory. Download the checksum signature and installers from the same release, then verify them before execution:

```sh
set -eu
RELEASE_VERSION=0.0.1
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
sudo bash ./install-server.sh --version 0.0.1 \
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
sudo bash ./install-agent.sh --version 0.0.1 \
  --server https://conflux.example.com \
  --enroll-token-file /root/conflux-enroll-token \
  --release-pubkey-pem "$PWD/release-pubkey.pem"
sudo systemctl status conflux-agent conflux-runtime
```

Keep the SQLite WAL ledger on local storage. Installers change firewall rules only when `--configure-firewall` is requested; open the configured proxy ports in host and cloud firewalls as needed.

### Remote CLI

From the verified download directory:

```sh
sh ./install-ctl.sh --version 0.0.1 --install-dir "$HOME/.local/bin"
export PATH="$HOME/.local/bin:$PATH"
conflux-ctl version

# Create a scoped token in Console -> Account & Security -> API Access Tokens.
conflux-ctl config set --server https://conflux.example.com --token cfx_pat_... --profile default
conflux-ctl whoami
conflux-ctl status
conflux-ctl ops tasks
conflux-ctl ops alerts
```

The native server/Agent installers verify binary manifests with Ed25519 and SHA-256. The standalone CLI installer checks available SHA-256 metadata; it does not verify the checksum signature itself. For authenticated manual binary verification, download the CLI asset from the same release and compare its SHA-256 against the signed `checksums.txt` verified above.

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
CONFLUX_SERVER_IMAGE=ghcr.io/veildawn/conflux-release:0.0.1
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

## User network probes

In releases that include `conflux-probe`, download the asset for your device from the same release as your control plane. The asset is named `conflux-probe-<version>-<os>-<arch>`, with an `.exe` suffix on Windows. The operating systems are `linux`, `darwin`, and `windows`, each for `amd64` and `arm64`. Verify `checksums.txt` from that release as above. Compare its entry for your asset with the hash from `sha256sum` (Linux), `shasum -a 256` (macOS), or `Get-FileHash -Algorithm SHA256` (Windows).

Open a remote node in the Conflux admin console and select **Probe from user device**. On the device and network you want to test, run the tool with your own sing-box subscription export.

```sh
chmod +x ./conflux-probe-0.0.0-darwin-arm64
./conflux-probe-0.0.0-darwin-arm64 --server https://conflux.example.com --config ./sing-box.json
```

```powershell
.\conflux-probe-0.0.0-windows-amd64.exe --server https://conflux.example.com --config .\sing-box.json
```

Replace the example version and filename with the asset you downloaded. Paste the console's pairing code when prompted. The code expires after five minutes. If the export contains multiple identical matches, specify the outbound tag with `--node-tag`. The export and its credentials stay on the device. Conflux shows the returned result alongside the control-plane result.

## Release Contents and Runtime Source

Each release includes:

- Standalone Linux binaries for all five components and macOS CLI binaries.
- `conflux-server_<version>_linux_<arch>.tar.gz` containing `conflux-server` and `conflux-ctl` for self-update.
- `install-server.sh`, `install-agent.sh`, `install-ctl.sh`, and `release-pubkey.pem`.
- Signed `checksums.txt`, its base64 Ed25519 signature `checksums.txt.sig`, and the identical `SHA256SUMS.txt` compatibility alias. The checksum list covers binaries, archives, installers, and the public key.
- Ten Linux component manifests, a container-image digest manifest, and combined `manifests.json`, signed with the `conflux-release-manifest-v1` domain. macOS CLI and user network probe assets are covered by signed checksums.
- `conflux-runtime-<version>-source.tar.gz` from the same source commit as the runtime binary. It contains the runtime module's README, license notice, and pinned dependencies. See [LICENSE](LICENSE).

To build the published runtime source, extract its archive, use the Go version declared in its `go.mod`, and run `go test ./...` and `go build ./cmd/conflux-runtime`. The public repository itself does not contain the application source tree.

## Maintainer Workflow

| Workflow | Trigger and purpose |
| --- | --- |
| [Verify Release Repo](.github/workflows/ci.yml) | Public `main` pushes and pull requests; checks the public key and Compose file |
| [Source CI](.github/workflows/source-ci.yml) | Manual `source_ref` or `repository_dispatch` type `source-ci`; verifies private source, frontend, both Go modules with race detection, and cross-builds the Linux components and the user network probe |
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
