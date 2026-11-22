# Changelog

## 2026-11-23: static IPs for every node

- cubie-2 `10.1.0.151` → `10.1.0.2`, cubie-3 `10.1.0.180` → `10.1.0.3`, acer-1 `10.1.0.106` → `10.1.0.4`; cubie-1 stays `10.1.0.1`.
- Set on each board (NetworkManager connection `lan` on the cubies, netplan on acer-1), not via DHCP, so the nodes keep their addresses even if DHCP on cubie-1 is down.
- cubie-1: dnsmasq now forwards to Pi-hole at `10.1.0.2`, the nftables DNS redirect exempts `10.1.0.2`, and the old DHCP reservations are gone. Old config backed up in `/root`.
- Pi-hole admin moved to <http://10.1.0.2/admin>.

## 2026-11-22: clean rebuild, Docker Swarm, forced DNS

**Starting point:**

- cubie-1 was already the router.
- cubie-2 ran Pi-hole and cubie-3 ran Uptime Kuma, in containers under the "Kiri" engine.
- acer-1, then called `acer`, was an idle Ubuntu PC.
- All four were still on their stock images.

### Router (cubie-1)

- Rebuilt the dnsmasq, hostapd and NetworkManager configs cleanly, with the same behaviour: one `lan.conf`, and NetworkManager on the keyfile plugin only.
- Removed the Radxa desktop stack and other bloat:

  | | Before | After |
  |---|---|---|
  | Packages | 601 | 442 |
  | RAM used | ~544 MB | ~245 MB |
  | Disk used | 4.8 GB | 3.2 GB |

- Boot went from 45.3 s to 11.6 s:
  - text-only boot target, which removed a 36 s HDMI wait;
  - no splash screen;
  - SQM no longer waits for the network.
- Moved to kernel 6.6.98-2:
  - added `net.ifnames=0`;
  - pinned the onboard Wi-Fi to `wlan0`;
  - blocked the USB dongle's driver.
- The WAN address changed from 192.168.1.220 to 192.168.1.36, because the new kernel changed the MAC address.

### cubie-2 and cubie-3

- Removed Pi-hole, Uptime Kuma, the Kiri engine and their data (no backup kept), plus the same bloat as on cubie-1.
- Results:

  | | Before | After |
  |---|---|---|
  | Packages | 559 | 439 |
  | Boot time | ~45 s | ~10 s |
  | RAM used | ~500 MB | ~170 MB |

- All three cubies now share the same package set, apart from cubie-1's router packages.

### acer-1

- Renamed `acer` to `acer-1` and made it reachable as `acer-1.local` through mDNS.
- Removed about 265 packages: snap, cloud-init, VM tooling, unused storage tools and unused firmware.

  | | Before | After |
  |---|---|---|
  | Packages | 715 | 451 |
  | Disk used | 12 GB | 9.7 GB |
  | Boot time | ~2 min 45 s | 32.7 s |
  | Failed services | 4 | 0 |

- Removing `crashkernel=` freed 512 MB of RAM.

### Cluster

- Started a K3s install on acer-1, interrupted it and removed it. Docker Swarm won because it needs far less RAM.
- Created a Docker Swarm with cubie-1 as the only manager and cubie-2, cubie-3 and acer-1 as workers.
- Deployed Portainer CE, with the server on cubie-1 and an agent on every node.
- Moved the cubies from Debian's `docker.io` 26 to Docker CE 29.10, because Portainer needs a newer Docker API.
- Purging `docker.io` wiped `/var/lib/docker` on the cubies, so the swarm and Portainer were rebuilt.
- Turned off Swarm task history.
- Redeployed Pi-hole as a Swarm stack pinned to cubie-2.
- Added the HaGeZi Pro, Threat Intelligence, Gambling and DoH blocklists, and disabled the Thai browser-filter lists that yield 0 domains.

### DNS, step by step

1. **Start:** the old Pi-hole on cubie-2 was handed out by DHCP, and nftables redirected other DNS traffic to it.
2. **While cubie-2 was rebuilt:** the LAN switched to `1.1.1.1` and `1.0.0.1`, and the redirect was removed.
3. **New Pi-hole:** DHCP handed out `10.1.0.151, 1.1.1.1`.
4. **Final:**
   - DHCP hands out `10.1.0.1`;
   - cubie-1's dnsmasq forwards to Pi-hole, with `1.1.1.1` as the fallback;
   - all port-53 traffic is redirected to cubie-1;
   - DNS-over-TLS and DNS-over-QUIC are blocked.

### Firewall changes on cubie-1 for Docker

- Allowed `br0` → `br0` traffic, needed once `br_netfilter` is loaded.
- Added rules for the Docker bridges.
- Docker now restarts after every nftables reload.

### Documentation

- Replaced the MkDocs site, the Word document and the PNG diagram with plain Markdown, and drew the diagrams in Mermaid.
