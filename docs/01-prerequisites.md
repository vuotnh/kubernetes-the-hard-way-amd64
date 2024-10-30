# Prerequisites

In this lab you will review the machine requirements necessary to follow this tutorial.

## Virtual or Physical Machines

This tutorial requires four (4) virtual or physical AMD64 machines running Ubuntu 22.04 LTS. The follow table list the four machines and thier CPU, memory, and storage requirements.

| Name    | Description            | CPU | RAM   | Storage | IP address |
|---------|------------------------|-----|-------|---------|---------|
| jumpbox | Deployment host    | 1   | 512MB | 10GB    | 192.168.1.2 |
| server  | Kubernetes server      | 1   | 2GB   | 20GB    | 192.168.1.3 |
| node-0  | Kubernetes worker node | 1   | 2GB   | 20GB    | 192.168.1.4 |
| node-1  | Kubernetes worker node | 1   | 2GB   | 20GB    | 192.168.1.5 |
| backup (for HA) | Kubernetes backup master | 1 | 2GB | 20GB | 192.168.1.6 |

Next: [setting-up-the-jumpbox](02-jumpbox.md)
