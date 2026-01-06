# Proxmox Datacenter Manager inside container

Proxmox Datacenter Manager in Docker, don't ask why!

## Quick start
With `docker run`:
```bash
docker run -d --name pdm --hostname pdm \
   -p 8443:8443 -p 2222:22 \
   --restart unless-stopped \
   --cgroupns=host \
   -v /sys/fs/cgroup:/sys/fs/cgroup \
   --security-opt=seccomp=unconfined \
   --security-opt=apparmor=unconfined \
   --cap-add=ALL \
   ghcr.io/longqt-sea/proxmox-datacenter-manager
```

Set root password:
```
docker exec -it pdm passwd
```

With `docker compose`:
```yaml
services:
  pdm:
    image: ghcr.io/longqt-sea/proxmox-datacenter-manager
    container_name: pdm
    hostname: pdm
    restart: unless-stopped
    cgroup: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup
    cap_add:
      - ALL
    security_opt:
      - seccomp=unconfined
      - apparmor=unconfined
    ports:
      - "2222:22"
      - "8443:8443"
```

Set root password:
```
docker exec -it pdm passwd
```

Access the web UI at https://Docker-IP:8443 (accept the self-signed cert).
