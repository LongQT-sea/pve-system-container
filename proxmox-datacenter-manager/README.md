# Proxmox Datacenter Manager inside container

Proxmox Datacenter Manager in Docker, don't ask why!

## Quick start
```
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

Access the web UI at https://Docker-IP:8443 (accept the self-signed cert).
