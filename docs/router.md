# Router: cubie-1

cubie-1 turns whatever upstream network it's plugged into into the cluster's private LAN. It's also the Swarm manager and runs the Portainer server (see [services](services.md)).

| Function | Implementation |
|---|---|
| WAN uplink | `eth1`, DHCP client (NetworkManager connection `wan`) |
| LAN bridge | `br0` = `eth0` + `wlan0`, static `10.1.0.1/24` |
| Wi-Fi access point | hostapd on the onboard AIC8800 chip (`wlan0`) |
| DHCP and DNS forwarder | dnsmasq, `/etc/dnsmasq.d/lan.conf` |
| Firewall and NAT | nftables, `/etc/nftables.conf` |
| Traffic shaping (SQM) | HTB + fq_codel at 85 Mbit on `eth1`, `sqm-eth1.service` |

Only three packages separate cubie-1 from the other cubies: `dnsmasq`, `hostapd` and `nftables`.

## Network interfaces

NetworkManager runs with the keyfile plugin only and manages the wired side through three connections in `/etc/NetworkManager/system-connections/` (UUIDs left out):

```ini
# wan.nmconnection
[connection]
id=wan
type=ethernet
interface-name=eth1
autoconnect-priority=10

[ipv4]
method=auto
dns=1.1.1.1;1.0.0.1;
ignore-auto-dns=true

[ipv6]
method=disabled
```

```ini
# br0.nmconnection
[connection]
id=br0
type=bridge
interface-name=br0
autoconnect=true
autoconnect-priority=10

[bridge]
stp=false

[ipv4]
method=manual
address1=10.1.0.1/24

[ipv6]
method=disabled
```

```ini
# br0-eth0.nmconnection
[connection]
id=br0-eth0
type=ethernet
interface-name=eth0
master=br0
slave-type=bridge
autoconnect=true
```

| Other file | Purpose |
|---|---|
| `/etc/NetworkManager/conf.d/99-unmanaged-wlan0.conf` | `unmanaged-devices=interface-name:wlan0` under `[keyfile]`, so NetworkManager leaves the AP interface to hostapd |
| `/etc/sysctl.d/99-forward.conf` | `net.ipv4.ip_forward=1` |
| `/etc/sysctl.d/99-disable-ipv6.conf` | `net.ipv6.conf.all.disable_ipv6=1` |

The `dns=` line in `wan` is for cubie-1's own lookups. They go straight to Cloudflare, not through Pi-hole, so the router doesn't depend on cubie-2.

## Wi-Fi access point

Settings in `/etc/hostapd/hostapd.conf` (passphrase left out):

```ini
interface=wlan0
bridge=br0
driver=nl80211
ssid=platong
country_code=TH
ieee80211d=1
hw_mode=a
channel=44
ieee80211n=1
ieee80211ac=1
ieee80211ax=1
ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=42
disassoc_low_ack=1
he_su_beamformer=1
he_su_beamformee=1
he_oper_chwidth=1
he_oper_centr_freq_seg0_idx=42
wmm_enabled=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=<not stored here>
ctrl_interface=/var/run/hostapd
```

That is 5 GHz channel 44 at 80 MHz (centre channel 42), Wi-Fi 4/5/6, WPA2-PSK with CCMP.

- `/etc/systemd/system/hostapd.service.d/restart.conf` sets `Restart=on-failure` and `RestartSec=5`.
- The onboard chip is an AIC8800 on SDIO (kernel driver `aicwf_sdio`), packaged as `aic8800-sdio-dkms` and `aic8800-firmware`. DKMS rebuilds it for every new kernel, which is why `gcc` and the kernel headers stay installed.
- `/etc/systemd/network/10-wifi-onboard.link` makes sure the onboard chip is always `wlan0`:

  ```ini
  [Match]
  Driver=aicwf_sdio

  [Link]
  Name=wlan0
  ```

- `/etc/modprobe.d/no-usb-wifi.conf` contains `blacklist 8821cu`, so the unused Realtek USB dongle can't take over `wlan0` (see [troubleshooting](troubleshooting.md#wi-fi-access-point-moves-to-the-usb-dongle)).

## DHCP and DNS

dnsmasq does both, on `br0` only. `/etc/dnsmasq.d/lan.conf`:

```ini
# LAN DHCP + DNS on br0. DNS forwards to Pi-hole (cubie-2), Cloudflare if Pi-hole is down.
interface=br0
bind-dynamic

# DNS: forward only, in order; add-subnet/add-mac let Pi-hole see the real client
no-resolv
strict-order
server=10.1.0.2
server=1.1.1.1
add-subnet=32,128
add-mac
cache-size=0

dhcp-range=10.1.0.100,10.1.0.200,24h
dhcp-option=option:router,10.1.0.1
dhcp-option=option:dns-server,10.1.0.1

```

| Setting | Effect |
|---|---|
| `strict-order` with two `server=` lines | Pi-hole first, `1.1.1.1` only when Pi-hole doesn't answer |
| `no-resolv` | Ignore `/etc/resolv.conf`; use only the servers listed here |
| `add-subnet=32,128` and `add-mac` | Pass each client's IP and MAC to Pi-hole, so its per-client statistics stay accurate |
| `cache-size=0` | No caching on the router; Pi-hole does the caching |
| `dhcp-option=option:dns-server,10.1.0.1` | Clients get the router as their only DNS server |
| No `dhcp-host` lines | The cluster nodes use static addresses `.1`–`.4`, outside the pool, so DHCP only serves other devices |

Tested on 2026-11-22: with Pi-hole scaled to zero, lookups still worked through `1.1.1.1` (unfiltered), and blocking resumed by itself once Pi-hole was back. Pi-hole logged the real client address, not `10.1.0.1`.

After editing: `sudo dnsmasq --test && sudo systemctl restart dnsmasq`. Clients only pick up new DHCP options when they renew their lease (24 h) or reconnect.

## Firewall and NAT

`/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
  # Force all LAN DNS through cubie-1 -> Pi-hole (Pi-hole itself talks to upstream directly)
  chain dns_redirect {
    type nat hook prerouting priority dstnat;
    iifname "br0" ip saddr != 10.1.0.2 ip daddr != { 10.1.0.1, 10.1.0.2 } meta l4proto { tcp, udp } th dport 53 redirect to :53
  }
  chain postrouting {
    type nat hook postrouting priority 100;
    oifname "eth1" masquerade
  }
}
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    iif "lo" accept
    ct state established,related accept
    ct state invalid drop
    iifname "br0" accept
    ip protocol icmp accept
  }
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    iifname "br0" tcp dport 853 reject with tcp reset   # block DNS-over-TLS
    iifname "br0" udp dport 853 reject                  # block DNS-over-QUIC
    iifname "br0" oifname "br0" accept   # LAN<->LAN (br_netfilter from Docker sends bridged traffic here)
    iifname "br0" oifname "eth1" accept
    iifname { "docker0", "docker_gwbridge" } accept   # containers out
    iifname "br0" oifname { "docker0", "docker_gwbridge" } accept   # LAN -> published ports
  }
}
```

| Rule | Why |
|---|---|
| `dns_redirect` | Any LAN query to an outside DNS server lands on cubie-1's dnsmasq instead. Pi-hole's own traffic is exempt so it can reach its upstreams |
| `oifname "eth1" masquerade` | NAT for the LAN |
| Input policy drop with `iifname "br0" accept` | The LAN can reach the router; from the WAN only replies and ping get in |
| Forward port 853 rejects | Block DNS-over-TLS and DNS-over-QUIC from the LAN |
| Forward `br0` → `br0` | Docker loads `br_netfilter`, which sends bridged Wi-Fi ↔ wired traffic through this chain. Without the rule the LAN splits in two |
| Forward `br0` → `eth1` | LAN to internet |
| Forward Docker bridges | Containers on cubie-1 can reach out, and the LAN can reach their published ports. The WAN still can't reach any container |

`flush ruleset` also wipes the chains Docker adds, so a drop-in restarts Docker after the firewall (re)loads. `/etc/systemd/system/nftables.service.d/docker.conf`:

```ini
[Service]
# flush ruleset wipes Docker chains; let Docker re-add them
ExecStartPost=-/bin/systemctl --no-block try-restart docker.service
```

`systemctl restart nftables` therefore also restarts Docker and the containers on cubie-1. Check a change before applying it with `sudo nft -c -f /etc/nftables.conf`.

Docker's own iptables `FORWARD` policy stays ACCEPT, because IP forwarding is already on before Docker starts. The restrictive policy is the `inet filter` table above.

## Traffic shaping (SQM)

SQM keeps latency low when the uplink is busy. The 85 Mbit cap should sit a little below the real line speed, so check it whenever the cluster moves to a new upstream connection.

`/usr/local/sbin/sqm-eth1.sh`:

```bash
#!/bin/bash
# SQM: HTB + fq_codel, 85 Mbit both directions on WAN eth1
set -u
apply() {
  ip link show ifb0 >/dev/null 2>&1 || ip link add ifb0 type ifb
  ip link set ifb0 up
  tc qdisc replace dev eth1 root handle 1: htb default 10
  tc class replace dev eth1 parent 1: classid 1:10 htb rate 85mbit ceil 85mbit
  tc qdisc replace dev eth1 parent 1:10 fq_codel
  tc qdisc replace dev ifb0 root handle 2: htb default 10
  tc class replace dev ifb0 parent 2: classid 2:10 htb rate 85mbit ceil 85mbit
  tc qdisc replace dev ifb0 parent 2:10 fq_codel
  tc qdisc replace dev eth1 handle ffff: ingress 2>/dev/null || true
  tc filter replace dev eth1 parent ffff: protocol all prio 1 matchall action mirred egress redirect dev ifb0
}
clear() {
  tc qdisc del dev eth1 root 2>/dev/null || true
  tc qdisc del dev eth1 ingress 2>/dev/null || true
  tc qdisc del dev ifb0 root 2>/dev/null || true
  ip link del ifb0 2>/dev/null || true
}
case "${1:-start}" in
  start) clear; apply ;;
  stop)  clear ;;
esac
```

Download traffic is shaped on `ifb0`, which receives a mirror of `eth1`'s ingress.

`/etc/systemd/system/sqm-eth1.service`:

```ini
[Unit]
Description=SQM (HTB + fq_codel 85Mbit) on WAN eth1
After=NetworkManager.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/sqm-eth1.sh start
ExecStop=/usr/local/sbin/sqm-eth1.sh stop

[Install]
WantedBy=multi-user.target
```

`/etc/NetworkManager/dispatcher.d/50-sqm-eth1` (mode 755) re-applies SQM whenever `eth1` comes up, at boot or after the cable is replugged. That's why boot doesn't wait for the network:

```sh
#!/bin/sh
[ "$1" = eth1 ] && [ "$2" = up ] && systemctl restart sqm-eth1.service
exit 0
```

HTB logs a harmless `quantum is big` warning.

## Resources and boot

- RAM: about 310–330 MB used of 899 MB, with the router, Swarm manager, Portainer server and agent all running.
- Boot: 16.6 s to a running system (11.6 s before Docker was added).
- A reboot takes routing, DHCP, DNS and Wi-Fi down for about 90 s. Board firmware and Wi-Fi bring-up take most of that.
- Boot tuning shared with the other cubies is in [nodes.md](nodes.md#boot-tuning).

## Config backups

`/root` on cubie-1 holds the tarballs taken before each change on 2026-11-22:

| File | Taken before |
|---|---|
| `router-backup-20261122-085845.tar.gz` | The config rebuild (dnsmasq, hostapd, NetworkManager, nftables) |
| `dns-backup-20261122-090508.tar.gz` | Switching the LAN from Pi-hole to Cloudflare DNS |
| `dns-force-backup-20261122-*.tar.gz` | Forcing all DNS through Pi-hole |
| `boot-backup/` | Boot tuning (original kernel command line, `extlinux.conf`, SQM unit) |
