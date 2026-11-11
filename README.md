# SheepOS

Custom immutable node OS for the [sheepnet](https://github.com/justup2002/sheepnet)
cluster — Ubuntu, kairosified with
[kairos-init](https://github.com/kairos-io/kairos-init), k3s bundled, built and
released entirely on GitHub via the
[Kairos Factory Action](https://github.com/kairos-io/kairos-factory-action).

This repo exists to own the node image supply chain: every byte that boots on a
sheep node is built from `images/Dockerfile` in CI, published as a versioned
GitHub release (ISOs for netboot) and OCI image (for in-place upgrades). The
image is **slimmed to a minimal immutable runtime**, mostly at *install* time:
only the firmware packages the fleet's two machine classes actually need are
installed (Ubuntu's split `linux-firmware-*` vendor packages + dpkg path
filters trim them to the exact keep-set — Panther Lake `xe/` + `i915/xe3*` +
the NPU 5 blob `intel/vpu/vpu_50xx*`, Skylake `i915/skl_dmc*`, Realtek
`rtl_nic/`), whole kernel-module classes
that can't matter on wired x86 k3s nodes are never written to disk, and the
final layer removes the package manager and the dpkg database entirely.
Upgrades are whole-image A/B swaps, so none of that is needed at runtime.
An in-place upgrade peaks at **three** image slots on the `COS_STATE`
partition (active + passive + transition). The amd64 build fails if the
rootfs exceeds 850 MB (≈ 925 MiB/slot with measured ext2 overhead, ≈ 2.8 GiB
for three), keeping that guarantee mechanical rather than aspirational. The
MS-03 fleet's `COS_STATE` is 6.8 GB, so this is a headroom budget, not the
physical limit — it was 720 MB while the fleet was M700s with a 2.4 GB
`COS_STATE` — and a small image stays a design goal: the budget is a
ceiling, not a target. Measured at the Ubuntu migration: ~660 MB rootfs
→ ~740 MiB/slot, smaller than the openSUSE image it replaces (733 MiB) while
adding MS-03 hardware support. Ubuntu 26.04's default coreutils (uutils: one
~11 MB multi-call binary behind 116 hardlinks) is swapped for GNU coreutils at
install time — the OCI → ISO → installer pipeline breaks hardlinks into full
copies, which would otherwise triple the installed footprint, and Ubuntu's
rust-coreutils 0.10.0 rejects symlinked applets outright — any remaining
hardlink group is collapsed to symlinks at build time, and the build fails if
one survives.

| | |
|---|---|
| Base OS | Ubuntu 26.04 LTS |
| Kubernetes | k3s — pinned by `ARG K3S_VERSION` in `images/Dockerfile` (see [Keeping it up to date](#keeping-it-up-to-date)); each release stamps that pin into its ISO name as `k3sv<version>` |
| Architectures | amd64 — the fleet target; the hardware contract and the 850 MB size budget are build-enforced. arm64 — built, CVE-scanned and published on every release, but **untested**: no arm64 nodes exist and the contract/size asserts do not run for it |
| Boot model | Kairos A/B immutable, `/etc` rebuilt each boot, config via `/oem` |
| ISO (per release) | `kairos-ubuntu-26.04-standard-<arch>-generic-<tag>-k3sv<k3s-version>.iso` (e.g. `...-v0.2.3-k3sv1.36.3+k3s1.iso`) |
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
  3. iso      — after manifest: Kairos Factory repackages the *pushed* image
                (passthrough images/Dockerfile.release) into a bootable ISO +
                .sha256 and uploads them to the GitHub release
```

The image is built exactly once: the ISO a node netboots and the OCI image a
node upgrades to are the same build. The OCI push happens in our own `oci`
job so the ISO jobs repackage exactly the bytes that were pushed, and the
GitHub release is only created once the multi-arch upgrade tag exists.

Pull requests run `.github/workflows/build.yaml` — the same factory pipeline
(image + ISO + Grype CVE report) for both arches, publishing nothing. Green PR
⇒ the release will build.

> **Supply-chain note:** the slim step deletes the dpkg database
> (`/var/lib/dpkg`), but SBOM/CVE coverage does **not** go dark: Ubuntu 26.04
> stamps ELF binaries with systemd `.note.package` metadata, which syft/Grype
> read directly (~190 source-named OS packages, plus full Go-module and
> kernel coverage). Grype results are byte-identical with or without the dpkg
> DB. The scan stays `report-only`: its Critical findings are NVD kernel-CPE
> matches against `/boot/vmlinuz` produced by `--add-cpes-if-none` —
> unactionable noise that would permanently fail an enforcing gate.

## Using a release with sheepnet

### Netboot / (re)provision

Point `config.yaml` at the release ISO (sheepnet downloads it, verifies the
`.sha256`, and extracts kernel/initrd/squashfs itself with a pure-Go ISO
reader that replicates AuroraBoot's extraction contract — no docker or
AuroraBoot on the provisioning side since sheepnet PR #72):

```yaml
kairos:
  iso_url: "https://github.com/New-Pluto/sheepos/releases/download/v0.2.3/kairos-ubuntu-26.04-standard-amd64-generic-v0.2.3-k3sv1.36.3+k3s1.iso"
```

Keep the `k3sv<version>` part of the filename intact — sheepnet parses it to
enforce Cilium/k3s compatibility.

### In-place node upgrade

On a node (one at a time, wait for Ready between nodes):

```bash
sudo kairos-agent upgrade --source oci:ghcr.io/new-pluto/sheepos:v0.2.3
sudo reboot
```

State (`/var/lib/longhorn`, `/var/lib/rancher`, `/oem`) is preserved; the
passive partition keeps the previous image as rollback. No `--force` is needed
even for the one-time migration off the older `kairos-hadron` image — current
`kairos-agent upgrade --source oci:` performs no version/downgrade check.

In sheepnet this is hands-off since 2026-08: a health-gated Rancher
system-upgrade-controller Plan (Kairos ships `/usr/sbin/suc-upgrade` for it)
rolls label-armed nodes one at a time — merging the sheepos version-bump PR in
sheepnet triggers the rollout. See the sheepnet runbook
`flux/clusters/sheepnet/node-os/README.md`.

## Keeping it up to date

All moving parts are pinned; updates arrive as Renovate PRs, and the `build`
workflow validates each one end-to-end (both arches, full ISO build) before
merge:

| Pin | Where | Policy |
|---|---|---|
| kairos-init | `FROM quay.io/kairos/kairos-init:vX` in `images/Dockerfile` | automerge non-major |
| Kairos Factory Action | `uses: ...reusable-factory.yaml@<ref>` in both workflows | automerge non-major |
| AuroraBoot | `auroraboot_version:` in both workflows (`# renovate:` annotation) | automerge non-major |
| Ubuntu | `ARG BASE_IMAGE` default in `images/Dockerfile` (single source of truth — `build.yaml` reads the ARG at runtime) | dashboard approval, LTS tags only |
| k3s | `ARG K3S_VERSION` in `images/Dockerfile` (`# renovate:` annotation) | human merge. Patch releases of the pinned minor arrive as ordinary PRs; a new **minor** (or major) needs Dependency Dashboard approval — it must respect sheepnet's Cilium compatibility floor and the Kubernetes support of everything the cluster runs |

Bump PRs come from the Mend Renovate app (installed on the New-Pluto org),
which also digest-pins the workflow `uses:` refs; GitHub's Dependabot
**alerts** stay enabled in the repo settings for security advisories, but
Dependabot version PRs are retired. Automerge is safe here because merging
publishes nothing — releases only ship when a `v*` tag is pushed, and
sheepnet's SUC rollout gates what actually reaches nodes.

Ubuntu security patches need no pin change at all — packages are fetched at
build time, so **cut a release roughly monthly** (or after a relevant CVE) to
pick them up. k3s bumps must respect sheepnet's Cilium compatibility floor and
ideally match what the cluster is being upgraded to: `main` should always be
on the Kubernetes minor the next release is meant to ship, because a release
tag ships whatever `main` pins. (`main` briefly carried 1.37.1 from
2026-10-21; it was re-pinned to the 1.36 line on 2026-11-11 before any
release was cut, since the cluster's CNI and storage did not yet list 1.37.)

## Customizing the image

Edit `images/Dockerfile`:

- **Packages that must land in the initrd** (storage/net drivers; firmware):
  install them *before* the kairos-init `RUN` (dracut builds the initrd during
  `-s init`), and check the dpkg path filters at the top don't exclude their
  files — new hardware usually means adding a `path-include=` line (firmware)
  or removing a module-class `path-exclude=` line.
- **Everything else** (tools, configs baked into the image): add layers *after*
  kairos-init but **before the final slim step** (which deletes the package
  manager, so it must stay last). Watch the size budget assert — it exists so
  the A/B upgrade path on `COS_STATE` never regresses, and so the image
  stays small by default.
- **Per-cluster/node config** (k3s args, users, network) does **not**
  belong here — that stays in sheepnet's cloud-config templates and `/oem`.
  SheepOS is the generic OS; sheepnet is the cluster personality.

## Serial console (Intel AMT Serial-over-LAN)

The MS-03 nodes are only ever watched remotely, over the AMT SOL UART — a
PCI 16550A at I/O `0x4020` that Linux enumerates as `ttyS4`. Three pieces
make that console a complete boot record:

- **Kernel console** — `console=ttyS4,115200n8` in `/etc/cos/bootargs.cfg`
  (travels inside the image).
- **Console loglevel** — `/usr/lib/sysctl.d/60-sheepos-console.conf` sets
  `kernel.printk = 7 4 1 7`, overriding Ubuntu's `4 4 1 7`, which otherwise
  silences everything below KERN_ERR about 3 s into the initrd. The drop-in
  is baked into the initrd too (build-asserted).
- **GRUB menu** — `/usr/bin/sheepos-grub-serial` runs on every boot and keeps
  a small snippet on `COS_STATE` (`/grub_sheepos_serial`, hooked from
  `/grubcustom`) that points GRUB's output at the same UART. Kairos never
  rewrites `COS_STATE`'s `grub.cfg` on upgrade, so this is the path that
  reaches existing nodes; the menu appears on SOL from the **second** boot of
  an image that carries the helper. The snippet is installed only while Linux
  sees a 16550A at exactly `0x4020` and is removed otherwise.

GRUB serial **input** is off by default: a GRUB that reads from a port which
is not a real UART sees phantom keypresses, which stops the menu countdown
and strands an unattended node at the menu. To drive the menu over SOL on a
node (for example to pick the fallback entry), opt in once:

```bash
sudo grub2-editenv /oem/grubenv set sheepos_serial_input=yes    # undo: ...=no
```

## Local build

```bash
docker build -t sheepos:dev --build-arg VERSION=v0.0.0-dev -f images/Dockerfile .

# optional: turn it into a bootable ISO
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD/artifacts:/output" quay.io/kairos/auroraboot:v0.21.2 \
  build-iso --output /output/ docker:sheepos:dev
```

## License

SheepOS is licensed under the [GNU Affero General Public License v3.0](LICENSE)
(SPDX: `AGPL-3.0-only`). Copyright © 2026 New-Pluto. The licenses of the
components the built image aggregates (Kairos, k3s, Ubuntu packages, …) are
unaffected and remain with their upstreams.
