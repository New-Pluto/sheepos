# SheepOS — immutable Kairos-based node OS for the sheepnet cluster.
#
# openSUSE Leap base, kairosified by kairos-init, k3s bundled ("standard"
# variant), branded SheepOS, then slimmed to a minimal immutable runtime
# (no package manager, no rpm database, no hardware firmware/microcode — see the
# slim step at the bottom). Built by .github/workflows/{build,release}.yaml
# via the Kairos Factory Action; buildable locally with plain docker:
#
#   docker build -t sheepos:dev --build-arg VERSION=v0.0.0-dev -f images/Dockerfile .
#
# Version pins live here (single source of truth for the released image):
#   - kairos-init: the FROM line below (Dependabot/Renovate bump it)
#   - openSUSE:    ARG BASE_IMAGE default — keep in sync with the base_image
#                  input in .github/workflows/build.yaml (Renovate updates both)
#   - k3s:         ARG K3S_VERSION — keep aligned with what the cluster runs;
#                  sheepnet's Cilium-compatibility guard parses it from the
#                  ISO filename (k3sv<X.Y.Z>)

# renovate: docker-image
ARG BASE_IMAGE=opensuse/leap:16.0

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
# CPUs where microcode updates fix real errata, install ucode-intel / ucode-amd
# BEFORE kairos-init (so dracut bakes it into the initrd).

# Kairosify: 'install' adds kernel/initrd tooling, the k3s provider and agent;
# 'init' wires the boot stack and writes /etc/kairos-release (which drives
# upgrades and the AuroraBoot artifact name, e.g.
# kairos-opensuse-leap-16.0-standard-amd64-generic-<VERSION>-k3sv<K3S_VERSION>.iso).
# k3s is hardcoded on purpose: SheepOS is by definition a k3s node image, so the
# pin lives here rather than in workflow inputs.
RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init -l debug -s install -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}" && \
    /kairos-init -l debug -s init -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}"

# SheepOS branding. Only cosmetic fields are touched: /etc/kairos-release is
# left alone (kairos-agent, immucore and AuroraBoot depend on it), and the
# os-release ID/ID_LIKE stay openSUSE so package logic keeps working.
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
# pointless at runtime. This removes, in order (all rpm work runs before the
# package manager itself is deleted; nothing at runtime or on the upgrade path
# uses it): ~650 MB of kernel firmware these wired x86 nodes never load (e1000e
# needs none); build-time leftovers (headers, static libs, toolchain); and
# zypper/rpm, their shared libs, the ~74 MB rpm database, and zypp metadata.
# Roughly halves the image so two A/B slots fit a small COS_STATE partition.
#
# NB: deleting the rpm database also empties the package inventory that the CI
# SBOM attestation (release.yaml) and the Grype CVE scan (build.yaml) read. Keep
# /usr/lib/sysimage/rpm below (and scan before stripping) if you want those back.
RUN set -u; \
    rpm -qa 'kernel-firmware*' 'ucode-*'     | xargs -r rpm -e --nodeps || true; \
    rpm -qa '*-devel' 'gcc*' 'cpp*' binutils | xargs -r rpm -e --nodeps || true; \
    rpm -qa 'zypper' 'libzypp*' 'libsolv*'   | xargs -r rpm -e --nodeps || true; \
    rm -rf /lib/firmware /usr/lib/firmware \
           /usr/include /usr/share/locale/* /usr/share/doc /usr/share/man /usr/share/info /usr/share/help \
           /usr/lib/sysimage/rpm /var/lib/rpm /usr/lib/rpm \
           /usr/bin/rpm /usr/bin/rpmdb /usr/bin/rpmkeys /usr/bin/rpm2cpio /usr/bin/zypper \
           /usr/lib64/librpm*.so* /usr/lib64/libzypp*.so* /usr/lib64/libsolv*.so* /usr/lib64/libsolvext*.so* \
           /etc/zypp /usr/lib/zypp /usr/share/zypp /var/cache/zypp \
           /var/cache/* /var/log/* /tmp/* /root/.cache 2>/dev/null || true; \
    ldconfig 2>/dev/null || true
