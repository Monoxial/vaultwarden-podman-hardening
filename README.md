# Vaultwarden Rootless Deployment with Podman Quadlet

This guide explains how to deploy Vaultwarden as a rootless Podman container using Quadlet and systemd user services.

## Requirements

* Linux system with systemd
* Podman installed
* Quadlet support enabled
* Root access for initial setup

## Create a Dedicated User

Create a dedicated system user for Vaultwarden:

```bash
sudo useradd -r -m -d /var/lib/vaultwarden -s /usr/sbin/nologin vaultwarden
sudo loginctl enable-linger vaultwarden
id vaultwarden
```

Note the UID returned by the `id` command.

## Configure Rootless UID/GID Mapping

Rootless containers require subordinate UID/GID ranges.

```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 vaultwarden
```

Verify:

```bash
grep '^vaultwarden:' /etc/subuid /etc/subgid
```

Start the user manager:

```bash
sudo systemctl start user@<UID>.service
```

Replace `<UID>` with the Vaultwarden user UID.

> [!NOTE]
> This step is only required for the very first setup before the next reboot.
> Once `loginctl enable-linger` is active, the user manager starts automatically
> at boot and this command becomes unnecessary.

## Create Required Directories

```bash
sudo mkdir -p /var/lib/vaultwarden/data
sudo mkdir -p /var/lib/vaultwarden/.config/containers/systemd
sudo chown -R vaultwarden:vaultwarden /var/lib/vaultwarden
```

## Create Environment File

Create the Vaultwarden environment file outside the data volume to keep
configuration and runtime data separated:

```bash
sudo -u vaultwarden touch /var/lib/vaultwarden/vaultwarden.env
sudo -u vaultwarden chmod 600 /var/lib/vaultwarden/vaultwarden.env
sudo -u vaultwarden nano /var/lib/vaultwarden/vaultwarden.env
```

Example:

```env
DOMAIN=https://vault.example.com
SIGNUPS_ALLOWED=false
```

## Create Quadlet Container Definition

Create the Quadlet file:

```bash
sudo -u vaultwarden nano /var/lib/vaultwarden/.config/containers/systemd/vaultwarden.container
```

Add the following configuration:

```ini
[Unit]
Description=Vaultwarden rootless container
After=network-online.target
Wants=network-online.target

[Container]
ContainerName=vaultwarden
Image=ghcr.io/dani-garcia/vaultwarden:latest
EnvironmentFile=/var/lib/vaultwarden/vaultwarden.env

Volume=/var/lib/vaultwarden/data:/data:Z,rw,nodev,nosuid
PublishPort=127.0.0.1:8080:80

DropCapability=all
AddCapability=NET_BIND_SERVICE
NoNewPrivileges=true

Tmpfs=/tmp:rw,noexec,nodev,nosuid,size=64m
Tmpfs=/run:rw,noexec,nodev,nosuid,size=16m

PidsLimit=256
Memory=512m
Ulimit=nofile=1024:4096

Notify=false
LogDriver=journald

[Service]
Restart=on-failure
RestartSec=10
TimeoutStartSec=120
TimeoutStopSec=30

[Install]
WantedBy=default.target
```

> [!NOTE]
> This configuration works with a few users. Adjust it to fit your environment

## Load systemd User Environment

Open a shell as the Vaultwarden user:

```bash
sudo -u vaultwarden bash
```

Export the required environment variables:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=${XDG_RUNTIME_DIR}/bus
```

> [!NOTE]
> To avoid manually exporting systemd user environment variables every time a shell is opened as the Vaultwarden user, add them to the user profile

## Start the Service

Reload the user systemd manager so Quadlet generates the unit, then start it:

```bash
systemctl --user daemon-reload
systemctl --user start vaultwarden.service
```

The service will start automatically on subsequent boots thanks to
`loginctl enable-linger` and the `[Install]` section of the Quadlet file.

## Verify Deployment

Check service status:

```bash
systemctl --user status vaultwarden.service --no-pager -l
```

Check containers:

```bash
podman ps
```

View logs:

```bash
podman logs vaultwarden
```

Test local access:

```bash
curl -I http://127.0.0.1:8080
```

## Service Management

Stop the service:

```bash
systemctl --user stop vaultwarden.service
```

Restart the service:

```bash
systemctl --user restart vaultwarden.service
```

View systemd logs:

```bash
journalctl --user -u vaultwarden.service -n 100 --no-pager
```

Update the container image:

```bash
podman pull ghcr.io/dani-garcia/vaultwarden:latest
systemctl --user restart vaultwarden.service
```

## Root-Level Service Control

To control the Vaultwarden service from a root context (cron job, automation
script, ansible task...) without restarting the entire user manager, run
`systemctl --user` as the target user with the proper runtime environment:

```bash
VW_UID=$(id -u vaultwarden)
sudo -u vaultwarden \
    XDG_RUNTIME_DIR=/run/user/$VW_UID \
    systemctl --user restart vaultwarden.service
```

Alternative using `machinectl` (requires `systemd-container`):

```bash
sudo machinectl shell vaultwarden@.host /bin/bash -c \
    'systemctl --user restart vaultwarden.service'
```

> [!WARNING]
> Avoid `sudo systemctl restart user@<UID>.service`: this restarts the whole
> user manager instance and every other rootless service owned by that user,
> not just Vaultwarden.

## Security Notes

* Vaultwarden runs as a rootless container.
* All Linux capabilities are dropped by default.
* `NET_BIND_SERVICE` is added to allow binding to port 80 inside the container.
* Service exposure is limited to `127.0.0.1:8080`.
* `/tmp` and `/run` are isolated using hardened tmpfs mounts.
* The data volume uses `nodev,nosuid` mount options.
* `NoNewPrivileges=true` prevents privilege escalation.
* Podman networking is handled by rootless user namespaces.
* The environment file is stored outside the data volume with `chmod 600`,
  preventing it from being included in routine `/data` backups.

## Automatic Startup

Because `loginctl enable-linger` is enabled, the Vaultwarden user service starts automatically after reboot.

Verify:

```bash
loginctl show-user vaultwarden
```

Expected:

```text
.
.
.
Linger=yes
```
