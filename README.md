# quadlet: my guidelines

## about

In this document I want to collect my guidelines / best practises when it comes to hosting services on a (virtual) server.
I prefer to use **Fedora**, but this should work on all (linux) systems supporting **Podman** and **quadlets**.

## (service) users

For best isolation, I would create one dedicated user account per service I want to run.
One could use service accounts, but they usually don't have the necessary ranges for user and group IDs created automatically.
Therefore, I use regular users and create their home directory in [/var/lib](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s08.html).

E.g. for a service called <service_name>, I would create a user with no login shell

```sh
sudo useradd -m -d /var/lib/service_name -s /usr/sbin/nologin service_name
```
