# Conflux Releases — Agent Instructions

## Repository Scope

- This public `veildawn/conflux-releases` repository owns distribution documentation, the Compose template, the release public key, and **all** Conflux GitHub Actions.
- Application code, tests, installers, the Dockerfile, and the release signer live in the private `veildawn/conflux-service` repository. Actions are disabled there; do not enable private Actions or move workflows back into it.
- Do not copy the private application source into this repository. Public source distribution is limited to the corresponding `runtime/` source archive attached to each release. A source commit SHA in a manifest is provenance, not permission to expose private code.
- Keep the standard instruction filename `AGENTS.md`. Keep [README.md](README.md) and [README.zh.md](README.zh.md) synchronized, using repository-relative links and commands checked against actual installers/workflows.

## Workflow Map

- [ci.yml](.github/workflows/ci.yml): automatic public `main` push/PR checks for the release public key and Compose file.
- [source-ci.yml](.github/workflows/source-ci.yml): manual `source_ref` or `repository_dispatch` type `source-ci`; reads private source and runs the license boundary check, UI consistency, frontend checks/build, race-enabled tests in both Go modules, and Linux/macOS/Windows cross-builds.
- [release.yml](.github/workflows/release.yml): manual/reusable release, or public `v*` tag push; resolves private source to a SHA, builds binaries/source archives/images, signs, and publishes. Manual inputs are `version`, `source_ref`, `channel`, and optional `changelog`. `since_sha` is a reusable-workflow input, not a manual dispatch input.
- [release-request.yml](.github/workflows/release-request.yml): a push changing `.github/release-requests/*.json` invokes `release.yml`. Requests accept `version`, `source_ref`, `channel`, optional `changelog` and `since_sha`; submit one request per push and use a full source SHA.

Private source pushes do not automatically start public CI. From the private source checkout, after committing and pushing:

```sh
gh workflow run source-ci.yml --repo veildawn/conflux-releases \
  -f source_ref="$(git rev-parse HEAD)"
```

Use the **private repository's** SHA, never the current public repository's HEAD. Inspect the matching public run before reporting validation success. Documentation-only changes do not require a new release.

## Private Source and Credentials on Public Runners

- `SOURCE_REPO_SSH_KEY` is a read-only deploy key scoped to the private source repository. Source CI requires it; the release workflow also supports `SOURCE_REPO_TOKEN` as an alternative. Avoid broad personal credentials.
- Preserve `persist-credentials: false`, pinned source commits across release jobs, private-checkout cleanup, and disabled Go caches for private builds. Never upload private checkouts, raw test logs, build contexts, or private source caches as public artifacts.
- Public logs must not expose secrets, source contents, sensitive fixtures, or unfiltered test output. Preserve failure-name summaries and sanitization when changing test commands; review shell tracing and compiler output as well.
- Never run untrusted pull-request code with source-read or release-signing secrets. Keep public PR validation separate from privileged dispatch/release jobs.
- Preserve minimal job permissions. The image job uses `packages: write`; publishing release assets needs `contents: write`. Do not enable additional credentials for documentation changes.
- Keep third-party Action references pinned to reviewed versions; when changing a SHA pin, verify that it resolves to the intended upstream version.

## Build and Release Contract

- Use Go from the source `go.mod`, Node.js 22, and pnpm 10.6.5 with the frozen frontend lockfile. PostgreSQL integration tests require `initdb` and `pg_ctl` on `PATH`; expose `pg_config --bindir`. Bound Go package test parallelism to four.
- Preserve the GPL module/process boundary: root `cmd/` and `internal/` must not import `conflux-runtime` or `sagernet/sing-box`. Run the source repository's `scripts/release/verify-license-boundary.sh` before publishing.
- Release a new version only from a pushed source SHA that passed public Source CI. Create the private source version tag at the exact commit being built. The public release tag belongs to this distribution repository and has a different commit SHA.
- Keep published tags and assets intact; ship corrections as a new version. A public `v*` tag push starts a release and requires the same source tag to exist; prefer explicit manual dispatch with a full source SHA.
- Preserve Linux `amd64`/`arm64` binaries for server, Agent, runtime, updater, and CLI; macOS `amd64`/`arm64` CLI and monitor-only Agent binaries; Windows `amd64`/`arm64` monitor-only Agent binaries; self-update archives containing server and CLI; all installers (`install-server.sh`, `install-agent.sh`, `install-agent.ps1`, `install-ctl.sh`); signed checksums; signed manifests; the runtime source archive; and the multi-architecture container image.
- Runtime source must come from the same source SHA as the runtime binary and include its dependency pins, build instructions, and license notice. Do not package the root/private application tree as the runtime source archive.
- The image name is `ghcr.io/veildawn/conflux-release` (singular). The repository is `veildawn/conflux-releases` (plural). Preserve public package visibility and verify anonymous image access; public repository visibility alone does not verify package visibility.

## Signing and Verification

- `RELEASE_SIGNING_KEY` is a base64 64-byte Ed25519 private key stored only in Actions secrets. `RELEASE_PUBLIC_KEY` is the base64 32-byte public key embedded in release builds. Never print or commit private keys.
- Keep [release-pubkey.pem](release-pubkey.pem), the source installer public key, and the configured signing identity aligned. Do not rotate keys as part of routine maintenance.
- `checksums.txt.sig` is base64 Ed25519 over the exact bytes of `checksums.txt`. `SHA256SUMS.txt` must be an identical alias. Checksums cover binaries, archives, installers, and the public key.
- Manifests use domain-separated `conflux-release-manifest-v1` signatures binding component, version, OS, architecture, digest, and source commits. Preserve the signer's canonical encoding. The five Linux components and the image have individual manifests. Signed checksums also cover the macOS CLI/Agent, Windows Agent, and the network probe binaries for Linux, macOS, and Windows.
- The server and Agent installers verify binary manifests. The CLI installer verifies signed checksums for GitHub Releases downloads and checks the `X-Artifact-SHA256` response header for direct control-plane downloads.
- Before declaring a release complete, verify its run succeeded, artifacts can be downloaded anonymously, signatures/digests validate against the trusted key, self-update archive contents match standalone binaries, the runtime source matches the recorded source SHA, and the image index supports `linux/amd64` and `linux/arm64`.

## Verification Before Commit

For documentation-only changes, validate relative links, bilingual command parity, fenced shell syntax, and `git diff --check`. Confirm install flags, asset names, workflow inputs, and public-key fingerprints against the current release/source. Do not execute installation or publication examples just to check documentation.

For public configuration changes, also run:

```sh
test -s release-pubkey.pem
test -s deploy/docker-compose.yml
openssl pkey -pubin -in release-pubkey.pem -noout
# Use placeholder values only; do not dump real deployment secrets.
CONFLUX_PUBLIC_BASE_URL=https://conflux.example.com \
CONFLUX_MASTER_KEY_HEX=0000000000000000000000000000000000000000000000000000000000000000 \
POSTGRES_PASSWORD=validation-only \
  docker compose -f deploy/docker-compose.yml config --quiet
```

Validate changed workflow syntax (for example with `actionlint` when available). Changes affecting source builds/tests must pass public Source CI using the intended source SHA; changes affecting publication require appropriate build/signature validation before a new release is claimed. Keep [README.md](README.md#maintainer-workflow) and [README.zh.md](README.zh.md) current when workflow behavior changes.

## Running Deployments

This repository distributes releases; changing documentation does not authorize changing a running server. When explicitly asked to operate a deployment, use `conflux-ctl` with a Console-issued scoped `cfx_pat_*` token. Start with `whoami`, `status`, `ops tasks`, and `ops alerts`. Never request raw passwords or TOTP secrets. If the private source checkout is available, follow its `AGENTS.md` and `.agents/skills/conflux-admin/SKILL.md` operations guidance, including quota invariants and destructive-operation guardrails.
