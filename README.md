# irondragonservices/iron-cloudflared

Hardened image for running `cloudflared` — Cloudflare Tunnel.

Forked from [ironpeakservices/iron-argo](https://github.com/ironpeakservices/iron-argo),
and renamed — see below.

> **This repository used to be `iron-argo`.** Cloudflare shipped this product as
> *Argo Tunnel* and the daemon has always been `cloudflared`; they renamed the
> product to *Cloudflare Tunnel* around 2021 and narrowed "Argo" to their
> smart-routing feature. In 2026 the old name reads as
> [Argo CD](https://argo-cd.readthedocs.io), which is a completely unrelated
> thing. GitHub redirects the repository, but **the container image does not
> redirect**: `ghcr.io/irondragonservices/iron-argo` is gone. Pull
> `ghcr.io/irondragonservices/iron-cloudflared` instead.

Cloudflare's own `cloudflared` release in a `scratch` image with nothing but an
unprivileged account, CA certificates and a writable `/tmp`. Runs as uid 1000.

```sh
docker pull ghcr.io/irondragonservices/iron-cloudflared:2026.8.3
```

The tag tracks the cloudflared release.

## Using it

```sh
docker run \
  -e TUNNEL_TOKEN=... \
  ghcr.io/irondragonservices/iron-cloudflared:2026 \
  --no-autoupdate tunnel run
```

The entrypoint is `cloudflared`, so every subcommand and flag works as
documented. `--no-autoupdate` is in the default `CMD` deliberately: a container
that rewrites its own binary at runtime defeats the point of a signed,
pinned image.

To supply a credentials file, mount it and point at it:

```sh
docker run -v ./creds.json:/etc/cloudflared/creds.json:ro \
  ghcr.io/irondragonservices/iron-cloudflared:2026 \
  --no-autoupdate tunnel --cred-file /etc/cloudflared/creds.json run my-tunnel
```

## Verifying what you pulled

```sh
cosign verify ghcr.io/irondragonservices/iron-cloudflared:2026 \
  --certificate-identity-regexp '^https://github\.com/irondragonservices/\.github/\.github/workflows/image-(release|refresh)\.yml@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository irondragonservices/iron-cloudflared
```

Be precise about the identity. The signature is produced by the shared
reusable workflow in
[irondragonservices/.github](https://github.com/irondragonservices/.github),
not by a workflow in this repository, so the certificate names *that* path.
A looser pattern such as `^https://github.com/irondragonservices/` would
accept a signature from any workflow in any repository in the organisation,
which is a much weaker claim than it looks. The
`--certificate-github-workflow-repository` flag is what ties the signature back
to this repository.

Both `image-release` and `image-refresh` sign: the nightly rebuild republishes
when the package set has actually changed, and it signs what it pushes.

## Known findings

Every vulnerability the scanner reports against this image lives inside the
`cloudflared` binary, which is Cloudflare's build. As of 2026.8.3 — the latest
release — that is Go 1.26.4 and `golang.org/x/crypto` v0.53.0, carrying one
CRITICAL (`CVE-2026-56854`, `x/crypto/ssh`) and nine HIGH stdlib findings.

There is no newer release to move to. The alternative is compiling
`cloudflared` here from the tagged source with a current toolchain and a bumped
`x/crypto`, which would clear all ten — and would also mean shipping a
`cloudflared` that Cloudflare did not build, did not sign and will not support,
for a daemon whose whole job is holding an authenticated tunnel into your
network. Taking their binary is the more defensible of the two.

So they are listed in [`.trivyignore`](.trivyignore) with expiry dates, and
suppressed **on the pull request gate only**. The release scan runs without a
gate, so the SARIF uploaded to code scanning still carries them, and the
nightly re-scan still opens and updates the tracking issue. When the dates pass
the gate blocks again and somebody has to check whether Cloudflare has shipped
a fix.

If that trade is not one you want to make, build from source and pin your own
toolchain.

## Changes from upstream

- **The base packages are now upgraded, not just added to.** The step commented
  *update base system* only installed `ca-certificates`, so the image shipped
  whatever the base image tag happened to contain. Distributions patch a
  package well before they rebuild and republish the base image, so a digest
  pin — which is what Renovate maintains — pins the *unpatched* set until
  upstream gets round to a rebuild. `alpine:3.24.1` was carrying openssl
  3.5.7-r0 with a fixed HIGH against it and 3.5.8-r0 already in the repository.
  This is also what makes the nightly cache-free rebuild worth running: without
  it, that job rebuilt the same packages every night and picked up nothing.
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
