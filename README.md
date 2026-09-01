# irondragonservices/iron-argo

Hardened image for running `cloudflared` — Cloudflare Tunnel.

Forked from [ironpeakservices/iron-argo](https://github.com/ironpeakservices/iron-argo).

Cloudflare's own `cloudflared` release in a `scratch` image with nothing but an
unprivileged account, CA certificates and a writable `/tmp`. Runs as uid 1000.

```sh
docker pull ghcr.io/irondragonservices/iron-argo:2026.8.3
```

The tag tracks the cloudflared release.

## Using it

```sh
docker run \
  -e TUNNEL_TOKEN=... \
  ghcr.io/irondragonservices/iron-argo:2026 \
  --no-autoupdate tunnel run
```

The entrypoint is `cloudflared`, so every subcommand and flag works as
documented. `--no-autoupdate` is in the default `CMD` deliberately: a container
that rewrites its own binary at runtime defeats the point of a signed,
pinned image.

To supply a credentials file, mount it and point at it:

```sh
docker run -v ./creds.json:/etc/cloudflared/creds.json:ro \
  ghcr.io/irondragonservices/iron-argo:2026 \
  --no-autoupdate tunnel --cred-file /etc/cloudflared/creds.json run my-tunnel
```

## Verifying what you pulled

```sh
cosign verify ghcr.io/irondragonservices/iron-argo:2026 \
  --certificate-identity-regexp '^https://github.com/irondragonservices/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Changes from upstream

- **cloudflared is no longer built from a git submodule.** Upstream vendored
  `github.com/cloudflare/cloudflared` as a submodule pinned to whatever commit
  was checked in — no tag, no release, no signature — and compiled it with a Go
  toolchain in the build. The binary is now Cloudflare's own release, taken
  from their official image. Since it is statically linked, there is nothing to
  copy but the file.
- **The submodule is gone**, so the repository clones without `--recursive` and
  Renovate can see the version, which it could not before.
- **Go 1.13.11 and Alpine 3.10** are no longer in the build. Both were roughly
  six years out of support.
- **`/tmp` added**, so anything cloudflared buffers to disk has somewhere to go.
- CI rebuilt as callers into
  [irondragonservices/.github](https://github.com/irondragonservices/.github).

Verified on build: the image reports its version and the binary runs as uid
1000.
