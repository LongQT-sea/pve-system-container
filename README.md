# Proxmox VE and Datacenter Manager inside container
Proxmox cluster in Docker. Learn, test, break, repeat.

- **Fast iteration** — Spin up, tear down, repeat in seconds
- **Cluster simulation** — Test HA, failover, and live migration
- **Automation testing** — Validate Terraform, Ansible, or scripts
- **Shared storage** — Mount ISOs, backups, disk images volume across nodes
- **KVM and LXC** — Work out of the box

---

## Requirements

- A modern Linux host with kernel 6.8+
- [Docker Engine](https://docs.docker.com/engine/install/)
- CPU with virtualization support (Intel VT-x / AMD-V)

---

## Quick Start
Standalone node with `docker run`:
```bash
docker run -d --name pve-1 --hostname pve-1 \
    -p 2222:22 -p 3128:3128 -p 8006:8006 \
    --restart unless-stopped  \
    --privileged --cgroupns=host -v /sys/fs/cgroup:/sys/fs/cgroup \
    -v /usr/lib/modules:/usr/lib/modules:ro \
    -v /sys/kernel/security:/sys/kernel/security \
    -v ./VM-Backup:/var/lib/vz/dump \
    -v ./ISOs:/var/lib/vz/template/iso \
    ghcr.io/longqt-sea/proxmox-ve
```
Replace `./ISOs` with the path to your ISO folder.

Set root password and restart the container at least once:
```
docker exec -it pve-1 passwd
docker restart pve-1
```

Access the web UI at `https://localhost:8006` (accept the self-signed cert).

---

## Multi-Node Cluster with Docker Compose (recommended)
A 3-node HA cluster sharing ISOs and backups setup:
```
mkdir pve_cluster && cd pve_cluster

nano docker-compose.yml
```
Paste the content below into nano, save with Ctrl+X, Y, Enter.
```yaml
services:
  # First node
  pve-1:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-1
    hostname: pve-1
    privileged: true
    restart: unless-stopped
    cgroup: host
    networks:
      - dual_stack
    
    # Port mapping only required for Docker Desktop or LAN access from other machines.
    # On Linux host, you can access this node directly via hostname or IP address, e.g. https://pve-1:8006 or https://[fd00::1]:8006
    ports:
      - "2222:22"
      - "3128:3128"
      - "8006:8006"   # First node Web GUI is listening on localhost:8006
    
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder


  # Second node
  pve-2:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-2
    hostname: pve-2
    privileged: true
    restart: unless-stopped
    cgroup: host
    networks:
      - dual_stack
    
    # Port mapping only required for Docker Desktop or LAN access from other machines.
    # On Linux host, you can access this node directly via hostname or IP address, e.g. https://pve-2:8006 or https://[fd00::2]:8006
    ports:
      - "2223:22"
      - "3129:3128"
      - "8007:8006"   # Second node Web GUI is listening on localhost:8007
    
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder


  # Third node
  pve-3:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-3
    hostname: pve-3
    privileged: true
    restart: unless-stopped
    cgroup: host
    networks:
      - dual_stack
    
    # Port mapping only required for Docker Desktop or LAN access from other machines.
    # On Linux host, you can access this node directly via hostname or IP address, e.g. https://pve-3:8006 or https://[fd00::3]:8006
    ports:
      - "2224:22"
      - "3130:3128"
      - "8008:8006"   # Third node Web GUI is listening on localhost:8008
    
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder


  # Optional: Proxmox Datacenter Manager
  pdm:
    image: ghcr.io/longqt-sea/proxmox-datacenter-manager
    container_name: pdm
    hostname: pdm
    restart: unless-stopped
    cgroup: host
    security_opt:
      - seccomp=unconfined
      - apparmor=unconfined
    cap_add:
      - ALL
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    networks:
      - dual_stack
    
    # Port mapping only required for Docker Desktop or LAN access from other machines.
    # On Linux host, you can access this container directly via hostname or IP address, e.g. https://pdm:8443 or https://[fd00::4]:8443
    ports:
      - "2225:22"
      - "8443:8443"

# Dual-stack network for this cluster
networks:
  dual_stack:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 10.0.99.0/24
          gateway: 10.0.99.99
        - subnet: fd00::/64
          gateway: fd00::99
```
Bring it up:
```
docker compose up -d
```
Set root password for all nodes:
```
docker exec -it pve-1 passwd
docker exec -it pve-2 passwd
docker exec -it pve-3 passwd
```
Restart all nodes at least once:
```
docker restart -t 3 pve-1 pve-2 pve-3
```

Each node can reach each other over hostname or IP address:
| hostname | IPv4       | IPv6    |
|----------|------------|---------|
| pve-1    | 10.0.99.1  | fd00::1 |
| pve-2    | 10.0.99.2  | fd00::2 |
| pve-3    | 10.0.99.3  | fd00::3 |
| pdm      | 10.0.99.4  | fd00::4 |

To tear down the cluster:
```
docker compose down -t 0
```

---

## Ports

| Port | Purpose |
|------|--------------|
| 8006 | Web UI |
| 3128 | SPICE proxy |
| 22 | SSH |
| 8443 | PDM Web UI |

## Volumes

| Host Path | Container Path | Purpose |
|-----------|----------------|---------|
| ./VM-Backup | /var/lib/vz/dump | VM backups |
| ./ISOs | /var/lib/vz/template/iso | ISO images |

## Networks

- `vmbr1` - NAT network for VM and LXC, works out of the box
- `vmbr2` - Empty bridge, configure it yourself, maybe with macvlan, veth or passthrough a physical NIC

---

> [!Note]
> When running with `podman`, make sure to run as root or with `sudo`, rootless Podman does not work even with `--privileged`.

> [!Warning]
> This setup uses the `--privileged` flag. The container can do almost everything the Linux host can do. Use with caution.

---

## License

This project is licensed under the GPLv3 or later (see [LICENSE](LICENSE) file).

## Disclaimer

This project is provided “as‑is”, without any warranty, for educational and research purposes. In no event shall the authors or contributors be liable for any direct, indirect, incidental, special, or consequential damages arising from use of the project, even if advised of the possibility of such damages.

All product names, trademarks, and registered trademarks are property of their respective owners. All company, product, and service names used in this repository are for identification purposes only.
