# quadlet: my guidelines

## about

In this document I want to collect my guidelines / best practises when it comes to hosting services on a (virtual) server.
I prefer to use **Fedora**, but this should work on all (linux) systems supporting **Podman** and **quadlets**.
You need to be able to run commands via **sudo** to set everything up (or run every service with your user account, which I would not recommend).
In this document I am going to use **nano** as the *file editor*, but every other editor like **vim** or **emacs** would also work similarly.

One could also use other means to achieve the same (e.g. via `docker compose`), but I personally prefer `quadlets` (and `podman` + `systemd`).

## (service) users

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

## service directories / permanent storage / volumes

Most services need both, a **configuration** and some **space on disk** to store its *state*.
The setup depends heavily on the service we want to run.
Some services consist of multiple containers which in most cases need their own configuration and storage.

### one single container

For the simple case, we would simple create 2 directories, one for configuration and one for persisting the service's data.

```sh
sudo -u service_name mkdir -p ~service_name/{data,config}
```

### several containers

In case a service requires several containers to operate, I prefer to create one directory per container.
Inside these I would then create the necessary subdirectories for each container individually.

## service definition / quadlet files
