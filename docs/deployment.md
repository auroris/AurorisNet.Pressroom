# Deployment on Windows and IIS

Host a Pressroom release as an IIS application. You set up IIS once, then copy a new release
folder into place for each update.

Windows and IIS deployments are meant for intranets. For internet-facing traffic, use the
[Linux container](container.md) with outbound blocking, or [Linux](linux.md) under a dedicated
unprivileged user.

## Server prerequisites

| Requirement | Notes |
|---|---|
| Windows Server x64 | A version that both .NET 10 and the packaged Playwright release support. Record the tested build in the [acceptance record](windows-acceptance.md). |
| IIS with the **.NET 10 Hosting Bundle** | The bundle supplies the ASP.NET Core module and the .NET 10 runtime. A runtime or SDK alone lacks the module, and a later major runtime does not run a `net10.0` application. |
| Application Initialization | Optional IIS feature that starts the renderer before the first request. |

See the [Hosting Bundle](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/hosting-bundle?view=aspnetcore-10.0),
[runtime selection](https://learn.microsoft.com/en-us/dotnet/core/versions/selection) and
[Playwright's supported systems](https://playwright.dev/dotnet/docs/intro).

## What is in a release

Each GitHub release holds `Pressroom-<version>-win-x64-framework-dependent.zip` and its
`.zip.sha256` checksum, beside the Linux tarball and the .NET client package. The ZIP holds:

| Content | Purpose |
|---|---|
| `Pressroom.Server.exe`, `.dll`, `web.config`, dependency assemblies | The application, set up for IIS in-process hosting |
| `.playwright/` | The private Node runtime and Playwright driver. A hidden folder, and **required** |
| `browsers/` | The matching Chromium headless shell and its companion files |
| `fonts/` | The Noto Sans faces, their license, and `manifest.json` with their hashes |
| `appsettings.json` | The default settings, without secrets |
| `notices/` | `THIRD-PARTY-NOTICES.md` and the license files of the packaged components |
| `release-manifest.json` | The Pressroom, Playwright, Node and Chromium versions, the fonts, and the SHA-256 hash of every file except `web.config` |
| `README.md`, `docs/`, `deploy/` | These guides and the container examples |

Check the archive against its checksum, then extract all of it.

## Directory layout and permissions

Keep everything that changes at run time outside the release folder, which each update replaces:

| Path (example) | Application pool identity |
|---|---|
| `C:\Sites\Pressroom` | Read and execute |
| `C:\ProgramData\Pressroom` | Read and traverse |
| `C:\ProgramData\Pressroom\pressroom.settings.json` | Read. Administrators manage it. |
| `C:\ProgramData\Pressroom\temp` | Modify |
| `C:\ProgramData\Pressroom\logs` | Modify |
| `C:\ProgramData\Pressroom\jobs`, when jobs are enabled | Modify |

Create the state root, `temp` and `logs` before the first start, and grant these rights to the
dedicated pool identity, such as `IIS AppPool\Pressroom`. Restrict the settings file to that
identity and administrators.

Write the settings file as [Configuration](configuration.md) describes. It needs at least
`Paths:StateRoot` and either an API key or anonymous mode.

With jobs enabled, give the state root room for `Jobs:MaxStoredBytes`, one more result, the logs
and temporary files.

## IIS settings

Create a site or application that points at the release folder, with an application pool of its
own:

| Setting | Value |
|---|---|
| Application pool | Dedicated to Pressroom, **No Managed Code** |
| Enable 32-Bit Applications | **False** |
| Maximum Worker Processes | **1** |
| Start Mode | **AlwaysRunning** |
| Load User Profile | **True** |
| Idle Time-out | **0**, so the renderer stays started |
| Application Preload | **True**, with Application Initialization installed |
| Warm-up URL | `/v1/health` |
| Authentication | **Anonymous Authentication** enabled, **Windows Authentication** disabled |
| Site binding | HTTPS with your certificate |

Anonymous Authentication lets requests reach the application, which then applies its own API-key
or anonymous access mode.

**Request size.** `web.config` sets IIS request filtering to 32 MiB, and the application rejects a
body over `Render:MaxBodyBytes`, 25 MiB by default, before it parses it. `MaxBodyBytes` must stay
below 32 MiB.

**Recycling.** Overlapped recycling runs two instances for a short time, each with its own browser
and temporary folder. Allow memory for both, or disable overlapped recycling and accept the
interruption. With jobs enabled, disable it: the job store accepts one instance, and the new
worker fails to start while the old one still holds the store.

## First start

1. Run the package check as the deploying user:

   ```powershell
   & 'C:\Sites\Pressroom\Pressroom.Server.exe' --check --config 'C:\ProgramData\Pressroom\pressroom.settings.json'
   ```

2. Start the application pool. Watch `C:\ProgramData\Pressroom\logs` for the
   `Browser warm-up succeeded` line.
3. Request `/v1/health` and expect 200. Request `/v1/info` with a key and confirm the version.
4. Render a document as in [Getting started](getting-started.md#5-render-your-first-pdf) and
   open the result.

The first deployment of a release to a new server should complete the
[Windows and IIS acceptance record](windows-acceptance.md).

## Updating

Plan a short maintenance window: stop the application, replace the release folder, start it again.

1. Extract the new release elsewhere on the server, check its checksum and run `--check`. Keep the
   previous release folder for rollback.
2. Stop the application pool, or place `app_offline.htm` at the site root, and wait until the
   browser and driver files are no longer locked. The application has
   `Render:ShutdownTimeoutMs` to finish renders and close Chromium.
3. Replace the application folder only. `robocopy /MIR` suits a destination that holds nothing but
   release files. Its exit codes 0 to 7 mean success, and 8 and above mean failure. If you use
   `app_offline.htm`, exclude it from the mirror so it stays until the copy ends.
4. If the copy fails, leave the application stopped and restore the previous complete folder
   before you start it.
5. Start the pool or remove the offline file. Check `/v1/health`, confirm the version in
   `/v1/info`, and render a document.

## Rolling back

Stop the pool, restore the previous complete release folder, start the pool, and repeat the
health, version and render checks. The application, driver, browser and fonts roll back together,
because they ship as one folder.

<a id="running-without-iis"></a>

## Running with Kestrel

The executable hosts itself on Kestrel for local use and checks:

```powershell
& 'C:\Sites\Pressroom\Pressroom.Server.exe' --config 'C:\ProgramData\Pressroom\pressroom.settings.json' --urls http://localhost:5177
```

Releases are checked for IIS in-process hosting. Running the executable as a Windows service or
behind another reverse proxy is left to you to package and qualify.

## Operations

- Monitor `/v1/health` for readiness, and `/v1/info` for queue depth and renders since startup.
- Track 429 responses, which mean the capacity is in use, apart from 503 responses, which mean the
  browser is starting or being replaced.
- To rotate an API key, add the new key, move the callers to it, then remove the old key. Each
  change needs a restart.
- After each deployment, read `release-manifest.json` to confirm the Chromium and Playwright
  versions.
- Pressroom keeps 14 log files of at most 10 MiB each. Archive them if you need them longer.
