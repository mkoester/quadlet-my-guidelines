# Quadlet: my guidelines

## About

In this document I want to collect my guidelines / best practises when it comes to hosting services on a (virtual) server.
I prefer to use **Fedora**, but this should work on all (linux) systems supporting **Podman** and **quadlets**.
You need to be able to run commands via **sudo** to set everything up (or run every service with your user account, which I would not recommend).
In this document I am going to use **nano** as the *file editor*, but every other editor like **vim** or **emacs** would also work similarly.

One could also use other means to achieve the same (e.g. via `docker compose`), but I personally prefer `quadlets` (and `podman` + `systemd`).

This document is about setting up the service itself. In order to access the service I usually set up [**Caddy**](./reverse_proxy/Caddyfile#L32-L35) or **Nginx** as a *reverse proxy*.

## Setup overview

Setting up a new service follows these steps in order:

1. **Create a dedicated service user** with a home directory under `/var/lib`
2. **Enable linger** so the user's processes persist without an active login session
3. **Create the quadlet directory** `~service_name/.config/containers/systemd`
4. **Create data and config directories** under the service user's home
5. **Write the `.container` file** (and optionally a `.network` file)
6. **Create environment variable files** (`.env` and `.override.env`)
7. **Reload the systemd daemon** so it picks up the new quadlet files
8. **Start the service** and verify it is running

Each step is described in detail in the sections below.

## Service users

For best isolation, I would create one dedicated user account per service I want to run.
One could use service accounts, but they usually don't have the necessary ranges for user and group sub IDs created automatically (needed to run rootless podman containers).
Therefore, we use regular users and create their home directory in [/var/lib](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s08.html).

E.g. for a service called <service_name>, we would create a user with *no login shell*

```sh
sudo useradd -m -d /var/lib/service_name -s /usr/sbin/nologin service_name
```

In order for the service to be able to run in the user's context when the user itself is not logged in, we have to allow its processes to persist

```sh
sudo loginctl enable-linger service_name
```

Since we need to create and store the quadlet service definitions, we create the proper directory as follows:

```sh
sudo -u service_name mkdir -p ~service_name/.config/containers/systemd
```

## Permanent storage

Most services need both a **configuration** and some **space on disk** to store its *state*.
The setup depends heavily on the service we want to run.
Some services consist of multiple containers which in most cases need their own configuration and storage.

### One single container

For the simple case, we would simply create 2 directories, one for configuration and one for persisting the service's data.

```sh
sudo -u service_name mkdir -p ~service_name/{data,config}
```

### Several containers

In case a service requires several containers to operate, I prefer to create one directory per container.
Inside these I would then create the necessary subdirectories for each container individually, e.g.

```sh
sudo -u service_name mkdir -p ~service_name/container_name_1/{data,config}
sudo -u service_name mkdir -p ~service_name/container_name_2/{data,config,cache}
```

## Service definition / quadlet files

The quadlet files should be in `~service_name/.config/containers/systemd`. The main configuration is done in a `.container` file (or several in case of more complex setups) and an optional `.network` file.

```sh
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.container
```

### Container definition

We'll start with the `Unit` section with a customized description:

```ini
[Unit]
Description=<description of our service>
After=network-online.target
Wants=network-online.target
```

Next, we'll define the service itself in the `Container` section:

```ini
[Container]
Image=<fully qualified name of image>
AutoUpdate=registry

# User Namespace Mapping - Container UID 1000 = Host UID (service_name)
UserNS=keep-id:uid=1000,gid=1000
User=1000
Group=1000

# config and state
Volume=%h/...

# port mapping
PublishPort=127.0.0.1:1234:5678

# Health check
HealthCmd=... || exit 1
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3
```

- `Image` specifies the image our container should be using. If we want to make use of automatic updates, the image has to be fully qualified (including registry and tag).
- `AutoUpdate` is set to `registry` ([details](https://docs.podman.io/en/v5.0.1/markdown/podman-auto-update.1.html))
- in case the image supports it, we explicitly set the user namespace mapping. This allows the executable to run with the same permissions as the service user. Whether this works and which user/group id to use depends on the image.
- The `Volume`s we use also depend on the image. I prefer to use **bind mounts** instead of (anonymously) named volumes
- `PublishPort` has to be set according to the port you want to use locally and the port the image uses internally. When the service sits behind a reverse proxy, bind to localhost only to prevent direct external access.
- `HealthCmd` has to be configured specifically for the image. Often a call via `curl` works.

### Auto start at boot / restart

In order to let the system automatically start our service, we would add an [`Install` section](https://man7.org/linux/man-pages/man5/systemd.unit.5.html#[INSTALL]_SECTION_OPTIONS) to the `.container` file and tell systemd that this service should be run by default:

```ini
[Install]
WantedBy=default.target
```

Additionally we want the service to be restarted automatically:

```ini
[Service]
Restart=always
TimeoutStartSec=900
```

(The value for `TimeoutStartSec` depends on the individual container image)

### Network definition (`.network` file)

By default, each rootless podman container runs in isolation and cannot reach other containers by name. If a service consists of multiple containers that need to communicate — for example an application container and a database — a shared network is needed.

A `.network` file defines a named podman network that containers can join:

```sh
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.network
```

The file itself typically needs no content beyond the section header:

```ini
[Network]
```

Each container that should be part of the network references it by filename (without the `.network` extension):

```ini
[Container]
Network=service_name.network
```

Containers on the same network can address each other by name. The name used is either the explicitly set `ContainerName`:

```ini
ContainerName=service_name_app
```

or, if omitted, the default name that quadlet derives from the unit file: `systemd-<filename_without_extension>`. For example, `service_name_app.container` gets the container name `systemd-service_name_app`.

Either way, the app container can connect to the database at `service_name_db:5432` (or `systemd-service_name_db:5432`) without any port being exposed on the host.

Without a `.network` file, inter-container communication would require exposing ports on the host and routing traffic through `localhost`, which is both less clean and less secure.

A complete example combining all sections can be found in [example_service.container](example_service.container).

## Configuration & storage

### Bind mounts / volumes

Bind mounts are defined in the `Container` section using the `Volume` key:

```ini
Volume=<host_path>:<container_path>:<options>
```

The `%h` specifier can be used as a shorthand for the service user's home directory.

#### SELinux

On SELinux-enabled systems (Fedora, RHEL), the container process is denied access to bind-mounted host directories by default, even if the file permissions are correct. You must add a relabel option to the mount:

- `:Z` — relabels the host path as private to this container (use for single-container mounts)
- `:z` — relabels the host path as shared (use when multiple containers access the same directory)

Note that `:Z` recursively relabels the host directory, so only apply it to directories owned by the service user.

#### Configuration file(s)

Configuration directories or files should be mounted read-only where possible:

```ini
Volume=%h/config:/etc/service_name:Z,ro
```

#### Data directories

Data and state directories are mounted writable (the default):

```ini
Volume=%h/data:/var/lib/service_name:Z
```

#### Container runs as a different UID

`UserNS=keep-id:uid=1000,gid=1000` maps container UID `1000` to the host service user. If the image runs as a different UID (e.g. `472` for Grafana, `999` for some databases), the bind-mounted directories will be inaccessible to the container process because their host-side ownership does not match.

First check which UID the image uses:

```sh
sudo -u service_name podman inspect <image> --format '{{.Config.User}}'
```

**Preferred: adjust the `UserNS` mapping.** Update the `[Container]` section to map the image's UID to the host service user instead:

```ini
UserNS=keep-id:uid=<uid>,gid=<gid>
User=<uid>
Group=<gid>
```

This maps the container's process UID directly to the host service user, so bind-mounted directories are accessible without any ownership changes.

**Alternative: use `PUID`/`PGID` env vars.** Some images support configuring their runtime user via environment variables. Setting `PUID=1000` and `PGID=1000` in the `.env` file keeps the original `UserNS` mapping intact.

**Fallback: `podman unshare chown`.** If `UserNS` cannot be used, container UIDs map to subUIDs in the host's subUID range rather than the service user directly. In that case, use `podman unshare` to chown the directories within the user namespace:

```sh
sudo -u service_name podman unshare chown -R <uid>:<gid> ~service_name/data
```

### Environment variables

Environment variables are passed to the container via `.env` files referenced in the `Container` section:

```ini
[Container]
EnvironmentFile=%h/.config/containers/systemd/service_name.env
EnvironmentFile=-%h/.config/containers/systemd/service_name.override.env
```

The `-` prefix on the second file makes it optional — systemd will not fail if it is absent.

The convention I use is two files:

- `service_name.env` — checked into version control (or at least documented), prefilled with sensible defaults and non-sensitive values
- `service_name.override.env` — not checked in, contains secrets and environment-specific overrides

Create them as the service user:

```sh
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.env
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.override.env
```

Both files use standard `KEY=value` syntax:

```ini
TZ=Europe/Berlin
UID=1000
GID=1000
```

## Operations

### Starting the service

Before starting the service for the first time (and after any changes to the `.container` or `.network` files), reload the systemd daemon so it picks up the generated unit:

```sh
sudo -u service_name systemctl --user daemon-reload
```

Then start the service:

```sh
sudo -u service_name systemctl --user start service_name
```

### Auto update

`AutoUpdate=registry` in the container file only takes effect if the `podman-auto-update.timer` is active for the service user. Enable and start it with:

```sh
sudo -u service_name systemctl --user enable --now podman-auto-update.timer
```

By default this runs a daily check and pulls updated images, restarting affected containers. To trigger an update manually:

```sh
sudo -u service_name podman auto-update
```

### Backup

The `data` directory contains all persistent state and is the primary target for backups. For file-based services a live backup is usually safe; for databases it is better to stop the service first to avoid an inconsistent snapshot:

```sh
sudo -u service_name systemctl --user stop service_name
rsync -a ~service_name/data/ /path/to/backup/
sudo -u service_name systemctl --user start service_name
```

The `config` directory should also be backed up, though it typically changes less frequently.

## Frequently used commands

### Recreate the systemd service definition after changes to the `.container` or `.network` files

```sh
sudo -u service_name systemctl --user daemon-reload
```

### Restart the service(s) / check the status

```sh
sudo -u service_name systemctl --user restart service_name
```

```sh
sudo -u service_name systemctl --user status service_name
```

### Check the logs

```sh
sudo -u service_name journalctl --user -u service_name -n 50
```

```sh
sudo -u service_name podman logs -f service_name
```

## Debugging

### Validate quadlet parsing

To check that a `.container` (or `.network`) file is parsed correctly without applying any changes, run the quadlet generator in dry-run mode:

```sh
sudo -u service_name /usr/lib/systemd/system-generators/podman-system-generator --user --dryrun 2>&1
```

This prints the generated systemd unit to stdout, which is useful for spotting syntax errors or unexpected values before reloading the daemon.

### Run the container interactively

To test the container directly, bypassing systemd, you can run it interactively as the service user:

```sh
sudo -u service_name podman run --rm -it <image> /bin/sh
```

This is useful for verifying that volume mounts, environment variables, and user mappings behave as expected.
