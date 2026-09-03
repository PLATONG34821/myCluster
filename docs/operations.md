# Operations

## Prerequisites

- All nodes physically connected to the switch
- Node 1 has WAN connectivity on `eth1`
- Docker installed on every node
- Time sync and hostname configuration completed

## Startup Flow

1. Boot Node 1 and verify `br0`, DHCP, and Wi-Fi AP are active
2. Boot Node 2 and confirm DNS service health
3. Boot Node 3 and confirm swarm manager health
4. Boot Node 4 and verify it joins as swarm worker
5. Confirm Portainer availability and agent connectivity

## Health Checks

- Validate node IP assignments (`10.1.0.1` to `10.1.0.4`)
- Validate DNS path through Pi-hole
- Check swarm node state and service replica health
- Verify external internet egress from cluster clients

## Day-to-Day Tasks

- Deploy/update stacks from Portainer or Swarm CLI
- Monitor service health and node resource usage
- Perform Pi-hole list and query management
- Drain/activate nodes before and after maintenance

