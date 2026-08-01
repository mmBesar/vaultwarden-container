# vaultwarden-container

Multi-arch (amd64 / arm64 / riscv64) Vaultwarden image, built **natively**
on every architecture — no QEMU anywhere in the pipeline.

Images: `ghcr.io/mmbesar/vaultwarden-container`

This is a **drop-in replacement** for `vaultwarden/server` — same
`docker/Dockerfile.debian` from upstream, unmodified, just built for
riscv64 too (which upstream doesn't publish) and rebuilt automatically.
No PUID/PGID or TZ handling baked into the image, same as upstream — use
`user:` and a bind mount for that, same as you would with the official
image (see `compose.yaml`).

## Why not a custom Dockerfile

Upstream's `docker/Dockerfile.debian` already builds natively when run on
the host's own architecture — `docker/bake.sh` with no arguments compiles
for whatever machine you run it on, no QEMU involved (the `xx-cargo`
tooling in there is for cross-compiling to *other* arches from one host;
when target == host it's effectively a native build already). So rather
than reimplement their build recipe, this repo just runs their exact
Dockerfile on three native CI runners instead of you doing it by hand on
three separate machines.

## Branches

- **`upstream`** — daily mirror of [dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden)
  `main`, with upstream's own `.github/` stripped (avoids the
  GITHUB_TOKEN workflow-permission block). Otherwise untouched — including
  their own `.dockerignore` and `docker/Dockerfile.debian`.
- **`main`** — this repo's own files: `compose.yaml`, the two workflows,
  and two ledger files:
  - `UPSTREAM_VERSION` — latest stable release tag
  - `UPSTREAM_MASTER_SHA` — latest upstream commit (drives `dev` builds)

## Workflows

1. **`sync-upstream.yml`** (daily, 02:15 UTC) — mirrors upstream into the
   `upstream` branch, mirrors the latest release tag (source only,
   stripped of `.github/`), and updates the ledger files on `main` if
   anything changed.
2. **`image-build.yaml`** (triggered by ledger file changes) — checks out
   upstream's source at the pinned tag (stable) or branch HEAD (dev), and
   builds `docker/Dockerfile.debian` directly on three native runners.
   Before a build is allowed to become the public `:latest`/`:1.37`/etc.
   tags, it has to clear two gates:
   - **Smoke test** — pulls the just-pushed per-arch image on the same
     native runner that built it, runs it with volatile/no-auth test
     settings, and polls `/alive` before anything downstream trusts it.
   - **Vulnerability scan** — Trivy scans the pushed image for CRITICAL,
     fixable CVEs (`ignore-unfixed: true`, so it doesn't block releases on
     upstream base-image issues nobody can patch yet). Results go to the
     repo's Security tab either way. Stable is hard-gated on this; `dev`
     is scan-and-report only, since it's a rolling build off unreleased
     upstream code by definition.

   Only after both gates pass does the manifest job merge the per-arch
   digests into the public multi-arch tags.

## Security notes

- `aquasecurity/trivy-action` is pinned to a full commit SHA, not a
  version tag. In March 2026 an attacker force-pushed 75 of 76 version
  tags in that repo to a credential-stealing payload (CVE-2026-33634).
  Tags are mutable and can be moved after the fact; a commit SHA can't.
  Don't change this back to a version tag without checking current
  advisories first.
- Every build includes an SBOM and provenance attestation
  (`sbom: true`, provenance is on by default in `docker/build-push-action@v7`).

## Tags produced

| Tag | Channel |
|---|---|
| `latest`, `1.37.1`, `1.37`, `1` | stable, floating + pinned |
| `dev` | rolling build off upstream's `main` |

## Running it

```yaml
services:
  vaultwarden:
    image: ghcr.io/mmbesar/vaultwarden-container:latest
    container_name: vaultwarden
    user: "${PUID}:${PGID}"
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.yourdomain.tld"
    volumes:
      - ./vw-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "127.0.0.1:8000:80"
```

Front it with a reverse proxy (Caddy, nginx, Traefik) for TLS.

## First-time setup on GitHub

1. Create the repo `vaultwarden-container` under your account.
2. Push these files to `main`.
3. Run `sync-upstream.yml` once manually (`workflow_dispatch`) — this
   creates the `upstream` branch and mirrors the current stable tag.
4. `image-build.yaml` fires automatically once the ledger files update.

No secrets to configure — both workflows use the default `GITHUB_TOKEN`
(`contents: write` / `packages: write`, which the repo grants by default).
