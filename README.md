# vps-config

Rootless Podman setup utilizing systemd Quadlets and socket activation.

## Prerequisites

Allow binding to ports 80/443 without root and reload configuration:
```bash
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-unprivileged-ports.conf
sudo sysctl --system
```

Enable lingering so services start on boot and stay alive:
```bash
sudo loginctl enable-linger $USER
```

## Deployment

Reload systemd and start the services:
```bash
systemctl --user daemon-reload
systemctl --user enable --now caddy.socket ntfy.service whoami.service
```
