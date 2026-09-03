# Disaster Recovery

## Replace an ARM Node (Nodes 1–3)

1. Re-flash compatible OS image
2. Restore expected hostname and network role
3. Reapply node-specific service config
4. Rejoin swarm if the node is part of swarm control/agent path
5. Validate service and network health

## Replace Node 4 (x86_64 Worker)

1. Provision host with matching OS and Docker version
2. Join swarm as worker using current join token
3. Reapply labels/constraints required by deployed stacks
4. Confirm workloads reschedule successfully

## Full Rebuild

1. Bring up Node 1 networking first (bridge, DHCP, NAT, Wi-Fi)
2. Restore Node 2 DNS/Pi-hole
3. Restore Node 3 swarm manager + Portainer
4. Add Node 4 worker and redeploy workload stacks
5. Run full connectivity and service validation

## Backup Recommendations

- Export stack definitions and environment configuration
- Back up Pi-hole configuration and query database
- Keep node role configuration files under version control where possible
- Maintain periodic snapshots of critical data volumes

