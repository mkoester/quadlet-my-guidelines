# Quadlet: my guidelines

## About

In this document I want to collect my guidelines / best practises when it comes to hosting services on a (virtual) server.
I prefer to use **Fedora**, but this should work on all (linux) systems supporting **Podman** and **quadlets**.
You need to be able to run commands via **sudo** to set everything up (or run every service with your user account, which I would not recommend).
In this document I am going to use **nano** as the *file editor*, but every other editor like **vim** or **emacs** would also work similarly.

One could also use other means to achieve the same (e.g. via `docker compose`), but I personally prefer `quadlets` (and `podman` + `systemd`).

## (Service) users

For best isolation, I would create one dedicated user account per service I want to run.
One could use service accounts, but they usually don't have the necessary ranges for user and group sub IDs created automatically (needed to run rootless podman containers).
Therefore, we use regular users and create their home directory in [/var/lib](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s08.html).

E.g. for a service called <service_name>, we would create a user with *no login shell*

```sh
sudo useradd -m -d /var/lib/service_name -s /usr/sbin/nologin service_name
```

In order for the service to be able run in the user's context when the user itself is not logged in, we have to allow his processes to persist

```sh
sudo loginctl enable-linger service_name
```

Since we need to create and store the quadlet service definitions, we create the proper directory as follows:

```sh
sudo -u service_name mkdir -p ~service_name/.config/containers/systemd
```

## Service directories / permanent storage / volumes

Most services need both, a **configuration** and some **space on disk** to store its *state*.
The setup depends heavily on the service we want to run.
Some services consist of multiple containers which in most cases need their own configuration and storage.

### One single container

For the simple case, we would simple create 2 directories, one for configuration and one for persisting the service's data.

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

## Service definition / quadlet files`.container` file

The quadlet files should be in `~service_name/.config/containers/systemd`. The main configuration is done in a `.container` file (or several in case of more complex setups) and an optional `.network` file.

```sh
sudo -u service_name nano ~service_name/.config/containers/systemd/service_name.container
```

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

### Container definition

We'll start with `Unit` section with a customized description:

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
PublishPort=1234:5678

# Health check
HealthCmd=... || exit 1
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3
```

- `Image` specifies the image our container should be using. If we want to make use of automatic updates, the image has to be fully quallified (including registry and tag).
- `AutoUpdate` is set to `registry` ([details](https://docs.podman.io/en/v5.0.1/markdown/podman-auto-update.1.html))
- in case the image supports it, we explicitly set  the user namespace mapping. This allows the executable to run with the same permissions as the service user. Whether this works and which user/group id to use depends on the image.
- The `Volume`s we use also depend on the image. I prefer to use **bind mounts** instead of (anonymously) named volumes
- `PublishPort` has to be set according to the port you want to use locally and the port the image uses internally.
- `HealthCmd` has to be configured specifically for the image. Often a call via curl works.

