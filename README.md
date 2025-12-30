# Proxmox VE inside Docker

Run Proxmox in a container. Don't ask why.

## Requirements

- Modern Linux host with kernel 6.14+
- [Docker Engine](https://docs.docker.com/engine/install/) (obviously)
- A machine that supports virtualization
- Patience

## Quick Start

```bash
docker run -d --name proxmox --hostname proxmox \
    -p 2222:22 -p 3128:3128 -p 8006:8006 \
    --privileged --cgroupns=host -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
    -v /usr/lib/modules:/usr/lib/modules --restart unless-stopped  \
    -v ./VM-Backup:/var/lib/vz/dump \
    -v ./ISOs:/var/lib/vz/template/iso \
    ghcr.io/longqt-sea/proxmox-ve
```
Replace `./ISOs` with the path to your ISO folder.

Set root password and reboot the container at least once:
```
docker exec -it proxmox passwd
docker restart proxmox
```

## Docker Compose

Here is `docker-compose.yml` if you prefer:
```yaml
services:
  proxmox:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: proxmox
    hostname: proxmox
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "2222:22"
      - "3128:3128"
      - "8006:8006"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /usr/lib/modules:/usr/lib/modules
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso  # Replace with your ISO folder path
```
Bring it up:
```bash
docker compose up -d
```
Set root password and reboot the container at least once:
```
docker exec -it proxmox passwd
docker restart proxmox
```

## Ports

| Port | What it does |
|------|--------------|
| 8006 | Web UI |
| 3128 | SPICE proxy |
| 2222 | SSH |

## Access

Open `https://localhost:8006` in your browser. Accept the self-signed cert warning. Default login is `root` with whatever password you set.

## Volumes

| Host Path | Container Path | Purpose |
|-----------|----------------|---------|
| ./VM-Backup | /var/lib/vz/dump | VM backups |
| ./ISOs | /var/lib/vz/template/iso | ISO images |

## Networks

- `vmbr0` - Empty bridge, configure it yourself, maybe with macvlan or passthrough a physical NIC
- `vmbr1` - NAT network for VM (172.16.99.0/24), works out of the box

## Notes

This thing needs `--privileged`. Yes, really. It's running a hypervisor in a container.

If something breaks, it's probably a kernel module issue. Make sure `/usr/lib/modules` is mounted correctly.

## License

GPLv3 or later. See the Dockerfile.
