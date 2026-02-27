# New Quadlet Setup Guide

A reference for setting up a new quadlet service from scratch.
These instructions are derived from the [README](./README.md).

## General remarks

- I want to use regular users
  + the home directory should be in /var/lib/
- I want to execute commands with my regular user (with sudo), e.g. `sudo -u <service_name> ...`
- Clone the repo into the service user's home directory for isolation
- Symlink `.container`, `.env`, and `.override.env` from the repo into `~service_name/.config/containers/systemd/`
- For `.override.env`: copy the `.override.env.template` into the repo directory (not the systemd directory), edit it there, then symlink it — add it to `.gitignore` to keep secrets out of version control
- Please create a markdown file with instructions
  + use `REPO_URL` and `REPO` shell variables so the admin sets the clone location once at the top

## Information to gather about the image

Before starting, resolve the following:

| # | Question | Used for |
|---|----------|----------|
| 1 | What is the fully qualified image name (registry/image:tag)? | `Image=` |
| 2 | What UID does the image run as? | `UserNS=`, `User=`, `Group=` |
| 3 | Does the image support `PUID`/`PGID` env vars? | Alternative to adjusting `UserNS` |
| 4 | What port(s) does the container expose? | `PublishPort=` |
| 5 | What paths inside the container need to be persisted? (config, data) | `Volume=` |
| 6 | What environment variables does the image require or support? | `.env` file |
| 7 | Is there a health check endpoint? | `HealthCmd=` |
| 8 | Does the service consist of multiple containers? | `.network` file needed |
| 9 | Will the service sit behind a reverse proxy? | bind `PublishPort` to `127.0.0.1`, create DNS entry |
| 10 | How long can the container take to start? | `TimeoutStartSec=` |

Check the image's UID:

```sh
podman inspect <image> --format '{{.Config.User}}'
```

If this returns a username instead of a numeric ID, look it up in `/etc/passwd` inside the container:

```sh
podman run --rm --entrypoint grep <image> <username> /etc/passwd
```

## Setup commands

Replace `service_name` and `<repo>` throughout.

```sh
REPO_URL=https://github.com/<user>/<repo>.git
REPO=~service_name/<repo>
```

```sh
# 1. Create service user
sudo useradd -m -d /var/lib/service_name -s /usr/sbin/nologin service_name

# 2. Enable linger
sudo loginctl enable-linger service_name

# 3. Clone the repo into the service user's home
sudo -u service_name git clone $REPO_URL $REPO

# 4. Create quadlet directory
sudo -u service_name mkdir -p ~service_name/.config/containers/systemd

# 5. Create data and config directories
sudo -u service_name mkdir -p ~service_name/{data,config}

# 6. Create .override.env from template and fill in required values
sudo -u service_name cp $REPO/service_name.override.env.template $REPO/service_name.override.env
sudo -u service_name nano $REPO/service_name.override.env

# 7. Symlink .container, .env, and .override.env from the repo
sudo -u service_name ln -s $REPO/service_name.container ~service_name/.config/containers/systemd/service_name.container
sudo -u service_name ln -s $REPO/service_name.env ~service_name/.config/containers/systemd/service_name.env
sudo -u service_name ln -s $REPO/service_name.override.env ~service_name/.config/containers/systemd/service_name.override.env

# 8. Reload and start
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user daemon-reload
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user start service_name

# 9. Verify
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user status service_name
```

## `.container` file template

```ini
[Unit]
Description=<description>
After=network-online.target
Wants=network-online.target

[Container]
Image=<registry/image:tag>
AutoUpdate=registry

# Set uid/gid to match the UID the image runs as
UserNS=keep-id:uid=<uid>,gid=<gid>
User=<uid>
Group=<gid>

# Environment variables
EnvironmentFile=%h/.config/containers/systemd/service_name.env
EnvironmentFile=%h/.config/containers/systemd/service_name.override.env

# Volumes — :Z for SELinux relabel, :ro where write access is not needed
Volume=%h/config:<config path in container>:Z,ro
Volume=%h/data:<data path in container>:Z

# Port mapping — bind to 127.0.0.1 when behind a reverse proxy
PublishPort=127.0.0.1:<host port>:<container port>

# Health check
HealthCmd=curl -f http://localhost:<container port>/<health endpoint> || exit 1
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

## UID mismatch

If the image UID differs from `1000`, set `UserNS`, `User`, and `Group` to match it.
If the image supports `PUID`/`PGID`, set them to `1000` in the `.env` file instead.
As a last resort, use `podman unshare chown`:

```sh
sudo -u service_name podman unshare chown -R <uid>:<gid> ~service_name/data
```

## Multi-container services

Create a `.network` file alongside the `.container` files:

```sh
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.network
```

```ini
[Network]
```

Reference it in each `.container` file:

```ini
Network=service_name.network
ContainerName=service_name_app   # optional — default is systemd-<filename>
```

Containers on the same network reach each other by `ContainerName` (or `systemd-<filename>`).

## Backup

The backup strategy is:
- A shared `backup-readers` group and a `backupuser` (login shell) are created once per server (see README)
- Each service writes a backup to `/var/backups/service_name/` and owns that directory
- `backupuser` has read access via the `backup-readers` group
- A remote machine pulls via `rsync` over SSH as `backupuser`

### General remarks

- Create a `service_name-backup.service` (Type=oneshot) and `service_name-backup.timer` (OnCalendar=daily, Persistent=true)
- Use the full path to the backup executable (e.g. `/usr/bin/sqlite3`) in `ExecStart`
- Symlink both files into `~service_name/.config/systemd/user/` (not `.config/containers/systemd/`)
- The backup command depends on the service: `sqlite3 .backup` for SQLite, `pg_dump` for PostgreSQL, `cp`/`rsync` for files

### Per-service setup commands

```sh
# Create backup staging directory
sudo mkdir -p /var/backups/service_name
sudo chown service_name:backup-readers /var/backups/service_name
sudo chmod 750 /var/backups/service_name

# Symlink backup units from the repo
sudo -u service_name mkdir -p ~service_name/.config/systemd/user
sudo -u service_name ln -s $REPO/service_name-backup.service ~service_name/.config/systemd/user/service_name-backup.service
sudo -u service_name ln -s $REPO/service_name-backup.timer ~service_name/.config/systemd/user/service_name-backup.timer

# Enable and start the timer
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user daemon-reload
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user enable --now service_name-backup.timer
```

## Image pruning

The system-wide template units (`podman-image-prune@.timer` and `podman-image-prune@.service`) are assumed to already exist in `/etc/systemd/user/`. Each service user enables their own instance, with the number of days to retain as the instance name.

```sh
# Enable and start the timer (replace 30 with the desired retention period in days)
sudo -u service_name XDG_RUNTIME_DIR=/run/user/$(id -u service_name) systemctl --user enable --now podman-image-prune@30.timer
```
