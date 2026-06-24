# SheepOS

Custom immutable node OS for the [sheepnet](https://github.com/justup2002/sheepnet)
cluster — openSUSE Leap, kairosified with
[kairos-init](https://github.com/kairos-io/kairos-init), k3s bundled, built and
released entirely on GitHub via the
[Kairos Factory Action](https://github.com/kairos-io/kairos-factory-action).

This repo exists to own the node image supply chain: every byte that boots on a
sheep node is built from `images/Dockerfile` in CI, published as a versioned
GitHub release (ISOs for netboot) and OCI image (for in-place upgrades). The
image is **slimmed to a minimal immutable runtime** — no package manager, no rpm
database, and none of the kernel firmware/microcode these wired x86 nodes never
load — roughly half the size of a stock build, so two A/B slots fit a small
`COS_STATE` partition (see the slim step in `images/Dockerfile`).

| | |
|---|---|
| Base OS | openSUSE Leap 16.0 |
| Kubernetes | k3s `v1.36.0+k3s1` (pinned in `images/Dockerfile`) |
| Architectures | amd64, arm64 |
| Boot model | Kairos A/B immutable, `/etc` rebuilt each boot, config via `/oem` |
| ISO (per release) | `kairos-opensuse-leap-16.0-standard-<arch>-generic-<tag>-k3sv1.36.0+k3s1.iso` |
| OCI image | `ghcr.io/new-pluto/sheepos:<tag>` (multi-arch), `:<tag>-amd64`, `:<tag>-arm64`, `:latest` |

## How a release works

```
git tag v0.x.y && git push origin v0.x.y
        │
        ▼
.github/workflows/release.yaml
  1. oci      — builds images/Dockerfile once per arch, pushes
                ghcr.io/new-pluto/sheepos:<tag>-{amd64,arm64}  (SBOM attached)
  2. manifest — stitches them into the multi-arch :<tag> and :latest
  3. iso      — Kairos Factory repackages the *pushed* image (passthrough
                images/Dockerfile.release) into a bootable ISO + .sha256 and
                uploads them to the GitHub release
```

The image is built exactly once: the ISO a node netboots and the OCI image a
node upgrades to are the same build. The OCI push happens in our own job
because the factory's reusable workflow does not request `packages: write`,
so its `GITHUB_TOKEN` cannot push to ghcr.io.

Pull requests run `.github/workflows/build.yaml` — the same factory pipeline
(image + ISO + Grype CVE report) for both arches, publishing nothing. Green PR
⇒ the release will build.

> **Supply-chain note:** the slim step deletes the rpm database, so the attached
> SBOM and the Grype CVE report currently come up **empty** (both read the rpm
> package inventory). To keep them, either retain `/usr/lib/sysimage/rpm` in the
> slim step, or run the scan in CI *before* stripping.

## Using a release with sheepnet

### Netboot / (re)provision

Point `config.yaml` at the release ISO (sheepnet downloads it, verifies the
`.sha256`, and extracts kernel/initrd/squashfs with AuroraBoot):

```yaml
kairos:
  iso_url: "https://github.com/New-Pluto/sheepos/releases/download/v0.1.0/kairos-opensuse-leap-16.0-standard-amd64-generic-v0.1.0-k3sv1.36.0+k3s1.iso"
```

Keep the `k3sv<version>` part of the filename intact — sheepnet parses it to
enforce Cilium/k3s compatibility.

### In-place node upgrade

On a node (one at a time, wait for Ready between nodes):

```bash
sudo kairos-agent upgrade --source oci:ghcr.io/new-pluto/sheepos:v0.1.0
sudo reboot
```

State (`/var/lib/longhorn`, `/var/lib/rancher`, `/oem`) is preserved; the
passive partition keeps the previous image as rollback. No `--force` is needed
even for the one-time migration off the older `kairos-hadron` image — current
`kairos-agent upgrade --source oci:` performs no version/downgrade check.

This can become hands-off with the Kairos Operator (`NodeOpUpgrade`) or a Rancher
system-upgrade-controller Plan (Kairos ships `/usr/sbin/suc-upgrade` for it),
`concurrency: 1`, one node at a time. Keep OS rollouts supervised while the
silent-freeze issue is open. See the sheepnet runbook `docs/sheepos-migration.md`.

## Keeping it up to date

All moving parts are pinned; updates arrive as PRs, the `build` workflow
validates them end-to-end (both arches, full ISO build) before merge:

| Pin | Where | Bumped by |
|---|---|---|
| kairos-init | `FROM quay.io/kairos/kairos-init:vX` in `images/Dockerfile` | Dependabot + Renovate |
| Kairos Factory Action | `uses: ...reusable-factory.yaml@vX` in both workflows | Dependabot + Renovate |
| openSUSE Leap | `ARG BASE_IMAGE` default in `images/Dockerfile` **and** `base_image:` in `build.yaml` (keep in sync) | Renovate (`# renovate: docker-image`) |
| k3s | `ARG K3S_VERSION` in `images/Dockerfile` | Renovate (`# renovate:` annotation) |
| AuroraBoot | `auroraboot_version:` in both workflows | Renovate (`# renovate:` annotation) |

Dependabot works out of the box; `renovate.json` adds full coverage if the
Renovate app is installed on the org (then delete `.github/dependabot.yml`).

openSUSE security patches need no pin change at all — packages are fetched at
build time, so **cut a release roughly monthly** (or after a relevant CVE) to
pick them up. k3s bumps must respect sheepnet's Cilium compatibility floor and
ideally match what the cluster is being upgraded to.

## Customizing the image

Edit `images/Dockerfile`:

- **Packages that must land in the initrd** (storage/net drivers; microcode for
  newer CPUs): install them *before* the kairos-init `RUN` (dracut builds the
  initrd during `-s init`).
- **Everything else** (tools, configs baked into the image): add layers *after*
  kairos-init but **before the final slim step** (which deletes the package
  manager, so it must stay last).
- **Per-cluster/node config** (k3s args, users, network) does **not**
  belong here — that stays in sheepnet's cloud-config templates and `/oem`.
  SheepOS is the generic OS; sheepnet is the cluster personality.

## Local build

```bash
docker build -t sheepos:dev --build-arg VERSION=v0.0.0-dev -f images/Dockerfile .

# optional: turn it into a bootable ISO
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD/artifacts:/output" quay.io/kairos/auroraboot:v0.21.2 \
  build-iso --output /output/ docker:sheepos:dev
```
