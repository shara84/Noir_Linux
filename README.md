# Noir Linux

A sleek, minimal penetration-testing Linux distribution.

- **Desktop:** Xfce — dark-themed (Arc-Dark + Papirus), LightDM autologin, no clutter
- **Kernel:** built from source fetched from **github.com/gregkh/linux** (stable tag
  `v7.2.11`, release string `7.2.11-noir`), with pentest-oriented options enabled
  (nftables/iptables, mac80211_hwsim virtual radios, common USB Wi-Fi adapter drivers,
  squashfs/overlay live-boot support, serial console)
- **Base:** Debian **trixie** (built with `live-build`), `iso-hybrid` image — El Torito
  CD boot + isohybrid MBR (boots from USB sticks too; BIOS/legacy, no UEFI yet)
- **Toolset:** ~100 verified packages — nmap, sqlmap, hydra, john, hashcat, aircrack-ng,
  bettercap, ettercap, recon-ng, hcxdumptool, impacket (PyPI), gobuster, ffuf, proxychains4,
  tor, tshark, and more (see `iso/config/package-lists/`)
- **Credentials (live image):** user `noir` / `noir`, root `noir`, passwordless sudo

## Repository layout

```
config.sh                    branding + versions (kernel tag, suite, mirror, work dir)
kernel/build-kernel.sh       fetch → configure → build → .deb packages → local apt repo
iso/check-packages.sh        verifies every listed package exists in trixie (Packages idx)
iso/build-iso.sh             lb config → config overlay → kernel debs → wallpaper → lb build
iso/boot-test.py             QEMU verification: serial checks + GUI screendump (PNG)
dist/                        final artifacts: ISO + .sha256 (copied from /opt/noir/out)
iso/config/
  package-lists/*.list.chroot   system / desktop / pentest / firmware sets
  hooks/0100-noir-kernel.chroot install the GitHub-built kernel as the only kernel
  hooks/0200-noir-system.chroot user, credentials, locale, services, cleanup
  includes.chroot/              banner, lightdm, sudoers, Xfce skeleton theming, wallpaper
```

## Building

Build **inside WSL/Ubuntu** (or any Linux) — never on a DrvFs/NTFS mount; the heavy
work happens in `$NOIR_WORK` (default `/opt/noir`).

```bash
# 1. one-time deps (Ubuntu/Debian)
sudo apt-get install -y build-essential bc bison flex libssl-dev libelf-dev dwarves \
  dpkg-dev rsync kmod initramfs-tools debhelper libdw-dev live-build debootstrap \
  squashfs-tools xorriso isolinux syslinux-common grub-pc-bin grub-efi-amd64-bin \
  mtools dosfstools qemu-system-x86 imagemagick fonts-dejavu-core curl git

# 2. kernel: fetch v7.2.11 from GitHub, configure, build, publish debs (~15 min)
cd kernel && sudo bash build-kernel.sh all

# 3. ISO (~30-60 min): validates package lists, configures live-build, builds
cd ../iso && sudo bash build-iso.sh
#    → /opt/noir/out/noir-linux-1.0-amd64.iso  (+ .sha256)
```

Use a different kernel tag or suite:

```bash
KERNEL_TAG=v7.3 sudo -E bash kernel/build-kernel.sh all
```

## Testing

```bash
sudo python3 iso/boot-test.py --iso /opt/noir/out/noir-linux-1.0-amd64.iso --out .
```

Phase 1 boots headless, autologins on the serial console and verifies: kernel
`7.2.11-noir`, ikconfig, systemd/lightdm/NetworkManager, squashfs root, NOPASSWD sudo,
`mac80211_hwsim` creating two radios, DHCP connectivity, and the presence of
nmap/sqlmap/hydra/aircrack-ng/impacket. Phase 2 boots the GUI and captures
`noir-desktop.png` (QEMU screendump) to verify the themed Xfce desktop.

**Latest verification (2026-10-22, shipped ISO): 18/18 PASS** — every serial check,
a clean guest poweroff, and a painted Xfce desktop screendump. Note: a full
`poweroff` of the live medium takes ~90-100 s (systemd-shutdown tears down the
squashfs/overlay layers) — that is normal, not a hang.

Verified end-to-end in QEMU/KVM on 2026-10-21: serial autologin, all tool checks,
passwordless sudo, two simulated radios, and the themed Xfce desktop
(see `noir-desktop.png` in this directory for the captured session).

## What is intentionally not packaged

These live only in Kali's repo (or not in Debian at all); install after boot if you
want them:

```bash
sudo apt install -y git
git clone --depth 1 https://github.com/sullo/nikto            # nikto
git clone --depth 1 https://github.com/danielmiessler/SecLists # wordlists (rockyou etc.)
git clone --depth 1 https://github.com/laramies/theHarvester
git clone --depth 1 https://github.com/commixproject/commix
pipx install mitmproxy          # or: pip3 install --break-system-packages mitmproxy
```

## Customization

Everything brandable lives in `config.sh` (name, release, credentials, kernel tag,
suite, mirror). Xfce look-and-feel lives in `iso/config/includes.chroot/etc/skel/`
(dark terminal, Arc-Dark, wallpaper) and the wallpaper is generated at build time by
`build-iso.sh` (ImageMagick).
