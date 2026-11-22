# Services

## Docker Swarm

| | |
|---|---|
| Manager | cubie-1, created with `docker swarm init --advertise-addr 10.1.0.1` |
| Workers | cubie-2, cubie-3, acer-1, joined to `10.1.0.1:2377` |
| Task history | Off: `docker swarm update --task-history-limit 0` |
| Availability | Every node `Active`. cubie-1 can't be drained, because the Portainer server has to run on a manager |
| Stack files | `/opt/stacks/` on cubie-1 |

Swarm traffic between nodes uses TCP 2377, TCP and UDP 7946, and UDP 4789 (overlay networks, which need the `vxlan` kernel module; all nodes have it). cubie-1 accepts everything that arrives on `br0`.

Adding a worker:

```sh
# on cubie-1
docker swarm join-token worker
# on the new node, with the token printed above
docker swarm join --token <token> 10.1.0.1:2377
```

## Portainer

| | |
|---|---|
| URL | <https://10.1.0.1:9443>. Through the routing mesh, port 9443 on any node's IP works |
| Images | `portainer/portainer-ce:lts`, `portainer/agent:lts` |
| Data | Volume `portainer_portainer_data` on cubie-1: admin account and settings |
| Stack file | `/opt/stacks/portainer.yml` on cubie-1 |

```yaml
version: "3.8"
services:
  agent:
    image: portainer/agent:lts
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks: [agent_network]
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]
  portainer:
    image: portainer/portainer-ce:lts
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9443:9443"
    volumes:
      - portainer_data:/data
    networks: [agent_network]
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
networks:
  agent_network:
    driver: overlay
    attachable: true
volumes:
  portainer_data:
```

Deploy or update it on cubie-1 with `docker stack deploy -c /opt/stacks/portainer.yml portainer`.

**First start:** create the admin account within about 5 minutes, or Portainer locks the setup page until it's restarted. The page asks for a setup token, which is in the service log:

```sh
docker service logs portainer_portainer 2>&1 | grep -o 'setup_token=[0-9a-f]*' | tail -1
```

The admin password needs at least 12 characters. Skip the Edge Compute screen: every node is already reachable through the agents.

Portainer LTS needs Docker API 1.51 or newer. Debian's own `docker.io` 26 offers API 1.45, which is why the cubies run Docker CE.

## Pi-hole

| | |
|---|---|
| Admin | <http://10.1.0.151/admin> or <http://cubie-2.local/admin> |
| Image | `pihole/pihole:latest`, container hostname `pihole` |
| Runs on | cubie-2 only (placement constraint) |
| Ports | 53/tcp, 53/udp and 80/tcp in host mode on cubie-2 |
| Upstream DNS | `1.1.1.1`, `1.0.0.1` |
| Data | Volume `pihole_pihole_etc` on cubie-2 (`/etc/pihole`: settings, lists, query database) |
| Stack file | `/opt/stacks/pihole.yml` on cubie-1, mode 600 because it holds the web password |

The stack file, with the password replaced:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    hostname: pihole
    environment:
      TZ: Asia/Bangkok
      FTLCONF_webserver_api_password: "<set a password>"
      FTLCONF_dns_listeningMode: all
      FTLCONF_dns_upstreams: "1.1.1.1;1.0.0.1"
    volumes:
      - pihole_etc:/etc/pihole
    ports:
      - {target: 53, published: 53, protocol: tcp, mode: host}
      - {target: 53, published: 53, protocol: udp, mode: host}
      - {target: 80, published: 80, protocol: tcp, mode: host}
    deploy:
      placement:
        constraints: [node.hostname == cubie-2]
      restart_policy: {condition: any}
volumes:
  pihole_etc:
```

- **Why host mode:** the ports bypass the routing mesh, which would NAT the source address, so Pi-hole sees where each query really came from.
- **Password:** if you change it in the web UI, put the same value in the stack file. Otherwise the next `docker stack deploy` resets it.
- **Client attribution:** LAN queries reach Pi-hole through cubie-1's dnsmasq, which adds the client's IP and MAC to each query. That's why the statistics still show individual devices.

### Blocklists

As of 2026-11-22:

| List | Domains | Enabled |
|---|---|---|
| [StevenBlack hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts) | 76,105 | Yes |
| [HaGeZi Pro](https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt): ads, trackers, telemetry | 283,999 | Yes |
| [HaGeZi Threat Intelligence, medium](https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.medium.txt): malware, phishing, scams | 412,890 | Yes |
| [HaGeZi Gambling](https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/gambling.txt) | 710,182 | Yes |
| [HaGeZi Encrypted DNS (DoH)](https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/doh.txt): DNS-over-HTTPS servers | 3,569 | Yes |
| [Faelayis ads](https://raw.githubusercontent.com/Faelayis/AdBlock-Thai-Filters/master/filters/ads.txt) | 22 | Yes |
| [thai-ads-filter subscription](https://adblock-thai.github.io/thai-ads-filter/subscription.txt) | 22 | Yes |
| [easylist-thailand](https://raw.githubusercontent.com/easylist-thailand/easylist-thailand/master/subscription/easylist-thailand.txt) | 0 | No: browser filter |
| [Faelayis banner](https://raw.githubusercontent.com/Faelayis/AdBlock-Thai-Filters/master/filters/banner.txt) | 0 | No: browser filter |
| [thai-ads-filter annoyance](https://adblock-thai.github.io/thai-ads-filter/annoyance.txt) | 0 | No: browser filter |
| [PhyschicWinter9 thai-adblock-list](https://raw.githubusercontent.com/PhyschicWinter9/thai-adblock-list/main/thai-adblock-list.txt) | 0 | No: browser filter |

Before the DoH list was added, gravity held about 1.32 million unique domains. Pi-hole used about 130 MB of RAM with these lists.

- **Thai lists:** almost all of them are cosmetic rules that hide banners inside web pages. DNS can't apply those, so they yield 0 domains. Use them in uBlock Origin in the browser, next to Pi-hole. HaGeZi Pro and Gambling already cover the ad networks and gambling sites that Thai sites use.
- **Don't add** PhyschicWinter9's `subscription/domains.txt`. It lists the sites the filters apply to (for example `googleusercontent.com`, URL shorteners and anime sites), not ad servers, so it would block whole sites.
- **HaGeZi Pro is fairly strict.** If an app breaks, allow its domain from Pi-hole's Query Log.

Rebuild gravity by hand (on cubie-2):

```sh
docker exec "$(docker ps -q -f name=pihole_pihole)" pihole -g
```

## Deploying an app

1. In Portainer, go to Stacks → Add stack, paste a Compose file (v3 format) and deploy it. Or, on cubie-1, run `docker stack deploy -c app.yml app`.
2. Keep apps off the router:

   ```yaml
   deploy:
     placement:
       constraints: [node.hostname != cubie-1]
   ```

3. Images must exist for the node's architecture. Pin amd64-only images to acer-1 with `node.platform.arch == x86_64`.
4. Put anything that needs more than a few hundred MB of RAM on acer-1. Each cubie has about 550–600 MB free.
