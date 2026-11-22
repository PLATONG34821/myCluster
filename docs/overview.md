# Project Overview

## What This Project Is

This cluster is a self-contained, portable computing environment that can be moved to different locations and brought online with minimal reconfiguration. It combines low-power ARM boards and a more capable x86_64 host under Docker Swarm.

## Use Cases

- Portable lab and experimentation environment
- Edge workload hosting (including future AI inference workloads)
- On-site network utility stack (DHCP, DNS, ad-blocking, Wi-Fi)
- Hands-on container orchestration practice with mixed architectures

## Design Principles

### 1) Portability First

All nodes run on a fixed internal subnet (`10.1.0.0/24`) with stable DHCP lease mapping so topology stays predictable after relocation.

### 2) Separation of Concerns

Each node has a primary role (gateway, DNS, manager, worker) to keep operations clear and failures isolated.

### 3) Mixed-Architecture Awareness

ARM and x86_64 nodes coexist in one swarm, so workloads must use multi-arch images or explicit placement constraints.

