# Nodes

## Hardware and OS

| | cubie-1, cubie-2, cubie-3 | acer-1 |
|---|---|---|
| Model | Radxa Cubie A5E | Acer Veriton |
| CPU | Allwinner A527 (sun55iw3), 8× Cortex-A55, arm64 | Intel Core i5-6400T, 4 cores, x86_64 |
| RAM | 1 GB (899 MB usable) | 23 GB usable |
| Storage | ~30 GB eMMC/SD, ext4 | 476 GB SSD, ext4 |
| OS | Debian 13 (trixie), Radxa image | Ubuntu 26.10 (server install) |
| Kernel | 6.6.98-2-aw2607, with 6.6.98-1 kept as a fallback boot entry | 7.3.0-9-generic |
| Login user | `radxa` | `veriton` |
| `.local` name | `cubie-N.local` via avahi | `acer-1.local` via systemd-resolved |
| Docker | Docker CE 29.10.0 from download.docker.com | `docker.io` 29.7.2 from the Ubuntu archive |

Measured on 2026-11-22:

| Node | RAM used | Disk used | Boot time |
|---|---|---|---|
| cubie-1 | ~330 MB | 4.2 GB | 16.6 s |
| cubie-2 | ~320 MB | 4.0 GB | 10.4 s * |
| cubie-3 | ~290 MB | 3.8 GB | 10.2 s * |
| acer-1 | ~0.8 GB | 11 GB | 32.7 s from power-on, 16 s after the bootloader * |

\* Measured before Docker was installed.

## Cubie base system

The three cubies have an identical package set; cubie-1 adds only `dnsmasq`, `hostapd` and `nftables`. Each runs nothing but the basics: NetworkManager, ssh, avahi, journald, udev, getty and time sync, plus Docker. cubie-1 adds its [router services](router.md).

### Removed

- Radxa's desktop stack: Mesa, LLVM, GTK3, the Xorg Mali driver, icon and font themes, `desktop-base`, and plymouth (boot splash).
- samba, the pipewire/wireplumber audio stack, haveged, and leftover X11 and Wayland libraries.
- On cubie-2 and cubie-3: the old Pi-hole and Uptime Kuma containers, the "Kiri" container engine and `/var/lib/kiri`.
- Home-folder tooling: `~/.t3`, `~/.pi`, `~/.local`, `~/.npm` and `~/.docker`.
- Every translation except English. dpkg is set to skip other languages on future installs.

### Kept on purpose

| Package or service | Why |
|---|---|
| `aic8800-sdio-dkms`, `aic8800-firmware` | Onboard Wi-Fi driver |
| `gcc-14`, kernel headers | DKMS needs them to rebuild the Wi-Fi driver after kernel updates |
| `avahi-daemon` | `cubie-N.local` names, used for SSH |
| `rsetup`, `radxa-ecm@`, `radxa-otgutils` | Radxa tooling. USB-gadget Ethernet is a recovery path if the network config breaks |
| `lm-sensors`, `wpa_supplicant`, vim, perl | Small, harmless basics |

### Boot tuning

| Change | How |
|---|---|
| Text-only boot | `systemctl set-default multi-user.target`. This removed a 36 s wait for HDMI left over from the desktop image |
| No splash | `splash` removed from `/etc/kernel/cmdline` and `/boot/extlinux/extlinux.conf` |
| Classic interface names | `net.ifnames=0` added to the same two files, so the kernel keeps `eth0`/`eth1` |
| Onboard Wi-Fi is always `wlan0` | `/etc/systemd/network/10-wifi-onboard.link` (see [router.md](router.md#wi-fi-access-point)) |
| Services disabled | `systemd-boot-update` (the boards boot through U-Boot and extlinux), `bluetooth`, `NetworkManager-wait-online`. On cubie-1, `rsetup-hciattach@ttyAS1` is also masked |

U-Boot's `timeout 10` is in tenths of a second, so the boot menu already waits only 1 s.

### Housekeeping

- The journal is capped at 30 MB: `/etc/systemd/journald.conf.d/size.conf` sets `SystemMaxUse=30M`.
- Docker CE comes from Docker's own repository (`/etc/apt/sources.list.d/docker.list`, key in `/etc/apt/keyrings/docker.asc`), because Debian's `docker.io` 26 is too old for Portainer. `radxa` is in the `docker` group.

## acer-1

- **Name:** hostname `acer-1` (it was `acer`); the login user is still `veriton`.
- **mDNS without avahi:** systemd-resolved announces `acer-1.local`.
  - `/etc/systemd/resolved.conf.d/mdns.conf`: `[Resolve]` `MulticastDNS=yes`
  - `/etc/systemd/network/10-netplan-enp1s0f0.network.d/mdns.conf` and the same for `wlp2s0`: `[Network]` `MulticastDNS=yes`
- **Network (netplan with systemd-networkd):**
  - `enp1s0f0` is wired, with DHCP from cubie-1 (10.1.0.106).
  - `wlp2s0` is on the upstream Wi-Fi (192.168.1.34). It's kept on purpose as a way into the LAN when cubie-1's Wi-Fi is down: SSH to 192.168.1.34, then hop to 10.1.0.x.
- **Removed (about 265 packages):**
  - snapd and its snaps;
  - cloud-init, open-vm-tools, lxd and landscape;
  - Ubuntu Pro, apport and motd-news;
  - fwupd, upki, packagekit, ModemManager, udisks2, upower and bolt;
  - LVM, RAID, multipath, iSCSI, XFS, btrfs and cryptsetup tooling (the disk is plain ext4);
  - plymouth, rsyslog (the journal is enough), sysstat, kdump and thermald;
  - firmware for hardware it doesn't have. Only Realtek, Intel Wi-Fi and Intel graphics firmware remain.
- **Kept:** ssh, netplan, the kernel, GRUB, shim and initramfs-tools, chrony, unattended-upgrades, git, tmux, vim, nano, htop, rsync, curl, strace, lsof and tcpdump.
- **Boot:**
  - text-only boot target;
  - `systemd-networkd-wait-online` disabled (it waited for both the wired and Wi-Fi links and caused a two-minute delay);
  - `crashkernel=` removed, which freed 512 MB of RAM;
  - GRUB menu hidden with a 1 s timeout (hold Shift or Esc to see it). `/etc/default/grub` was recreated with Ubuntu's defaults after a package purge deleted it.
- **Same as the cubies:** English-only translations and a 30 MB journal cap.
- **Docker:** Ubuntu's `docker.io` package. Don't purge it: its removal script deletes `/var/lib/docker` (see [troubleshooting](troubleshooting.md#swarm-and-volumes-vanished-after-swapping-docker-packages)).
