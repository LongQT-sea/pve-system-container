# Proxmox VE inside container
Proxmox cluster in Docker. Learn, test, break, repeat.

- **Fast iteration** — Spin up, tear down, repeat in seconds
- **Cluster simulation** — Test HA, failover, and live migration
- **Automation testing** — Validate Terraform, Ansible, or scripts
- **Shared storage** — Mount ISOs, backups, disk images volume across nodes
- **KVM and LXC** support out of the box

## Requirements

- A modern Linux host with kernel 6.8+
- [Docker Engine](https://docs.docker.com/engine/install/) (obviously)
- CPU with virtualization support (Intel VT-x / AMD-V)

## Quick Start

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

Set root password and reboot the container at least once:
```
docker exec -it pve-1 passwd
docker restart pve-1
```

Access the web UI at `https://localhost:8006` (accept the self-signed cert).

## Multi-Node Cluster with Docker Compose (recommended)
A 3-node HA cluster sharing ISOs and backups setup:
```
mkdir pve_cluster && cd pve_cluster

nano docker-compose.yml
```
Paste the content below into nano, save with Ctrl+X, Y, Enter.
```yaml
services:
  pve-1:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-1
    hostname: pve-1
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "2222:22"
      - "3128:3128"
      - "8006:8006"   # First node Web GUI is listening on 8006
    networks:
      - dual_stack
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder

  pve-2:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-2
    hostname: pve-2
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "2223:22"
      - "3129:3128"
      - "8007:8006"   # Second node Web GUI is listening on 8007
    networks:
      - dual_stack
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder

  pve-3:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-3
    hostname: pve-3
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "2224:22"
      - "3130:3128"
      - "8008:8006"   # Third node Web GUI is listening on 8008
    networks:
      - dual_stack
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace ./ISOs with the path to your ISO folder

networks:
  dual_stack:
    enable_ipv6: true
    driver: bridge
    ipam:
      config:
        - subnet: fd00::/48
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
Reboot all nodes at least once:
```
docker restart pve-1 pve-2 pve-3
```
Tear down cluster:
```
docker compose down
```

## Ports

| Port | Purpose |
|------|--------------|
| 8006 | Web UI |
| 3128 | SPICE proxy |
| 22 | SSH |

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

## License

GPLv3 or later. See the Dockerfile.
