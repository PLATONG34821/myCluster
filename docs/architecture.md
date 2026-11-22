# Architecture

## Node Roles

| Node | Hostname | IP | Role |
|---|---|---|---|
| Node 1 | `cubie-network` | `10.1.0.1` | Gateway, bridge, DHCP, Wi-Fi AP |
| Node 2 | `cubie-dns` | `10.1.0.2` | Pi-hole DNS/ad-blocking |
| Node 3 | `cubie-manager` | `10.1.0.3` | Swarm manager + Portainer server |
| Node 4 | `veriton-worker-01` | `10.1.0.4` | Main x86_64 workload runner |

## Network Model

- Node 1 is the only WAN ingress node
- Internal cluster subnet: `10.1.0.0/24`
- Wired LAN and Wi-Fi are bridged through `br0`
- NAT masquerading on Node 1 provides outbound internet access
- DHCP clients receive Node 2 (`10.1.0.2`) as DNS resolver

## Key Design Decisions

1. **Central DHCP on Node 1** to guarantee deterministic leases
2. **Dedicated DNS node** so ad-blocking and name resolution are independent
3. **Manager in drain mode** to avoid placing user workloads on constrained ARM manager hardware
4. **Heavy services on Node 4** for better CPU/RAM capacity
5. **Portainer via Swarm stack + agents** for cluster-wide visibility

