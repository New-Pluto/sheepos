# SheepOS — immutable Kairos-based node OS for the sheepnet cluster.
#
# openSUSE Leap base, kairosified by kairos-init, k3s bundled ("standard"
# variant), branded SheepOS. Built by .github/workflows/{build,release}.yaml
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

FROM quay.io/kairos/kairos-init:v0.14.6 AS kairos-init

FROM ${BASE_IMAGE}

ARG TARGETARCH
ARG MODEL=generic
ARG TRUSTED_BOOT=false
ARG VERSION
# renovate: datasource=github-releases depName=k3s-io/k3s
ARG K3S_VERSION=v1.36.0+k3s1

# Early-loading Intel CPU microcode. Installed BEFORE kairos-init so dracut
# (early_microcode=yes by default) prepends it to the initrd — the OS then no
# longer depends on the nodes' stale BIOS for microcode. This is the image-side
# mitigation for the Skylake silent-freeze issue tracked in sheepnet
# docs/audit-2026-06-09 (issue #1). Add ucode-amd here if AMD nodes ever join.
RUN if [ "${TARGETARCH}" = "amd64" ]; then \
      zypper --non-interactive install --no-recommends ucode-intel && \
      zypper clean --all; \
    fi

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
