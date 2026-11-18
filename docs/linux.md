# Linux deployment

Install a Pressroom release on a Linux x64 host and run it under systemd. The
[acceptance checklist](#acceptance-checklist) at the end lists the checks to repeat on each host.

A Linux host suits internet-facing and intranet use when Pressroom runs as a dedicated
unprivileged user. Use API keys, and put an HTTPS reverse proxy in front of the loopback-bound
service. Pressroom does not restrict the outbound connections of its own processes: add host
network policy when the deployment needs that. The [Linux container](container.md) comes with a
tested recipe.

## Prerequisites

Use Ubuntu 24.04 LTS or Debian 13 on x64, with Linux 5.4 or later, which has the pidfd system
calls Pressroom uses to stop its browser processes. Install the .NET 10 ASP.NET Core runtime and
Chromium's shared libraries. On Ubuntu 24.04:

```sh
sudo apt-get install libasound2t64 libatk-bridge2.0-0t64 libatk1.0-0t64 \
  libatspi2.0-0t64 libcairo2 libcups2t64 libdbus-1-3 libdrm2 libgbm1 \
  libglib2.0-0t64 libnspr4 libnss3 libpango-1.0-0 libx11-6 libxcb1 \
  libxcomposite1 libxdamage1 libxext6 libxfixes3 libxkbcommon0 libxrandr2 \
  fontconfig fonts-noto-core
```

This is the Chromium dependency list of the packaged Playwright release. Debian 13 uses the same
package names. Keep `fontconfig` and the Noto fonts: header and footer templates use system fonts,
while the document body uses the packaged faces. Playwright checks the installed libraries when
the browser starts.

## Sandbox

The headless shell builds its sandbox from unprivileged user namespaces. The host must allow
them:

- `user.max_user_namespaces` greater than zero and, where the setting exists,
  `kernel.unprivileged_userns_clone=1`;
- on Ubuntu, an AppArmor profile that grants `userns` to the release's
  `chrome-headless-shell` executable;
- in a container, the supplied [seccomp profile](container.md).

**Never turn off Chromium's sandbox to work around host policy.** Pressroom refuses to start with
`Render:ChromiumSandbox` set to `false`.

For the installation layout below, save this profile as `/etc/apparmor.d/pressroom-browser`. It
grants user namespaces to that executable and leaves Ubuntu's global
`kernel.apparmor_restrict_unprivileged_userns` setting as it is:

```text
abi <abi/4.0>,
include <tunables/global>
profile pressroom-release /opt/pressroom/releases/**/chrome-headless-shell flags=(unconfined) {
  userns,
}
```

Load it with `sudo apparmor_parser -r /etc/apparmor.d/pressroom-browser`. Keep the release tree
owned by root and unwritable by the service. An installation outside `/opt/pressroom/releases`
needs a profile for its own browser path.

## Install and run

Download `Pressroom-<version>-linux-x64-framework-dependent.tar.gz` and its `.sha256` file from
the [releases page](https://github.com/auroris/AurorisNet.Pressroom/releases). Check the archive,
extract all of it into `/opt/pressroom/releases/<version>` with its file modes, owned by root, and
point `current` at it:

```sh
sha256sum -c Pressroom-<version>-linux-x64-framework-dependent.tar.gz.sha256
sudo install -d -o root -g root -m 755 /opt/pressroom/releases/<version>
sudo tar --no-same-owner -xzf Pressroom-<version>-linux-x64-framework-dependent.tar.gz \
  -C /opt/pressroom/releases/<version>
sudo ln -sfn /opt/pressroom/releases/<version> /opt/pressroom/current
```

Create the service user, the state root and the settings folder:

```sh
sudo useradd --system --home /var/lib/pressroom --shell /usr/sbin/nologin pressroom
sudo install -d -o pressroom -g pressroom -m 700 /var/lib/pressroom
sudo install -d -o root -g pressroom -m 750 /etc/pressroom
```

Create `/etc/pressroom/pressroom.settings.json`, readable by root and the service group only,
with API keys as [Configuration](configuration.md) describes. The default state root is
`/var/lib/pressroom`. Pressroom creates `temp`, `logs` and `jobs` in it, and stops at startup when
it cannot write there. Write paths in full, with the exact letter case.

Run the package check as the service user:

```sh
sudo -u pressroom /opt/pressroom/current/Pressroom.Server --check \
  --config /etc/pressroom/pressroom.settings.json
```

A missing library or a sandbox that cannot start fails the check and the service.

Example systemd unit:

```ini
[Unit]
Description=Pressroom HTML-to-PDF service
After=network.target

[Service]
Type=simple
User=pressroom
Group=pressroom
WorkingDirectory=/opt/pressroom/current
ExecStart=/opt/pressroom/current/Pressroom.Server --config /etc/pressroom/pressroom.settings.json --urls http://127.0.0.1:8080
Restart=on-failure
TimeoutStopSec=45
KillMode=control-group
UMask=0077
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/pressroom

[Install]
WantedBy=multi-user.target
```

Give the HTTPS reverse proxy a body limit above `Render:MaxBodyBytes` and timeouts above
`Render:MaxTimeoutMs`. Leave out systemd restrictions that forbid user namespaces.

At shutdown, Pressroom tracks the descendants of its browser driver, sends them SIGTERM and then
SIGKILL, and signals through pidfds so it cannot reach a process that reused an ID.
`KillMode=control-group` stops whatever remains if the server itself dies.

## Updates and storage

To update, stop the service, extract the new release into its own versioned folder, run `--check`,
point `current` at it and start the service. To roll back, stop the service and point `current` at
the previous complete release. Settings and state stay where they are.

With jobs enabled, give the state root room for `Jobs:MaxStoredBytes`, one more result, the logs
and temporary files. A second instance on the same state root fails to start. Before you roll
back to a release without jobs, let the stored jobs finish or back them up.

## Acceptance checklist

Repeat these checks on each host before go-live, and after a change of release, kernel or sandbox
policy. Record passed, failed or not run for each, with the release manifest and the host's
inventory.

- **Package and sandbox.** `--check` passes as the service user, with the release read-only, the
  AppArmor profile loaded and `Render:ChromiumSandbox` `true`. `/v1/health` returns 200 after
  startup, and `/v1/info` reports the expected version, Chromium version and fonts.
- **Access.** A valid key succeeds. A missing, invalid, empty or duplicated key header returns
  401. Anonymous mode, if you use it, renders without a key and rejects an invalid key.
- **Output.** The sample request
  [`orders-bilingual.request.json`](https://github.com/auroris/AurorisNet.Pressroom/blob/main/samples/orders-bilingual.request.json)
  renders two Letter pages with page numbers, French accents, bold and italic faces, and the
  inline graphic.
- **Limits and cancellation.** A body over the limit returns 413. A full queue returns 429 with
  `Retry-After` while `/v1/health` stays 200. Cancelled and timed-out requests free their
  capacity.
- **Shutdown and recovery.** `systemctl stop` finishes within `TimeoutStopSec` and leaves no
  browser or driver process. The service becomes ready again after a restart and after a browser
  crash.
- **Printing**, when configured. The CUPS command submits as the service user, an unknown
  identifier returns 400, a print request without a key returns 401, and the `temp/print-*`
  folders are removed after submission.
- **Jobs**, when enabled. A restart resumes queued jobs and marks the interrupted one failed.
  Another key cannot read or delete a job. An expired result returns 410.
- **Outbound connections**, if you block them. A connection from the service user to a target you
  can otherwise reach is blocked by the host policy.
