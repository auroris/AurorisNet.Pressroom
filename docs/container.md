# Containers

Run Pressroom from its Linux or Windows container image. The files this page names are in the
`deploy` folder of each release.

For an internet-facing service, use the Linux container with the
[outbound blocking](#optional-linux-outbound-isolation) below, API keys, and an HTTPS reverse
proxy in front of the loopback-bound API. Windows containers are meant for intranets.

## Images

Images are published to GitHub Container Registry as `ghcr.io/auroris/pressroom`.

| Platform | Tags |
|---|---|
| Linux x64 | `<version>` and `<version>-linux`. `latest` and `latest-linux` for the newest stable version. |
| Windows Server LTSC 2022 | `<version>-windows-ltsc2022`. `latest-windows-ltsc2022` for the newest stable version. |

A prerelease version never moves the `latest` tags. In production, pull by an explicit tag or by
digest.

An image is built with the newest stable Playwright release of its build day, so its Chromium can
be newer than the one in the Windows and Linux archives of the same version. `/v1/info` reports
the versions that run.

The Linux image is built on Microsoft's .NET 10 ASP.NET Core image and runs as UID and GID 10001.
It listens on TCP 8080, reads `/etc/pressroom/pressroom.settings.json`, and writes under
`/var/lib/pressroom`. Chromium's sandbox stays on.

## How the Linux container runs

The Compose file and the `docker run` command below set this up:

- **One listener.** TCP 8080 serves rendering, jobs and health.
- **Read-only file system.** `/tmp` and `/var/lib/pressroom` are memory-backed and last as long as
  the container.
- **No capabilities.** `cap_drop: [ALL]` removes NET_ADMIN and NET_RAW among the rest, so the
  application cannot change network policy or open raw sockets.
- **Settings from outside.** The JSON settings file, with the API keys, is mounted read-only.
- **PDF output only.** The image has no CUPS client. For [Linux printing](printing.md), install
  Pressroom on the host.

**Jobs and their results are lost when the container stops, restarts or is replaced.** Download
results first. Callers must submit lost work again. A Linux host installation keeps jobs on disk.

Docker's default networking lets the container open outbound connections. The next section blocks
them.

## Optional Linux outbound isolation

Create a dedicated bridge network and add these rules before you start a container on it. The
commands are for Docker Engine with its iptables firewall backend, which includes iptables-nft.

```sh
docker network create --driver bridge \
  --opt com.docker.network.bridge.name=pressroom0 \
  --opt com.docker.network.bridge.enable_icc=false pressroom-egress-blocked

sudo iptables -I DOCKER-USER 1 -i pressroom0 -m conntrack ! --ctstate ESTABLISHED,RELATED -j DROP
sudo iptables -I INPUT 1 -i pressroom0 -m conntrack ! --ctstate ESTABLISHED,RELATED -j DROP
sudo ip6tables -I FORWARD 1 -i pressroom0 -m conntrack ! --ctstate ESTABLISHED,RELATED -j DROP
sudo ip6tables -I INPUT 1 -i pressroom0 -m conntrack ! --ctstate ESTABLISHED,RELATED -j DROP
```

The rules drop every new connection that leaves the `pressroom0` bridge, to the host, to other
containers and to the internet, and they let replies to API requests through.

- The `INPUT` rules cover services on the host itself. The forwarding rules alone leave those
  reachable.
- The IPv6 rules also cover hosts where Docker has no IPv6 `DOCKER-USER` chain.
- The rules match `pressroom0` only. Use that network for Pressroom alone, and attach no second
  network to the container.
- Run the commands once. Make them persistent with your host's firewall management, so they are
  in place before the container starts after a reboot.
- To remove these exact rules, stop the containers and repeat each command with `-D <chain>` in
  place of `-I <chain> 1`. Leave other rules alone.

A Docker `internal` network is no substitute: a container on one can still reach services on the
host gateway, and its ports are not published. The isolated setup also points the container's
DNS at its own loopback address, so Docker's resolver forwards no outside queries.

The host must allow user namespaces and, on Ubuntu, satisfy AppArmor: see [Linux](linux.md). The
seccomp profile allows Chromium's namespace calls, and its `chroot` inside that namespace, with
every host capability dropped. **Never add `SYS_ADMIN` or turn off Chromium's sandbox to get past
a startup failure.**

After you apply the rules, verify them. From inside the running container, a connection to a
listener on the host gateway and a connection to an internet address must both fail, while
`/v1/health` on the published port still answers.

## Run

Create `deploy/pressroom.settings.json` with API keys. Make it readable by UID or GID 10001, for
example owned by root, with group 10001 and mode 0640. Then start the container:

```sh
PRESSROOM_VERSION=<version> docker compose -f deploy/compose.yaml up -d
```

With outbound blocking, apply the rules above first, then add the override file:

```sh
PRESSROOM_VERSION=<version> docker compose -f deploy/compose.yaml -f deploy/compose.host-isolated.yaml up -d
```

Compose publishes the API on `127.0.0.1:8080` only. Put an HTTPS reverse proxy on the host for
remote clients. The same isolated container with `docker run`:

```sh
docker run -d --name pressroom --init -p 127.0.0.1:8080:8080 \
  --network pressroom-egress-blocked --dns 127.0.0.1 \
  --read-only --user 10001:10001 --cap-drop ALL \
  --security-opt no-new-privileges \
  --security-opt seccomp=deploy/pressroom-seccomp.json \
  --shm-size=1g --stop-timeout 45 \
  --tmpfs /tmp:rw,nosuid,noexec,size=256m \
  --tmpfs /var/lib/pressroom:rw,nosuid,noexec,size=512m,uid=10001,gid=10001,mode=0700 \
  --mount type=bind,source="$PWD/deploy/pressroom.settings.json",target=/etc/pressroom/pressroom.settings.json,readonly \
  ghcr.io/auroris/pressroom:<version>
```

- **Keep `init: true` in Compose, or `--init` with `docker run`.** Docker's init then runs as
  PID 1. It passes SIGTERM to the service, so Pressroom shuts down in order, and it reaps
  Chromium helper processes that lose their parent.
- **Stop timeout.** Keep it above `Render:ShutdownTimeoutMs` plus 5 seconds. The examples use 45
  seconds for the default of 30.
- **Health.** Probe `/v1/health`. It stays 200 while the render queue or the job queue is full.
- **Logs.** Read the console log through Docker. The log files in the state folder go with the
  container.
- **Settings.** Application settings come from the JSON file only. `ASPNETCORE_URLS` is a hosting
  setting, and the image sets it to `http://+:8080`.
- **Size.** Compose limits the container to 1 CPU and 1.5 GiB of memory, and the memory-backed
  folders count toward that memory. With jobs enabled, set `Jobs:MaxStoredBytes` below the 512 MiB
  of the state folder, with room for the logs, the browser's temporary files and one more result.
  256 MiB is a starting point. Measure your documents before you change the sizes.

## Check a running container

- `GET /v1/health` returns 200.
- `GET /v1/info` with a key reports the expected version, and with an invalid key returns 401.
- A render with a key returns a PDF.
- With outbound blocking, the connection tests under
  [Optional Linux outbound isolation](#optional-linux-outbound-isolation) fail as described.

## Kubernetes

The [Kubernetes example](../deploy/kubernetes.yaml) has one replica, a Secret named
`pressroom-settings` for the settings file, memory-backed volumes for the state, `/tmp` and
`/dev/shm`, and a NetworkPolicy that allows TCP 8080 in and nothing out. This manifest has not
been run on a cluster.

- The cluster's network plugin must enforce NetworkPolicy.
- Install `pressroom-seccomp.json` as `seccomp/pressroom.json` under the kubelet root of each
  node.
- An `emptyDir` volume lasts as long as its Pod. Replacing the Pod removes its jobs and results.
- Kubernetes has no equivalent of `init: true`, so the service is PID 1 in its Pod. To reap
  Chromium processes that lose their parent, set `shareProcessNamespace: true`, or build an image
  whose entry point is a small init such as `tini`.

## Windows containers

The Windows image is built on the Windows Server LTSC 2022 base with the .NET 10 runtime, for a
Windows Server 2022 host. A Windows base image must match the host's version. The image needs
the full Windows Server base: on Server Core, the sandboxed browser check fails.

Put `pressroom.settings.json` in a folder of its own, with `Paths:StateRoot` set to `C:\state`
and no printing settings. The helper refuses other values. Start the container with the helper
from the `deploy` folder:

```powershell
.\deploy\Start-WindowsContainer.ps1 -ConfigurationDirectory C:\Pressroom\config `
  -Image ghcr.io/auroris/pressroom:<version>-windows-ltsc2022
```

The helper mounts the folder read-only at `C:\config`, publishes container port 8080 on host port
8080 (`-Port` selects another), and sets a stop timeout of 45 seconds. The container runs as
`ContainerUser`.

**State.** The helper passes `--rm`, so a container that stops or crashes is removed with
everything it wrote. Start a new one with the helper after a stop or a host reboot. To discard
state, stop the container and start a new one: `docker restart` keeps the same writable layer.

**Outbound blocking.** Add `-BlockOutbound`, from an administrator session. The helper then runs
`Install-WindowsContainerPolicy.ps1` before it starts the container. That script creates the
`pressroom-egress-blocked` NAT network and installs and verifies HNS rules on it: they allow
inbound TCP 8080, block every new outbound connection, and close the other ports of the container
interface. The rules apply to that network only. If the script's default subnet,
`172.30.77.0/24`, overlaps one of yours, run the script yourself first with `-Subnet` and
`-Gateway`.

## Upgrade and rollback

Pull an explicit tag or digest, download the job results you need, then create the container
again with the same settings file. To roll back, create it from the previous image digest. Either
way the new container starts with no jobs or results. Keep the host firewall rules in place
throughout.
