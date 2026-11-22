# Operations

## Access

An SSH config for the admin machine. The keepalive matters: without it, a session hangs instead of ending when the other side reboots.

```
Host cubie-1 cubie-2 cubie-3 acer-1
  HostName %h.local
  ServerAliveInterval 10
Host cubie-*
  User radxa
Host acer-1
  User veriton
```

If cubie-1's Wi-Fi is down, join the upstream network and hop through acer-1's Wi-Fi address:

```sh
ssh -J veriton@192.168.1.34 radxa@10.1.0.1
```

Web UIs: Portainer at <https://10.1.0.1:9443>, Pi-hole at <http://10.1.0.2/admin>.

## Health checks

On cubie-1:

```sh
docker node ls        # 4 nodes Ready, cubie-1 Leader
docker service ls     # pihole_pihole 1/1, portainer_agent 4/4, portainer_portainer 1/1
systemctl is-active hostapd dnsmasq NetworkManager nftables sqm-eth1 docker
sudo hostapd_cli -i wlan0 status | grep -E '^(state|channel|ssid)'
sudo tc qdisc show dev eth1    # htb root with fq_codel
```

From any LAN client:

```sh
dig +short @10.1.0.1 doubleclick.net   # 0.0.0.0: Pi-hole is filtering
dig +short @8.8.8.8 doubleclick.net    # 0.0.0.0 as well: the DNS redirect works
dig +short @10.1.0.1 example.com       # a real address
```

On any node, to confirm it's still clean:

```sh
systemctl --failed
dpkg -l | grep -c '^rc'                  # leftover package configs, expect 0
docker ps -a --filter status=exited      # expect nothing
```

## Updating

Go one node at a time: cubie-3, cubie-2, acer-1, and cubie-1 last.

```sh
sudo apt update && sudo apt full-upgrade
```

- **New kernels:** `full-upgrade` can install a new Radxa kernel, and DKMS then rebuilds the Wi-Fi driver for every installed kernel. That takes several minutes on a cubie (over 10 minutes on cubie-1 last time). Keep the SSH keepalive on. If the session drops anyway, reconnect and check `pgrep -a apt` before doing anything else.
- **Before rebooting after a kernel update:**
  - `dkms status` shows `aic8800` built for the new kernel;
  - `grep net.ifnames=0 /etc/kernel/cmdline /boot/extlinux/extlinux.conf` still matches.
- **After the reboot:**
  - `uname -r` shows the new kernel;
  - `ip -br link` shows `eth0`, `eth1` and `wlan0`;
  - on cubie-1, `sudo hostapd_cli -i wlan0 status` shows `state=ENABLED`.
- **No network after the reboot:** check that the interface is still called `eth1` (`net.ifnames=0`); the static `lan` connection is bound to that name.
- **Docker updates:** on the cubies, Docker CE updates through apt like everything else. Upgrading `docker-ce` restarts Docker and that node's containers; on cubie-1 that includes the Portainer server.
- **Never purge `docker.io`** on acer-1, or on any node with Docker state. Its removal script deletes `/var/lib/docker`, including the swarm state and volumes.

## Rebooting

| Node | What goes down | Notes |
|---|---|---|
| cubie-1 | Routing, DHCP, DNS and Wi-Fi for about 90 s, and swarm management | Containers on the other nodes keep running |
| cubie-2 | Pi-hole | DNS falls back to `1.1.1.1` without ad-blocking until it's back |
| cubie-3 | Nothing important | |
| acer-1 | Its own containers | |

## Routine tasks

| Task | How |
|---|---|
| Deploy or update a stack | Portainer → Stacks, or `docker stack deploy -c /opt/stacks/<name>.yml <name>` on cubie-1 |
| Update Pi-hole | `docker service update --image pihole/pihole:latest pihole_pihole` on cubie-1. DNS uses the `1.1.1.1` fallback for a few seconds |
| Update Portainer | `docker service update --image portainer/portainer-ce:lts portainer_portainer`, then the same for `portainer_agent` with `portainer/agent:lts` |
| Unblock a domain a list broke | Pi-hole → Query Log → Allow |
| Move a device onto new DHCP settings | Turn its Wi-Fi off and on; otherwise it waits for its 24 h lease to renew |
| Clear stopped containers and unused images | `docker container prune -f && docker image prune -f`, on each node |
| Clear the apt cache | `sudo apt clean`. The daily apt job fills `/var/cache/apt` again over time |
| Change the SQM speed | Edit the `85mbit` values in `/usr/local/sbin/sqm-eth1.sh`, then `sudo systemctl restart sqm-eth1` |
