# Disaster recovery

## What is backed up

| Data | Lives on | Backed up |
|---|---|---|
| Router configs | cubie-1 `/etc` | Tarballs in cubie-1's `/root` from 2026-11-22 (see [router.md](router.md#config-backups)); every setting is also in this repo |
| Stack files | cubie-1 `/opt/stacks` | In [services.md](services.md), without the Pi-hole password |
| Portainer data: admin account, settings | Volume on cubie-1 | No |
| Pi-hole data: settings, lists, query history | Volume on cubie-2 | No |
| Swarm state | cubie-1 `/var/lib/docker/swarm` | No. Create a new swarm instead |
| Passwords: SSH, Wi-Fi, Pi-hole | Not in this repo | Not in this repo |

The cheapest improvements would be a Pi-hole export (Settings → Teleporter) and a copy of `/opt/stacks` kept off cubie-1.

## Full rebuild

Start with the router: every other node gets its address, DNS and internet through it.

1. **cubie-1 base:**
   - flash the Radxa Debian 13 image and set the hostname;
   - run `apt full-upgrade`;
   - apply the [cubie base system](nodes.md#cubie-base-system): remove the bloat, tune boot, add `net.ifnames=0`, the Wi-Fi `.link` file and the journal cap.
2. **cubie-1 router:**
   - `sudo apt install --no-install-recommends dnsmasq hostapd nftables`;
   - recreate everything in [router.md](router.md): the NetworkManager connections, sysctl files, `lan.conf`, `hostapd.conf` with the passphrase, `nftables.conf` with its Docker drop-in, the SQM script, unit and dispatcher hook, and the USB dongle blacklist;
   - enable `hostapd`, `dnsmasq`, `nftables` and `sqm-eth1`;
   - check that a client gets a `10.1.0.x` lease and reaches the internet.
3. **cubie-2 and cubie-3:**
   - flash them and apply the base system;
   - connect their `eth1` ports to the switch;
   - give them static addresses on `eth1`: a NetworkManager connection `lan` with `10.1.0.2/24` or `10.1.0.3/24`, gateway and DNS `10.1.0.1`.
4. **acer-1:**
   - install Ubuntu Server, set the hostname `acer-1`, a static `10.1.0.4/24` on `enp1s0f0` in netplan, and the mDNS files;
   - slim it down as described in [nodes.md](nodes.md#acer-1);
   - `sudo apt install docker.io`.
5. **Docker on the cubies.** Install Docker CE, because Debian's `docker.io` is too old for Portainer:

   ```sh
   sudo install -d -m0755 /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
   echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
     | sudo tee /etc/apt/sources.list.d/docker.list
   sudo apt update
   sudo apt install --no-install-recommends docker-ce docker-ce-cli containerd.io
   sudo usermod -aG docker radxa
   ```

   Install Docker CE before creating any containers or swarm state on the board.
6. **Swarm:**
   - on cubie-1, `docker swarm init --advertise-addr 10.1.0.1`;
   - then `docker swarm update --task-history-limit 0`;
   - join the three workers with the token from `docker swarm join-token worker`.
7. **Portainer:**
   - write `/opt/stacks/portainer.yml` and deploy it;
   - create the admin account within 5 minutes.
8. **Pi-hole:**
   - write `/opt/stacks/pihole.yml` with a new password and `chmod 600` it;
   - deploy it, re-add the [blocklists](services.md#blocklists) and rebuild gravity.
9. **Check:** run the [health checks](operations.md#health-checks).

## Replacing cubie-2 or cubie-3

1. On cubie-1, remove the old node: `docker node rm --force <hostname>`.
2. Set up the new board as in steps 1, 3 and 5 above, with the same hostname.
3. Give it its static address (`lan` connection on `eth1`).
4. Join it to the swarm.
5. **cubie-2 only:** Pi-hole's volume was on the old board, so the Pi-hole service starts again with default settings. Set the password through the stack file and re-add the blocklists. Until then, DNS goes through the `1.1.1.1` fallback.

## Losing cubie-1

cubie-1 is the only manager, so if it's lost, the swarm is lost with it. While it's down:

- there is no internet, DHCP or Wi-Fi on the LAN;
- containers on the workers keep running;
- nothing can be deployed or rescheduled.

To rebuild it:

1. Follow steps 1, 2 and 5 of the full rebuild.
2. Create a new swarm.
3. On each worker, run `docker swarm leave --force` before joining the new swarm.
4. Redeploy Portainer, which needs a new admin account.
5. Redeploy Pi-hole. Its data volume is on cubie-2, so it survives as long as cubie-2 does.

## Replacing acer-1

1. Install Ubuntu and apply [nodes.md](nodes.md#acer-1).
2. `sudo apt install docker.io`.
3. Join the swarm as a worker.
4. On cubie-1, remove the old node entry with `docker node rm --force acer-1`.
