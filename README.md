# homelab-lxc-images

Custom LXC container images, built with [distrobuilder](https://linuxcontainers.org/distrobuilder/),
for use with [homelab-tofu](https://github.com/dentonbrown/homelab-tofu)
(private — the actual OpenTofu config for `pve01`, kept separate from
this repo).

## Why a separate, public repo

`homelab-tofu`'s Proxmox resources fetch templates via
`proxmox_virtual_environment_download_file`, which downloads over a
plain URL — Proxmox itself makes the outbound request, so nothing needs
inbound access to the home network. But that also means the URL can't
carry a GitHub auth token, so it has to be unauthenticated — which a
private repo's release assets can't be. This repo is public so that
works, and public is fine here: it's a generic Alpine + nginx build
recipe and the resulting image, nothing about the home network itself.

## Images

### `alpine-nginx`

Alpine base, `nginx` installed via `apk`, `nginx.conf` and a site config
baked in at build time (via distrobuilder's `files` generators), nginx
enabled under OpenRC. No post-boot provisioning needed — it's ready to
serve on first start. An early proof-of-concept; superseded in practice
by `alpine-docker` below for anything that needs to actually run a
service (see [homelab-tofu](https://github.com/dentonbrown/homelab-tofu)
for why: per-service LXCs running Docker, not bespoke distrobuilder
images per app).

- Definition: [`alpine-nginx/alpine-nginx.yaml`](alpine-nginx/alpine-nginx.yaml)
- Config baked in: [`alpine-nginx/files/`](alpine-nginx/files/)

### `alpine-docker`

Alpine + Docker + the Compose v2 CLI plugin — no app baked in. This is
the actual base image for the "one LXC per service, Docker inside"
pattern: `homelab-tofu` provisions one of these per service, sized to
that service's needs, and a `docker-compose.yml` supplies the actual
application on top. Logic mirrors
[community-scripts/ProxmoxVE's `alpine-docker.sh`](https://github.com/community-scripts/ProxmoxVE/blob/main/install/alpine-docker-install.sh),
rebuilt declaratively — same packages (`tzdata`, `openssl`, `docker`),
Compose pulled from GitHub releases the same way, `rc-update add docker
default` under OpenRC. Deliberately **excludes** two of that script's
optional extras: Portainer (a GUI, orthogonal to managing this via code)
and the Docker TCP socket on `0.0.0.0:2375` (unauthenticated remote
Docker API access — a real risk on a LAN; add it per-instance later, on
purpose, if something actually needs it).

- Definition: [`alpine-docker/alpine-docker.yaml`](alpine-docker/alpine-docker.yaml)

### Publishing

Both are built and published automatically by
[`.github/workflows/build.yml`](.github/workflows/build.yml) (a matrix
job) on every push to `main` that touches either image's directory, or
manually via `workflow_dispatch`. Each publishes as its own
**`<image>-latest`** GitHub release (overwritten every build, so the
download URL never changes):

```
https://github.com/dentonbrown/homelab-lxc-images/releases/download/alpine-nginx-latest/alpine-nginx.tar.zst
https://github.com/dentonbrown/homelab-lxc-images/releases/download/alpine-docker-latest/alpine-docker.tar.zst
```

Those URLs are what `homelab-tofu` points its `download_file` resources
at.

## Build pipeline notes

- distrobuilder installed via `go install` in CI (Go is preinstalled on
  `ubuntu-latest`) — simpler than the snap package in a CI context.
- `distrobuilder build-lxc` outputs raw LXC's two-part format
  (`rootfs.tar.xz` + `meta.tar.xz`). Proxmox only wants the rootfs, as a
  single `.tar.zst` (not `.tar.xz`) — the workflow converts it and
  discards the metadata file.
- Alpine release version is pinned in each YAML (`release: "3.24"`,
  current as of 2026-07) — check
  [alpinelinux.org/releases](https://alpinelinux.org/releases/) and bump
  both periodically rather than letting them silently go stale.

## Adding more images

Same pattern: a new subdirectory with its own `<name>.yaml` (+ `files/`
if it needs baked-in config), and an entry in the workflow's `matrix.image`
list.
