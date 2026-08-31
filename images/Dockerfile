# SheepOS — immutable Kairos-based node OS for the sheepnet cluster.
#
# Ubuntu LTS base, kairosified by kairos-init, k3s bundled ("standard"
# variant), branded SheepOS, slimmed to a minimal immutable runtime. Slimming
# happens at INSTALL time where possible: dpkg path filters + the minimal
# firmware package set below keep unneeded firmware and kernel-module classes
# from ever landing (so the dracut initrd is built lean, and the OCI pull
# shrinks too), then a final slim layer drops the package manager, toolchain
# leftovers and the dpkg database. Built by .github/workflows/{build,release}.yaml
# via the Kairos Factory Action; buildable locally with plain docker:
#
#   docker build -t sheepos:dev --build-arg VERSION=v0.0.0-dev -f images/Dockerfile .
#
# Version pins live here (single source of truth for the released image):
#   - kairos-init: the FROM line below (Dependabot/Renovate bump it)
#   - Ubuntu:      ARG BASE_IMAGE default — build.yaml reads this ARG at
#                  runtime (base-image job), so the gate always validates the
#                  same base a release build uses
#   - k3s:         ARG K3S_VERSION — keep aligned with what the cluster runs;
#                  sheepnet's Cilium-compatibility guard parses it from the
#                  ISO filename (k3sv<X.Y.Z>)
#
# HARDWARE CONTRACT — this image must boot two machine classes:
#   - Lenovo M700 Tiny (Skylake: i915 + skl_dmc, e1000e, SATA) — current fleet
#   - Minisforum MS-03 (Panther Lake: xe iGPU needing xe/ptl_guc+ptl_huc and
#     i915/xe3* display DMC — without them the xe driver tears down simpledrm
#     and then fails to probe, permanently killing the only console; RTL8127A
#     10GbE via r8169 + rtl_nic/rtl8127a firmware; X710 SFP+ via i40e and
#     i226-LM via igc, both firmware-less; NVMe) — incoming
# plus the cluster runtime: Longhorn (iscsi_tcp), Cilium (bpf/netfilter/
# vxlan/geneve/wireguard), NFSv4.1 client. The firmware keep-set and module
# excludes below encode exactly that, and the post-conditions in the final
# layer FAIL THE BUILD if the contract regresses.
#
# SIZE CONTRACT — COS_STATE on the M700s is 2.4 GB and an in-place upgrade
# peaks at THREE image slots (active + passive + transition). Self-sustaining
# A/B therefore needs each ext2 slot <= ~816 MiB used. Measured on this image:
# a 3072 MiB ext2 slot adds ~116 MB of fs overhead over raw du, so the final
# layer asserts du -sxb / <= 720 MB, which keeps the slot <= ~800 MiB with
# real-world overhead — upgrades can never wedge COS_STATE on ENOSPC.
# The budget only holds if it measures what installers MATERIALIZE: the
# artifact pipeline (AuroraBoot OCI unpack -> mksquashfs -> installer rsync)
# breaks hardlinks into independent full copies, and Ubuntu's Rust coreutils
# is a 114-entry hardlink farm of one ~11 MB multi-call binary (+1.28 GB
# installed — found by the v0.2.0 boot smoke, where the slot came out 1.9 GB
# against this 720 MB assert). The slim layer collapses hardlink farms into
# symlinks, which every pipeline stage preserves, and the final layer fails
# the build if any multi-hardlinked file remains.

# renovate: docker-image
ARG BASE_IMAGE=ubuntu:26.04

FROM quay.io/kairos/kairos-init:v0.17.3 AS kairos-init

FROM ${BASE_IMAGE}

ARG TARGETARCH
ARG MODEL=generic
ARG TRUSTED_BOOT=false
ARG VERSION
# renovate: datasource=github-releases depName=k3s-io/k3s
ARG K3S_VERSION=v1.36.4+k3s1

# ---------------------------------------------------------------------------
# Everything up to the kairos-init RUN shapes what apt/dpkg install and what
# dracut bakes into the initrd — order matters.
# ---------------------------------------------------------------------------

# Drop Ubuntu's stock 'ubuntu' user (uid/gid 1000, in sudo+adm) so uid 1000
# stays free: upgraded M700s keep their persisted kairos/sheep uids and a
# fresh MS-03 allocates the same ones — identical account layout on both
# machine classes instead of a collision with a never-used distro account.
RUN userdel -r ubuntu 2>/dev/null || true

# Install-time slimming: dpkg path filters. Packages still install (the
# dependency graph stays intact), but dpkg only writes whitelisted paths.
#
# Firmware: keep exactly what the two machine classes load, as DIRECTORIES —
# blob names encode firmware branches (ptl_guc_70.bin), so a filename list
# would silently re-break on version bumps. The bare 'path-include=...:/i915'
# line is required: dpkg only auto-recreates a filtered-out directory when an
# include pattern's pre-wildcard prefix covers it, so file-glob includes like
# i915/skl_dmc* need their parent directory re-included explicitly.
#
# Kernel modules: exclude classes that cannot matter on these wired x86 k3s
# nodes. Exemptions inside drivers/media: cec and rc-core are hard modules.dep
# dependencies of BOTH i915 and xe. Deliberately kept: all of
# drivers/net/ethernet, nvme/ata/scsi core, md/dm, netfilter/sched, usb/hid,
# thunderbolt (MS-03 USB4), fs/nfs (client; only the nfsd SERVER is cut).
RUN printf '%s\n' \
        '# SheepOS install-time slimming — see images/Dockerfile header' \
        'path-exclude=/usr/lib/firmware/*' \
        'path-exclude=/lib/firmware/*' \
        'path-include=/usr/lib/firmware/xe/*' \
        'path-include=/usr/lib/firmware/rtl_nic/*' \
        'path-include=/usr/lib/firmware/i915' \
        'path-include=/usr/lib/firmware/i915/skl_dmc*' \
        'path-include=/usr/lib/firmware/i915/xe3*' \
        'path-exclude=/usr/lib/modules/*/kernel/sound/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/media/*' \
        'path-include=/usr/lib/modules/*/kernel/drivers/media/cec/*' \
        'path-include=/usr/lib/modules/*/kernel/drivers/media/rc/rc-core.ko*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/net/wireless/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/wireless/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/mac80211/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/bluetooth/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/bluetooth/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/gpu/drm/amd/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/gpu/drm/nouveau/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/gpu/drm/radeon/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/infiniband/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/staging/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/comedi/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/iio/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/isdn/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/accel/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/mtd/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/net/wwan/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/net/can/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/can/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/net/dsa/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/dsa/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/net/wan/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/scsi/lpfc/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/scsi/qla2xxx/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/scsi/bfa/*' \
        'path-exclude=/usr/lib/modules/*/kernel/drivers/scsi/elx/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/ocfs2/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/ceph/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/gfs2/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/afs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/ubifs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/jfs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/f2fs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/nfsd/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/smb/*' \
        'path-exclude=/usr/lib/modules/*/kernel/fs/btrfs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/net/ceph/*' \
        'path-exclude=/usr/lib/modules/*/kernel/zfs/*' \
        'path-exclude=/usr/lib/modules/*/kernel/kheaders.ko*' \
        'path-exclude=/usr/lib/modules-load.d/zfs.conf' \
        'path-exclude=/lib/modules-load.d/zfs.conf' \
        > /etc/dpkg/dpkg.cfg.d/50sheepos-slim

# Install ONLY the firmware packages these machines need, before kairos-init:
# linux-image-generic hard-depends on plain 'linux-firmware', but
# linux-firmware-minimal declares Provides/Conflicts/Replaces: linux-firmware,
# so once it is installed apt treats that dependency as satisfied and can
# never co-install the full metapackage — the other 16 vendor firmware
# subpackages (~640 MB) are never even downloaded (verified: zero
# linux-firmware unpacks during the kairos-init stage).
# linux-firmware-intel-graphics ships xe/ + i915/, linux-firmware-realtek
# ships rtl_nic/ — the path filters above trim both to the keep-set.
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        linux-firmware-minimal linux-firmware-intel-graphics linux-firmware-realtek && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Disable device-mapper multipath — defense-in-depth #1: a blacklist that
# actually parses, in a path no package owns. NOTE: /etc/multipath.conf is a
# registered dpkg conffile of multipath-tools on Ubuntu (kairos-init installs
# it below); pre-writing THAT file makes dpkg hit an unanswerable conffile
# prompt in the noninteractive build and abort. /etc/multipath/conf.d/ is
# owner-free and dracut's 70multipath module copies it into the initrd.
# devnode ".*" — the pattern is a POSIX ERE: a bare "*" has nothing to repeat,
# fails to parse, and silently disables the entire file (v0.1.0 shipped that
# bug, so its blacklist never worked; the cmdline kill switch was the only
# real lever). These are single-path nodes: one system disk + single-portal
# Longhorn iSCSI LUNs; multipathd wrapping the local disk broke the
# 2026-06-24 migration (sheepnet docs/sheepos-migration-2026-06-24-postmortem.md).
RUN mkdir -p /etc/multipath/conf.d && \
    printf '%s\n' 'blacklist {' '    devnode ".*"' '}' \
        > /etc/multipath/conf.d/00-sheepos-blacklist.conf

# Two dracut knobs that must exist before kairos-init runs dracut:
#
# 1. No early-microcode cpio (~21.7 MB/initrd). The M700 BIOS already carries
#    the only matching Skylake patch (06-5e-03 rev 0xf0; Linux skips non-newer
#    revisions), Ubuntu's intel-microcode has no Panther Lake blobs yet, and
#    the path filter above drops the blobs anyway (the microcode packages
#    themselves are hard deps of linux-image-generic and still install).
#    The real microcode lever for both machines is a BIOS/UEFI update.
#
# 2. Netboot with several NICs cabled must not stall: the initrd's
#    systemd-networkd-wait-online gates dracut-initqueue with no --any, so
#    rd.neednet=1 waits ~2 min for EVERY link to come up. Kairos ships the
#    --any override only in the ROOTFS; dracut's networkd module skips /etc
#    unit drop-ins when hostonly=no (kairos sets that), so use install_items
#    (same mechanism 99-immucore.conf uses for /etc/hosts). In the rootfs the
#    kairos override coexists harmlessly with this one.
RUN mkdir -p /etc/dracut.conf.d /etc/systemd/system/systemd-networkd-wait-online.service.d && \
    printf 'early_microcode=no\n' > /etc/dracut.conf.d/02-sheepos-no-ucode.conf && \
    printf '%s\n' '[Service]' 'ExecStart=' \
        'ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --any' \
        > /etc/systemd/system/systemd-networkd-wait-online.service.d/10-any.conf && \
    printf 'install_items+=" /etc/systemd/system/systemd-networkd-wait-online.service.d/10-any.conf "\n' \
        > /etc/dracut.conf.d/10-sheepos-networkd-any.conf

# Kairosify: 'install' adds kernel/initrd tooling, the k3s provider and agent;
# 'init' wires the boot stack, builds the initrd (dracut — reading everything
# staged above) and writes /etc/kairos-release (which drives upgrades and the
# AuroraBoot artifact name, e.g.
# kairos-ubuntu-26.04-standard-amd64-generic-<VERSION>-k3sv<K3S_VERSION>.iso).
# k3s is hardcoded on purpose: SheepOS is by definition a k3s node image, so
# the pin lives here rather than in workflow inputs.
RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init -l debug -s install -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}" && \
    /kairos-init -l debug -s init -m "${MODEL}" -t "${TRUSTED_BOOT}" \
        -p k3s --provider-k3s-version "${K3S_VERSION}" --version "${VERSION}"

# Heal persisted /etc/systemd state on upgraded nodes. /etc/systemd is a
# Kairos PERSISTENT_STATE_PATH synced with rsync WITHOUT --delete, so unit
# symlinks from earlier OS generations survive every upgrade forever. The
# openSUSE-era fleet persisted /etc/systemd/system/dbus.service ->
# dbus-broker.service; Ubuntu has no dbus-broker, so that becomes a dangling
# /etc override that permanently and silently kills D-Bus on every upgraded
# M700 (dbus.socket start-limit-hit, logind dead, no boot failure => no
# auto-rollback, and no later image can heal it because rsync never deletes).
# Prune dangling symlinks at the initramfs stage: after immucore has bind-
# mounted the persisted /etc/systemd, before systemd loads units. Verified a
# no-op on fresh installs (zero dangling links in the built image).
RUN printf '%s\n' \
        'stages:' \
        '  initramfs:' \
        '    - name: "Prune dangling unit symlinks from persisted /etc/systemd"' \
        '      commands:' \
        '        - find /etc/systemd -xtype l -delete || true' \
        > /system/oem/90_sheepos_state_prune.yaml

# Kernel cmdline, appended to every installed-system GRUB entry via
# /etc/cos/bootargs.cfg (which travels INSIDE the image — upgrades never
# rewrite COS_GRUB, so these args are guaranteed on the first boot of every
# new image):
#   - rd.multipath=0 multipath=off nompath — multipath kill switch, the
#     primary lever; honored by all four activation gates on Ubuntu 26.04
#     (udev 60-multipath.rules, dracut 70multipath's multipathd-dracut.conf
#     drop-in, and the runtime multipathd .service/.socket), all reading the
#     live /proc/cmdline — no dracut rebuild needed. This applies to the
#     INSTALLED system only: the netboot/ISO installer boots its own cmdline
#     (sheepnet boot.ipxe.tmpl / AuroraBoot grub.cfg), where the conf.d
#     blacklist above is the operative layer.
#   - net.naming-scheme=v259 — freeze udev interface naming at the systemd
#     259 scheme so future systemd bumps can never silently rename eno1 and
#     strand the kube-vip VIP. NB: openSUSE used 'sle15-sp4', which Ubuntu's
#     udev rejects ("Unknown interface naming scheme"), so the migration hop
#     itself lands on v259 regardless — verify `ip -br link` still shows eno1
#     on the canary M700 before touching the other nodes.
# bootargs.cfg is written by kairos-init's install stage, so this MUST run
# after the kairosify RUN. The greps FAIL the build if the append didn't
# land; multipath -t proves the conf.d blacklist parses (a parse failure
# silently disables the whole blacklist — exactly the v0.1.0 bug).
# The find prunes ~5 zero-byte dangling symlinks dpkg's filter layer leaves
# behind in the kept firmware dirs (targets were excluded).
RUN sed -i -E \
        's|^([[:space:]]*set kernelcmd="[^"]*)"|\1 rd.multipath=0 multipath=off nompath net.naming-scheme=v259"|' \
        /etc/cos/bootargs.cfg && \
    grep -Eq '^[[:space:]]*set kernelcmd=.*multipath=off' /etc/cos/bootargs.cfg && \
    grep -Eq '^[[:space:]]*set kernelcmd=.*net\.naming-scheme=v259' /etc/cos/bootargs.cfg && \
    multipath -t > /dev/null && \
    multipath -t | grep -q 'devnode "\.\*"' && \
    find /usr/lib/firmware -xtype l -delete

# Ubuntu's pam_unix ships 'nullok': an account with an empty password field
# passes console auth (openSUSE rejected it, and the cluster account is
# passwordless by design). SSH was never exposed — sshd short-circuits empty
# passwords before PAM — this closes the console path. The guard fails the
# build if the option ever returns.
RUN sed -i 's/\(pam_unix\.so\)[[:space:]]\+nullok/\1/' /etc/pam.d/common-auth && \
    ! grep -q nullok /etc/pam.d/common-auth

# rpcbind: nfs-common (kept — its mount.nfs serves in-tree nfs volumes and
# debug mounts; the planned csi-driver-nfs pins NFSv4.1 which needs no
# portmapper) hard-depends on rpcbind, whose maintainer scripts enable it on
# 0.0.0.0:111 + [::]:111. Nothing here needs a public portmapper: drop the
# enable links and pin the socket to loopback. The [::1] lines are NOT
# optional — with only the IPv4 lines rpcbind self-opens [::]:111 wildcard
# sockets alongside the systemd-owned ones (verified on a live systemd 259
# boot). Done in the image so the enable links never reach the nodes'
# persisted /etc/systemd (synced without --delete: once there, no later
# image could remove them).
# fail2ban: kairos-init installs and enables it; Ubuntu adds an ACTIVE sshd
# jail with banaction=nftables — an unrequested nftables-mutating agent on a
# Cilium-dataplane node. openSUSE's jail.d was empty; converge to that
# (fail2ban-server v1.1.0 verified healthy with zero jails).
RUN rm -f /etc/systemd/system/multi-user.target.wants/rpcbind.service \
          /etc/systemd/system/sockets.target.wants/rpcbind.socket \
          /etc/fail2ban/jail.d/defaults-debian.conf && \
    mkdir -p /etc/systemd/system/rpcbind.socket.d && \
    printf '%s\n' '[Socket]' 'ListenStream=' 'ListenDatagram=' \
        'ListenStream=/run/rpcbind.sock' \
        'ListenStream=127.0.0.1:111' 'ListenDatagram=127.0.0.1:111' \
        'ListenStream=[::1]:111' 'ListenDatagram=[::1]:111' \
        > /etc/systemd/system/rpcbind.socket.d/10-loopback-only.conf

# SheepOS branding. Only cosmetic fields are touched: /etc/kairos-release is
# left alone (kairos-agent, immucore and AuroraBoot depend on it), the
# os-release ID stays ubuntu (kairos-agent's install path reads it) and
# PRETTY_NAME keeps containing "Ubuntu" (yip's only_os stages match it
# against 'Ubuntu.*'). PRETTY_NAME is what kubelet reports as the node OS.
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

# Install-target picker: the fleet's definition of "auto" device selection.
# sheepnet's rendered cloud-config references it as
#   install.device: "script:///usr/bin/sheepos-pick-disk"
# kairos-agent (>= v2.30.0, in-image) resolves script:// at install-spec
# build time — BEFORE any partitioning or cloud-config stage — executing the
# command and using its stdout as the device; non-zero exit or empty output
# aborts the install with nothing written. That timing is also why the script
# must live in the image: no stage has run yet that could have written it.
# Deliberately stricter than kairos-agent's own "auto" (largest disk, USB
# included): install to the machine's ONLY eligible disk, refuse otherwise.
COPY --chmod=0755 <<'PICKDISK' /usr/bin/sheepos-pick-disk
#!/bin/sh
# SheepOS install-target picker (install.device: script:// resolver).
# stdout = chosen device; non-zero exit = abort install (pre-partitioning).
# Eligible disk: TYPE=disk, not USB-attached (TRAN!=usb), not removable
# (RM!=1), not zram/ram/fd/eMMC-boot. Exactly one eligible disk -> print it;
# zero or several -> refuse with the inventory on stderr, so the operator
# sets an explicit nodes[].install_device (prefer /dev/disk/by-id/...) in
# sheepnet's config.yaml for that node.
set -u

count=0
picked=""
for name in $(lsblk -dn -o NAME,TYPE | awk '$2 == "disk" { print $1 }'); do
    case "$name" in
        zram*|ram*|fd*|mmcblk*boot*) continue ;;
    esac
    # TRAN/RM are queried per device: a combined multi-column listing
    # misparses in awk when TRAN is empty (fields shift left). A FAILED
    # query must not fail open — an empty result would pass both filters
    # and count an unqueryable (possibly USB) disk as eligible — and must
    # not fail half-closed either (skipping just this disk could bless a
    # sibling while this one's nature is unknown): refuse outright. The
    # status check needs assign-then-trim; piping into tr would mask it.
    if ! tran=$(lsblk -dn -o TRAN "/dev/$name"); then
        echo "sheepos-pick-disk: cannot query TRAN of /dev/$name; refusing to guess." >&2
        exit 1
    fi
    tran=$(printf '%s' "$tran" | tr -d '[:space:]')
    if [ "$tran" = "usb" ]; then continue; fi
    if ! rm_flag=$(lsblk -dn -o RM "/dev/$name"); then
        echo "sheepos-pick-disk: cannot query RM of /dev/$name; refusing to guess." >&2
        exit 1
    fi
    rm_flag=$(printf '%s' "$rm_flag" | tr -d '[:space:]')
    if [ "$rm_flag" = "1" ]; then continue; fi
    picked="/dev/$name"
    count=$((count + 1))
done

if [ "$count" -eq 1 ]; then
    printf '%s\n' "$picked"
    exit 0
fi

{
    echo "sheepos-pick-disk: REFUSING to pick an install target: $count eligible disks (need exactly 1)."
    echo "Eligible = TYPE=disk, TRAN!=usb, RM!=1. Full inventory:"
    lsblk -dn -o NAME,SIZE,TYPE,TRAN,RM,MODEL 2>/dev/null
    echo "Fix: set nodes[].install_device for this node in sheepnet config.yaml (prefer a stable /dev/disk/by-id/ path)."
} >&2
exit 1
PICKDISK

# Syntax-gate the picker at build time; a broken script would otherwise only
# surface as an install-time abort on real hardware.
RUN sh -n /usr/bin/sheepos-pick-disk && test -x /usr/bin/sheepos-pick-disk

# Final slim + post-conditions (keep this the LAST layer). SheepOS upgrades
# are whole-image Kairos A/B swaps (kairos-agent), never package operations,
# so the package manager is pointless at runtime. Purged: toolchain
# leftovers; the ZFS stack (no pools here — also stops zfs-load-module
# dragging deprecated systemd-udev-settle into every boot); ubuntu-pro/
# python3-apt (dead without apt, their timers/services enabled otherwise);
# file/magic, mime and xkb databases nothing consumes; the console-setup
# stack (console-setup/-linux, keyboard-configuration — headless nodes on
# the kernel's built-in US keymap/font; purging xkb-data while leaving its
# consumer made console-setup.service fail every boot, issue #15); neovim
# (kairos-init also installs nano, which stays as the emergency editor);
# then apt/dpkg themselves (all dpkg work runs before dpkg is deleted).
# Also dropped:
# kairos p2p extras (edgevpn/pebble — sheepnet provisions via its own
# netboot), console fonts except the default, /boot/System.map (nothing
# reads it at runtime), /boot/grub (install-time GRUB assets live in
# /usr/lib/grub + /usr/share/grub), and the motd-news/dpkg-db-backup units
# (owned by essential packages, so links/units are removed by hand). The
# console-setup enablement links and /etc/console-setup cache are removed
# by hand too: postinst-generated, so purge may leave them behind.
#
# set -eu: any failed purge, rm or assert FAILS THE BUILD (the old
# '|| true' chain let a broken slim step ship silently). --no-triggers:
# purges must not re-run the dracut trigger — the initrd is final, and the
# md5 assert proves it. NB: deleting /var/lib/dpkg empties the dpkg
# inventory, but SBOM/Grype coverage comes from the systemd ELF
# .note.package stamps that survive on every binary (~190 source-named
# packages; Grype output is byte-identical with or without the dpkg DB).
#
# NB: each dpkg-query pipeline is guarded per-group: dpkg-query exits
# non-zero when a pattern matches nothing (expected: e.g. 'cpp*' matches no
# real package) but still prints what did match; the pipe discards that
# status and xargs -r skips empty input. A failing dpkg --purge DOES fail
# the build.
#
# Hardlink farms become symlink farms: AuroraBoot's OCI unpack breaks
# hardlinks into independent copies (mksquashfs dedup then hides the cost in
# the ISO, and the installer rsync materializes every copy on disk — Ubuntu's
# uutils coreutils alone inflates the installed rootfs by ~1.28 GB, 3x the
# slot budget). Each hardlink group is collapsed to its first path as the one
# real file plus symlinks for the rest; uutils dispatches on argv[0] basename,
# so /usr/bin/md5sum -> farm/md5sum -> farm/<kept> behaves identically. The
# rm/ln used inside the loop are staged as dereferenced copies first: both
# ARE farm members, so the loop iteration that processes ln would otherwise
# delete the only ln between its rm and its ln -s ("ln: not found", and the
# staged copies must keep their exact basenames for argv[0] dispatch). The
# post-condition below fails the build if any hardlink GROUP (two or more
# visible paths sharing an inode) survives — it cannot test raw st_nlink:
# the unlinks land as overlayfs whiteouts, so the kept lower-layer file
# still reports its old link count even though only one path remains (and
# only path count matters: that is what the unpack multiplies by).
RUN set -eu; \
    INITRD_MD5="$(md5sum /boot/initrd | cut -d' ' -f1)"; \
    dpkg-query -f '${Package}\n' -W '*-dev' 'gcc*' 'cpp*' 'binutils*' 2>/dev/null \
        | xargs -r dpkg --purge --force-depends --force-remove-reinstreq --no-triggers; \
    dpkg-query -f '${Package}\n' -W 'zfsutils*' 'libzfs*' 'libnvpair*' 'libuutil*' 'libzpool*' \
        'ubuntu-pro-client*' 'ubuntu-advantage*' 'python3-apt' \
        'file' 'libmagic*' 'xkb-data' 'console-setup*' 'keyboard-configuration' \
        'shared-mime-info' 'neovim*' 2>/dev/null \
        | xargs -r dpkg --purge --force-depends --force-remove-reinstreq --no-triggers; \
    dpkg-query -f '${Package}\n' -W 'apt' 'apt-utils' 'libapt-pkg*' 2>/dev/null \
        | xargs -r dpkg --purge --force-depends --force-remove-reinstreq --no-triggers; \
    rm -rf /usr/include /usr/share/locale/* /usr/share/doc /usr/share/man /usr/share/info /usr/share/help \
           /var/lib/dpkg /var/lib/apt /var/cache/apt \
           /usr/bin/apt* /usr/bin/dpkg* /usr/sbin/dpkg* \
           /usr/lib/apt /etc/apt /usr/share/keyrings \
           /usr/lib/systemd/system/apt-daily* /etc/cron.daily/apt-compat \
           /usr/lib/*/libapt-pkg*.so* \
           /usr/share/mime /usr/share/nvim \
           /usr/bin/edgevpn /usr/bin/pebble \
           /boot/grub /usr/lib/systemd/system/motd-news.service /usr/lib/systemd/system/motd-news.timer \
           /usr/lib/systemd/system/dpkg-db-backup.service /usr/lib/systemd/system/dpkg-db-backup.timer \
           /etc/systemd/system/timers.target.wants/motd-news.timer \
           /etc/systemd/system/timers.target.wants/dpkg-db-backup.timer \
           /etc/systemd/system/multi-user.target.wants/console-setup.service \
           /etc/systemd/system/sysinit.target.wants/keyboard-setup.service \
           /etc/console-setup \
           /var/cache/* /var/log/* /tmp/* /root/.cache; \
    rm -f /boot/System.map-* /usr/lib/firmware/regulatory.db*; \
    [ ! -d /usr/share/consolefonts ] || \
        find /usr/share/consolefonts -type f ! -name 'default8x16*' -delete; \
    find / -xdev -type f -links +1 -printf '%D:%i\t%p\n' | sort \
        | awk -F'\t' '{ if ($1 == prev) print keep "\t" $2; else { prev = $1; keep = $2 } }' \
        > /tmp/hardlink-dups.tsv; \
    mkdir -p /tmp/slimtools; \
    cp -L /usr/bin/rm /usr/bin/ln /tmp/slimtools/; \
    while IFS="$(printf '\t')" read -r keep dup; do \
        /tmp/slimtools/rm "$dup" && /tmp/slimtools/ln -s "$keep" "$dup"; \
    done < /tmp/hardlink-dups.tsv; \
    rm -rf /tmp/hardlink-dups.tsv /tmp/slimtools; \
    ldconfig; \
    \
    [ "$INITRD_MD5" = "$(md5sum /boot/initrd | cut -d' ' -f1)" ] \
        || { echo "FATAL: slim step modified /boot/initrd"; exit 1; }; \
    [ -z "$(find / -xdev -type f -links +1 -printf '%D:%i\n' | sort | uniq -d)" ] \
        || { echo "FATAL: hardlink groups remain (the OCI->ISO->installer pipeline breaks hardlinks into full copies, voiding the size budget)"; exit 1; }; \
    if [ "${TARGETARCH:-amd64}" = "amd64" ]; then \
        KVER="$(ls /usr/lib/modules)"; \
        for m in xe i915 cec rc-core e1000e igc igb i40e r8169 nvme ahci \
                 iscsi_tcp dm-multipath overlay br_netfilter vxlan geneve \
                 wireguard veth cls_bpf xt_socket xt_TPROXY nft_compat; do \
            modprobe --dry-run -S "$KVER" "$m" > /dev/null \
                || { echo "FATAL: module $m unresolvable (hardware contract)"; exit 1; }; \
        done; \
        FW_LIST="$(modinfo -k "$KVER" xe r8169 i915 2>/dev/null | awk '/^firmware:/{print $2}' \
            | grep -E '^(xe/ptl_(guc|huc)|rtl_nic/rtl8127|i915/skl_dmc|i915/xe3)' | sort -u)"; \
        [ "$(printf '%s\n' "$FW_LIST" | grep -c .)" -ge 5 ] \
            || { echo "FATAL: firmware want-list too short: $FW_LIST"; exit 1; }; \
        for f in $FW_LIST; do \
            [ -e "/usr/lib/firmware/$f" ] || [ -e "/usr/lib/firmware/$f.zst" ] || [ -e "/usr/lib/firmware/$f.xz" ] \
                || { echo "FATAL: firmware $f missing (hardware contract)"; exit 1; }; \
        done; \
    fi; \
    SZ="$(du -sxb / 2>/dev/null | cut -f1)"; \
    echo "SheepOS rootfs: ${SZ} bytes"; \
    if [ "${TARGETARCH:-amd64}" = "amd64" ]; then \
        [ "$SZ" -le 720000000 ] \
            || { echo "FATAL: rootfs ${SZ} B over the 720000000 B budget (ext2 slot would exceed ~800 MiB; COS_STATE self-sustaining ceiling is ~816 MiB)"; exit 1; }; \
    fi
