# Troubleshooting

Problems hit while building this setup, with their causes and fixes.

## Network

### Wi-Fi and wired devices can't reach each other

- **Seen when:** Docker and Swarm first started on cubie-1.
- **Cause:** Docker loads `br_netfilter`, so traffic bridged inside `br0` (Wi-Fi ↔ wired) now goes through the nftables `forward` chain, whose policy is drop.
- **Fix:** `iifname "br0" oifname "br0" accept` in the forward chain. It's already in [`/etc/nftables.conf`](router.md#firewall-and-nat).

### Containers on cubie-1 lose their network after a firewall reload

- **Cause:** `flush ruleset` at the top of `/etc/nftables.conf` also deletes the chains Docker added.
- **Fix:** the drop-in `/etc/systemd/system/nftables.service.d/docker.conf` restarts Docker after every nftables start. If it's missing, run `sudo systemctl restart docker`.

### Nodes come back with the wrong IP or `end0`/`end1` interfaces

- **Seen when:** after `apt full-upgrade` installed Radxa kernel 6.6.98-2.
- **Cause:** the new kernel renamed the Ethernet interfaces (`eth1` → `end1`) and changed the boards' MAC addresses, so the DHCP reservations stopped matching.
- **Fix:** add `net.ifnames=0` to `/etc/kernel/cmdline` and every entry in `/boot/extlinux/extlinux.conf`, then put the new MAC addresses in the `dhcp-host` lines of `/etc/dnsmasq.d/lan.conf`. The new MAC addresses stayed the same across later reboots.

### Wi-Fi access point moves to the USB dongle

- **Seen when:** on cubie-1, after the kernel update.
- **Cause:** the USB dongle's driver (`8821cu`) loaded and took the name `wlan0`, so hostapd started the access point on the dongle. The onboard chip became `wlan1`.
- **Fix:**
  - `blacklist 8821cu` in `/etc/modprobe.d/no-usb-wifi.conf`, then `sudo update-initramfs -u`;
  - `/etc/systemd/network/10-wifi-onboard.link` pins the `aicwf_sdio` driver to `wlan0`.
- **Check:** `basename "$(readlink /sys/class/net/wlan0/device/driver)"` should print `aicwf_sdio`.

### Devices still use the old DNS after a DHCP change

- **Cause:** leases last 24 hours, and clients only pick up new DHCP options when they renew.
- **Fix:** turn Wi-Fi off and on, on the device. Devices with a hard-coded DNS server are redirected to Pi-hole anyway.

### iCloud Private Relay warning, or Cloudflare WARP won't connect

- **Cause:** on purpose. The router blocks encrypted DNS (port 853 and known DNS-over-HTTPS servers) so every device goes through Pi-hole.
- **Fix:** none needed. Let Private Relay stay off for this network, and turn WARP off or switch it to a mode that uses the network's DNS. A full VPN still works and bypasses Pi-hole.

### Lost access to the LAN while cubie-1 reboots

- **Cause:** when the `platong` Wi-Fi disappears, the laptop may switch to the upstream Wi-Fi.
- **Fix:** rejoin `platong`, or reach the LAN through acer-1: `ssh -J veriton@192.168.1.34 radxa@10.1.0.1`.

## Docker and Portainer

### Swarm and volumes vanished after swapping Docker packages

- **Seen when:** Debian's `docker.io` was purged after Docker CE had been installed on the cubies.
- **Cause:** `docker.io`'s removal script deletes `/var/lib/docker`, including the swarm state, the containers and the volumes. Portainer's admin account went with it.
- **Fix:** nothing to recover afterwards; the swarm and Portainer were rebuilt.
- **Prevention:** never purge `docker.io` on a node that has Docker state (acer-1 still uses it). Back up `/var/lib/docker` before swapping Docker packages, or swap them before creating any state.

### Portainer reports that the Docker client version is too new

- **Cause:** Portainer LTS needs Docker API 1.51 or newer. Debian's `docker.io` 26 offers API 1.45.
- **Fix:** Docker CE 29 from download.docker.com, as in [disaster recovery](disaster-recovery.md#full-rebuild) step 5.

### Portainer setup page says it timed out

- **Cause:** no admin account was created within about 5 minutes of Portainer starting.
- **Fix:** `docker service update --force portainer_portainer`, then get the new setup token from the log (see [services.md](services.md#portainer)).

### Red "failed" or "shutdown" tasks in Portainer's cluster view

- **Cause:** by default Swarm keeps up to 5 old tasks per service as history, for example the copies a reboot stopped.
- **Fix:** `docker swarm update --task-history-limit 0`, then `docker service update --force` each service once to clear the old records.

### Docker won't start: `ip-forward-no-drop` doesn't match any configuration option

- **Cause:** the option was put in `/etc/docker/daemon.json`, and Debian's Docker 26 doesn't know it.
- **Fix:** remove it. It isn't needed here: IP forwarding is enabled before Docker starts, so Docker leaves the `FORWARD` policy at ACCEPT.

### A Pi-hole blocklist shows 0 domains

- **Cause:** the list is a browser filter (cosmetic rules for uBlock/AdBlock), not a list of domains.
- **Fix:** disable it in Pi-hole and use it in uBlock Origin instead. See [blocklists](services.md#blocklists).

## Boot and system

### Boot takes 45 seconds or more

- **Cubies:**
  - **Cause:** `hdmi-toggle-once.service`, left over from the desktop image, waited about 36 s for a screen.
  - **Fix:** `systemctl set-default multi-user.target`.
- **acer-1:**
  - **Cause:** `systemd-networkd-wait-online` waited for both the wired and the Wi-Fi link, which added about two minutes.
  - **Fix:** disable it.
- **Measure:** `systemd-analyze` and `systemd-analyze blame` show where the time goes.

### `/etc/default/grub` missing on acer-1

- **Cause:** purging packages during the cleanup deleted it, and `update-grub` silently fell back to built-in defaults.
- **Fix:** it was recreated with Ubuntu's defaults and a 1 s hidden menu. After big purges, check the `update-grub` output.

### SSH session hangs during a reboot or a long job

- **Cause:** without a keepalive, the client never notices that the other side went away.
- **Fix:** set `ServerAliveInterval 10` (see [operations.md](operations.md#access)). After reconnecting, check whether apt is still running (`pgrep -a apt`) before starting it again.
