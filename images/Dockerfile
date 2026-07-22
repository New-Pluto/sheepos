# SheepOS — immutable Kairos-based node OS for the sheepnet cluster.
#
# Ubuntu LTS base, kairosified by kairos-init, k3s bundled ("standard"
# variant), branded SheepOS, then slimmed to a minimal immutable runtime
# (no package manager, no dpkg database, no hardware firmware/microcode — see the
# slim step at the bottom). Built by .github/workflows/{build,release}.yaml
# via the Kairos Factory Action; buildable locally with plain docker:
#
#   docker build -t sheepos:dev --build-arg VERSION=v0.0.0-dev -f images/Dockerfile .
#
# Version pins live here (single source of truth for the released image):
#   - kairos-init: the FROM line below (Dependabot/Renovate bump it)
#   - Ubuntu:      ARG BASE_IMAGE default — keep in sync with the base_image
#                  input in .github/workflows/build.yaml (Renovate updates both)
#   - k3s:         ARG K3S_VERSION — keep aligned with what the cluster runs;
#                  sheepnet's Cilium-compatibility guard parses it from the
#                  ISO filename (k3sv<X.Y.Z>)

# renovate: docker-image
ARG BASE_IMAGE=ubuntu:26.04

FROM quay.io/kairos/kairos-init:v0.15.0 AS kairos-init

FROM ${BASE_IMAGE}

ARG TARGETARCH
ARG MODEL=generic
ARG TRUSTED_BOOT=false
ARG VERSION
# renovate: datasource=github-releases depName=k3s-io/k3s
ARG K3S_VERSION=v1.36.0+k3s1

# No CPU microcode is baked into SheepOS. Early-load microcode was previously
# installed here for the Skylake nodes, but it carries no fix for their
# silent-freeze erratum, and SheepOS targets a minimal image. To target newer
# CPUs where microcode updates fix real errata, install intel-microcode /
# amd64-microcode BEFORE kairos-init (so dracut bakes it into the initrd).

# Disable device-mapper multipath — defense-in-depth #1 (blacklist-all config).
# These are single-path nodes: one SATA system disk + single-portal Longhorn
# iSCSI LUNs, so multipath has no use. The base force-includes the dracut
# "multipath" module (/etc/dracut.conf.d/kairos-multipath.conf) and ships NO
# /etc/multipath.conf, so multipathd's default would claim every devnode and wrap
# the local disk (COS_STATE -> /dev/mapper/... instead of /dev/sda4) in the
# initramfs AND grab the Longhorn LUNs at runtime — which broke the 2026-06-24
# migration (sheepnet docs/sheepos-migration-2026-06-24-postmortem.md). Written
# BEFORE kairos-init so dracut's multipath module (70multipath in dracut 110)
# bakes it into the initrd too; the cmdline kill switch below is the primary
# lever and this is an independent backup.
RUN printf '%s\n' 'blacklist {' '    devnode "*"' '}' > /etc/multipath.conf

# Kairosify: 'install' adds kernel/initrd tooling, the k3s provider and agent;
# 'init' wires the boot stack and writes /etc/kairos-release (which drives
# upgrades and the AuroraBoot artifact name, e.g.
# kairos-ubuntu-26.04-standard-amd64-generic-<VERSION>-k3sv<K3S_VERSION>.iso).
# k3s is hardcoded on purpose: SheepOS is by definition a k3s node image, so the
# pin lives here rather than in workflow inputs.
RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init -l debug -s install -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}" && \
    /kairos-init -l debug -s init -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}"

# Disable device-mapper multipath — primary lever: kernel cmdline kill switch.
# The disk wrap happens in the INITRAMFS/early-udev (immucore mounts COS_STATE
# before the rootfs systemd starts), so a runtime `systemctl mask` is too late.
# `rd.multipath=0 multipath=off nompath` is the only lever honored in BOTH layers
# — re-verified against Ubuntu 26.04 packages (multipath-tools 0.14.3, dracut
# 110) in all four activation gates: udev 60-multipath.rules (nompath /
# multipath=off), the initrd multipathd.service from dracut's 70multipath
# (!nompath !multipath=off, plus the module's multipathd-dracut.conf drop-in
# gating !rd.multipath=0), and the runtime multipathd .service/.socket
# (!nompath !multipath=off). All gates read the live /proc/cmdline, so NO
# `dracut -f` is needed. /etc/cos/bootargs.cfg::setKernelCmd assembles
# $kernelcmd, which /etc/cos/grub.cfg appends to every boot entry's `linux`
# line, independent of any per-node grubenv. bootargs.cfg is written by
# kairos-init's install stage (the kairosify RUN above runs install+init
# together), so this MUST run after that RUN. The trailing grep FAILS the
# build if the append didn't land, so the image can never silently regress to
# multipath-on. iscsid is untouched.
RUN sed -i -E \
        's|^([[:space:]]*set kernelcmd="[^"]*)"|\1 rd.multipath=0 multipath=off nompath"|' \
        /etc/cos/bootargs.cfg && \
    grep -Eq '^[[:space:]]*set kernelcmd=.*multipath=off' /etc/cos/bootargs.cfg

# SheepOS branding. Only cosmetic fields are touched: /etc/kairos-release is
# left alone (kairos-agent, immucore and AuroraBoot depend on it), and the
# os-release ID stays ubuntu (ID_LIKE=debian) so package logic keeps working.
# PRETTY_NAME is what kubelet reports as the node OS image.
ARG BASE_IMAGE
# NB: source os-release in a subshell — it defines its own VERSION and would
# otherwise clobber the VERSION build arg.
RUN BASE_PRETTY="$(. /etc/os-release && echo "${PRETTY_NAME}")" && \
    sed -i --follow-symlinks \
        -e 's/^NAME=.*/NAME="SheepOS"/' \
        -e "s/^PRETTY_NAME=.*/PRETTY_NAME=\"SheepOS ${VERSION} (${BASE_PRETTY}, Kairos)\"/" \
        /etc/os-release && \
    printf '%s\n' \
        "SHEEPOS_VERSION=\"${VERSION}\"" \
        "SHEEPOS_BASE_IMAGE=\"${BASE_IMAGE}\"" \
        "SHEEPOS_K3S_VERSION=\"${K3S_VERSION}\"" \
        "SHEEPOS_HOME_URL=\"https://github.com/New-Pluto/sheepos\"" \
        > /etc/sheepos-release

# Slim to a minimal immutable runtime. SheepOS upgrades are whole-image Kairos
# A/B swaps (kairos-agent), never package operations, so the package manager is
# pointless at runtime. This removes, in order (all dpkg work runs before the
# package manager itself is deleted; nothing at runtime or on the upgrade path
# uses it): kernel firmware these wired x86 nodes never load (e1000e needs
# none); build-time leftovers (headers, static libs, toolchain); and apt/dpkg,
# their shared libs, the dpkg database, and apt metadata. Cuts hundreds of MB
# from the image so two A/B slots fit a small COS_STATE partition.
#
# NB: each dpkg-query runs one pattern group with its own guard — it exits
# non-zero when any pattern matches nothing (e.g. the microcode packages on
# arm64) while still printing whatever did match.
#
# NB: deleting the dpkg database also empties the package inventory that the CI
# SBOM attestation (release.yaml) and the Grype CVE scan (build.yaml) read. Keep
# /var/lib/dpkg below (and scan before stripping) if you want those back.
RUN set -u; \
    dpkg-query -f '${Package}\n' -W 'linux-firmware*' 'intel-microcode' 'amd64-microcode' 2>/dev/null | xargs -r dpkg --purge --force-all || true; \
    dpkg-query -f '${Package}\n' -W '*-dev' 'gcc*' 'cpp*' 'binutils*'                    2>/dev/null | xargs -r dpkg --purge --force-all || true; \
    dpkg-query -f '${Package}\n' -W 'apt' 'apt-utils' 'libapt-pkg*'                      2>/dev/null | xargs -r dpkg --purge --force-all || true; \
    rm -rf /lib/firmware /usr/lib/firmware \
           /usr/include /usr/share/locale/* /usr/share/doc /usr/share/man /usr/share/info /usr/share/help \
           /var/lib/dpkg /var/lib/apt /var/cache/apt \
           /usr/bin/apt* /usr/bin/dpkg* /usr/sbin/dpkg* \
           /usr/lib/apt /etc/apt /usr/share/keyrings \
           /usr/lib/systemd/system/apt-daily* /etc/cron.daily/apt-compat \
           /usr/lib/*/libapt-pkg*.so* \
           /var/cache/* /var/log/* /tmp/* /root/.cache 2>/dev/null || true; \
    ldconfig 2>/dev/null || true
