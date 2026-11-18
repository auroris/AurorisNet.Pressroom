# Developing

Build and test Pressroom, publish the server folder with its browser, and package it for Windows,
Linux and containers. [Rendering service design](rendering-service-design.md) explains how the
code works.

## Prerequisites

- The .NET 10 SDK. `global.json` selects 10.0.100 or a later feature band.
- The .NET 8 runtime, for the client tests' second target. Without it, `dotnet test` reports the
  `net8.0` contracts tests as a launch failure. Pass `-f net10.0` to run the rest.
- Windows x64 to publish the Windows folder, with a Visual Studio that supports .NET 10 if you
  publish from it.
- Linux x64 with PowerShell 7 and the libraries in the [Linux guide](../linux.md) to publish the
  Linux folder.

## Repository layout

| Path | Contents |
|---|---|
| `src/Pressroom.Server` | The ASP.NET Core host: routes, authentication, admission, jobs, printing, the publish targets and the IIS `web.config` |
| `src/Pressroom.Rendering` | The browser supervisor, document validation, the font catalog and the packaged Noto Sans fonts |
| `src/Pressroom.Contracts` | The request and option records, `IPdfRenderer`, `RequestValidator` and the JSON settings |
| `src/Pressroom.Client` | `PressroomClient`, packed as `AurorisNet.Pressroom.Client` |
| `tests/` | Contract and client tests, renderer and sandbox tests, and server tests |
| `build/` | PowerShell helpers that MSBuild, the workflows and the container builds call |
| `deploy/` | Compose and Kubernetes examples, the seccomp profile and settings for `--check` |
| `samples/` | The bilingual sample document, its request, and an example settings file |
| `docs/` | The administrator guides, which ship in every release |
| `docs/development/` | This guide, the API reference, the authoring and client guides, and the design page |
| `.github/workflows/` | `ci.yml`, `release.yml` and `containers.yml` |

## Rules for changes

- **Keep the API compatible.** Add routes or optional fields. The request shape, headers and
  status codes of existing routes stay as they are. Update `src/Pressroom.Server/openapi.json`,
  which is written by hand, with the routes.
- **Settings come from JSON only.** A new setting needs a property on its settings class, a
  default in `appsettings.json` where one applies, validation at startup, and a row in
  [Configuration](../configuration.md). A key that matches no property stops startup, and
  `ConfigurationTests` binds `appsettings.json`, `samples/pressroom.settings.example.json` and
  `deploy/check.settings.json` to prove they hold known keys only.
- **The sandbox stays on.** Never add `--no-sandbox`, let a failed sandboxed launch pass, or skip
  Playwright's host validation in a release.
- **Logs hold metadata only.** Never log document content, PDF bytes, secrets, URLs, console text
  or an exception message that came from the browser. Log the exception type.
- **A release passes `--check` with its packaged browser** before it is published. Publish runs
  that check.
- **Test at both levels.** Server and client tests use a substitute renderer. Renderer and
  sandbox tests use real Chromium.

## Documentation rules

`build/Publish-Pressroom.ps1` copies `README.md`, `docs/*.md`, `docs/images/` and the examples in
`deploy/` into every release, where administrators read them offline. `docs/development/` and
`samples/` stay in the repository.

- In a page that ships, link to other shipped pages with relative links, and to anything else
  with an absolute `https://github.com/auroris/AurorisNet.Pressroom/blob/main/...` link.
- Keep these file names, which code refers to: `docs/linux.md` (a startup error message),
  `docs/container.md` (`build/Test-Container.ps1` and `deploy/compose.host-isolated.yaml`) and
  `docs/windows-acceptance.md` (the release manifest).
- `src/Pressroom.Client/README.md` is packed into the NuGet package and shown on nuget.org. Use
  absolute links and no HTML in it.
- `docs/*.md` is for administrators who install and operate Pressroom. Material for application
  developers and contributors goes in `docs/development/`.

## Build and test

NuGet restores from the project files. There are no package lock files.

```sh
dotnet restore Pressroom.slnx
dotnet build Pressroom.slnx -c Release --no-restore
dotnet test --solution Pressroom.slnx -c Release --no-build
```

These tests use a substitute renderer and run anywhere, also on a machine that has Pressroom
installed: the test host reads its own settings file and never the machine-wide one.

The tests that drive real Chromium are skipped unless you enable them. Publish once (see below)
to get the browser, then set:

| Variable | Value |
|---|---|
| `PRESSROOM_RUN_BROWSER_TESTS` | `1` |
| `PRESSROOM_TEST_BROWSERS_PATH` | The published `browsers` folder |
| `PRESSROOM_TEST_SANDBOX` | Leave unset or `true`. `false` turns the sandbox off for the renderer tests. |
| `PRESSROOM_TEST_CHROMIUM_PATH` | Optional: another Chromium executable to test |

```sh
dotnet test --project tests/Pressroom.Rendering.Tests -c Release --no-build
dotnet test --project tests/Pressroom.Server.Tests -c Release --no-build
```

`release.yml` runs both with the packaged browser on every release.

### Windows firewall prompts during browser tests

The sandbox tests open local TCP, UDP and WebRTC sockets on purpose. Windows may show a firewall
prompt for each new path of `chrome-headless-shell.exe`. Choose Cancel or Block, which keeps
inbound traffic blocked, and reuse one browser folder for repeated runs. To avoid the prompt, run
this helper in an administrator PowerShell before you test a browser copy:

```powershell
./build/Block-TestBrowserInbound.ps1 -BrowserExecutable 'D:\path\to\chrome-headless-shell.exe'
```

It adds an inbound block rule for that one executable on every network profile. Another browser
path needs another rule.

### Longer jobs acceptance run

With the browser tests enabled, set `PRESSROOM_JOB_ACCEPTANCE_COUNT=500` to run 500 jobs in place
of 50. Set `PRESSROOM_JOB_METRICS_PATH` to a file path to save the 95th-percentile latency of the
synchronous requests and the count of 503 responses during browser replacement. The 500-job run
requires that latency to stay within twice its idle value. Either run fails on a 429 for a
synchronous request.

## Running locally

Publish the `IisFolder` profile once, then run the published executable with a settings file of
your own:

```powershell
& '.\artifacts\publish\IisFolder\Pressroom.Server.exe' --config 'C:\PressroomConfig\pressroom.settings.json' --urls http://localhost:5177
```

Set `Paths:StateRoot` to a writable folder outside the published folder, and configure a key or
anonymous mode. `--check` runs the startup checks and exits.

For work that needs no browser, `-p:PressroomBundleBrowser=false` publishes without the browser
and without the check. That output cannot render and is not a release.

## Publishing the Windows folder

From Visual Studio, open `Pressroom.slnx`, choose **Pressroom.Server → Publish**, select
**IisFolder** and click **Publish**. From a command line:

```powershell
dotnet publish src/Pressroom.Server -c Release -p:PublishProfile=IisFolder
```

The output is `artifacts\publish\IisFolder`: `net10.0`, `win-x64`, framework-dependent, with
single-file, trimming and AOT off. **IisFolderSelfContained** adds the .NET runtime. Releases do
not include it.

`Pressroom.Publish.targets` runs `build/Publish-Pressroom.ps1` twice during publish, under Windows
PowerShell 5.1 on Windows and `pwsh` on Linux:

1. **Prepare.** The script runs the Playwright package's own driver with its private Node runtime
   to install the matching Chromium headless shell into a build folder. It copies the
   third-party notices, the license files of each NuGet package, `README.md`, the administrator
   guides and the `deploy` examples beside it. MSBuild adds all of it to the publish file list.
2. **Verify.** MSBuild copies the complete file list to a separate folder, and the script runs
   `Pressroom.Server --check` there, with the sandbox on, temporary state and anonymous settings.
   Publish fails when the check fails or takes more than three minutes.
3. **Manifest.** After a passed check, the script writes `release-manifest.json` with the
   versions, the fonts and the SHA-256 hash of every file.

Keep these details when you change the script:

- It removes Playwright's `DEPENDENCIES_VALIDATED` markers, which record a check of the build
  machine, and keeps `INSTALLATION_COMPLETE` and every file the installer placed.
- It leaves `web.config` out of the hashes, because the Web SDK rewrites that file for the
  publish profile afterwards.
- It removes `PLAYWRIGHT_SKIP_VALIDATE_HOST_REQUIREMENTS`, `PLAYWRIGHT_NODEJS_PATH` and
  `PLAYWRIGHT_DRIVER_SEARCH_PATH` from the check's environment, so the check uses the packaged
  driver with host validation on.
- It stops when the driver's version differs from the `Microsoft.Playwright` package version.

The first publish needs internet access to restore packages and download the browser. An ordinary
build downloads no browser.

The check proves that the folder starts and renders on the build machine. Qualify a release on
its target server with the [acceptance checklist](../windows-acceptance.md).

### Web Deploy profiles

A Visual Studio **IIS / Web Deploy** profile of your own works for `Pressroom.Server` with
**Release**, **net10.0**, **win-x64**, framework-dependent, and single-file, trimming and AOT
off. The publish targets add the browser files and run the check before the transfer starts. Keep
credentials in Visual Studio's user settings, out of a committed `.pubxml`.

`EnableMsDeployAppOffline` defaults to `true`, so the ASP.NET Core module stops the application
before Web Deploy replaces locked files. For a Folder profile that points straight at an IIS
folder, stop the pool before the copy and start it afterwards. Keep the previous complete folder
for rollback.

## Building the Linux package

On Linux x64, as a user other than root:

```sh
pwsh ./build/Publish-Linux.ps1 -Version 1.2.3 -OutputDirectory ./artifacts/linux-publish
pwsh ./build/Package-Pressroom.ps1 -PublishDirectory ./artifacts/linux-publish \
  -OutputDirectory ./artifacts/linux-release -Version 1.2.3 \
  -Variant framework-dependent -RuntimeIdentifier linux-x64
```

`Publish-Linux.ps1` restores, builds, runs the tests, publishes with the same sandboxed check as
on Windows, then runs the real-browser tests against the published browser.
`Package-Pressroom.ps1` checks the folder and its manifest, creates the tarball with the
executable file modes that the Linux guide relies on, and writes the `.sha256` file. With
`-RuntimeIdentifier win-x64`, its default, it creates the Windows ZIP.

## Building container images

On a Linux Docker host with PowerShell 7:

```sh
pwsh ./build/Build-Container.ps1 -Version 1.2.3 -Image pressroom:local
```

Start with no `artifacts/container-publish` folder. The script:

1. builds the `builder` stage of the `Dockerfile`, then runs it as UID 10001 with the seccomp
   profile, which runs `Publish-Linux.ps1` into `artifacts/container-publish`. Publish runs in a
   container and not in a Docker build step, because a build step cannot use the seccomp profile
   that the sandboxed check needs. The builder needs network access;
2. builds the `runtime` image from that verified folder;
3. runs `--check` in the image with networking off;
4. runs `build/Test-Container.ps1`: readiness within 10 seconds, a two-page bilingual PDF with a
   key, 401 for an invalid key, an empty printer list, a clean stop with a job rendering, and the
   loss of temporary state on restart.

`-TestNetworkIsolation` also tests outbound blocking against a listener on the host. Apply the
iptables rules of the [container guide](../container.md#optional-linux-outbound-isolation)
first: the scripts install no firewall rules.

On a Windows Docker host with PowerShell 7 and the .NET 10 SDK:

```powershell
./build/Build-WindowsContainer.ps1 -Version 1.2.3 -Image pressroom:windows-local
```

This publishes the Windows folder, builds `Dockerfile.windows` on the full
`windows/server:ltsc2022` base, and runs `build/Test-WindowsContainer.ps1`.
`-TestNetworkIsolation` installs and checks the HNS policy and needs an administrator session.

### Playwright version selection

`Publish-Linux.ps1`, `Publish-WindowsContainer.ps1` and the two container build scripts take
`-PlaywrightVersion`:

| Value | Selects |
|---|---|
| `pinned` (default) | `PressroomPlaywrightVersion` in `src/Pressroom.Rendering/Pressroom.Rendering.csproj` |
| `latest` | The newest stable release on nuget.org |
| A version such as `1.64.1` | That release |

The apt package list in the `Dockerfile` follows the Chromium dependency list of the pinned
Playwright release. Review the list and its version comments when the pin changes.

## The client package

`Pressroom.Contracts` and `Pressroom.Client` target `net8.0` and `net10.0`. Only
`Pressroom.Client` is packed, as `AurorisNet.Pressroom.Client`, and the package holds both
assemblies with no package dependency. The server and the renderer target .NET 10 and are not
packed.

```sh
dotnet restore Pressroom.slnx
dotnet pack src/Pressroom.Client -c Release -o artifacts/nuget --no-restore
```

Pass the same `-p:Version=1.2.3-rc.1` to restore and pack for another version. The package has
`Pressroom.Client.dll` and `Pressroom.Contracts.dll` for each target, the README in
`src/Pressroom.Client`, the icon `docs/images/icon.png` and the MIT license expression. The
`.snupkg` has the portable PDBs of both assemblies, with Source Link to the build's commit.

Package validation checks that the two targets are compatible. No baseline version is set, so a
new version is not compared with the previous one. Set `PackageValidationBaselineVersion` in
`build/ClientPackage.props` to a published version to add that comparison.

`build/Test-ClientPackages.ps1` checks a packed package: both assemblies and their symbols for
each target, no package dependency, and two new web applications, one for each target, that
install the package from `artifacts/nuget` and compile a render call. With
`-ServerUrl http://127.0.0.1:5177/` they also render against an anonymous server on the loopback
address. With `-BuildOnly` they only compile, as `release.yml` does. `-Net8Dotnet` names another
`dotnet` executable for the .NET 8 run.

## Releases

The [release guide](https://github.com/auroris/AurorisNet.Build/blob/main/docs/releasing.md)
describes the automation shared by the AurorisNet repositories: the workflows, the inputs of
`release.yml`, version numbers, retries and nuget.org trusted publishing. This section covers
what is particular to Pressroom.

**Primary dependency.** Pressroom follows `Microsoft.Playwright`, which selects the Chromium
build that every installer and container ships. Its version is the `PressroomPlaywrightVersion`
property in `src/Pressroom.Rendering/Pressroom.Rendering.csproj`.

**Dependabot** (`.github/dependabot.yml`) checks NuGet each day and workflow actions each week.
It opens one pull request for Playwright, one for the test packages, one for the other packages
and one for the actions. It proposes no major update of Playwright, or of the
`Microsoft.AspNetCore.*` and `Microsoft.Extensions.*` packages, which follow the `net10.0` target
framework. The Dockerfiles use floating base image tags that the build scripts always pull, so
Dependabot does not watch them.

**`ci.yml`** runs on each pull request. It builds and runs the tests on Windows and Linux, without
the real-browser tests. It then merges a Dependabot update that passed, and starts `release.yml`
when the update is of a package that a project under `src` references.

**`release.yml`**, named Release, takes the inputs `version`, `bump`, `test`, `dry-run` and
`notes`. A version with a suffix, such as `1.2.3-rc.1`, is a prerelease. Its jobs:

| Job | What it does |
|---|---|
| Windows package | Builds, runs the tests unless `test` is off, publishes `IisFolder`, and creates `Pressroom-<version>-win-x64-framework-dependent.zip`. Then runs the real-browser tests against the packaged browser, on every run. |
| Client package | Builds and tests both targets, packs, and runs `Test-ClientPackages.ps1 -BuildOnly`. |
| Linux package | Applies the outbound-blocking rules, runs `Build-Container.ps1` with `-TestNetworkIsolation`, creates `Pressroom-<version>-linux-x64-framework-dependent.tar.gz` from the verified folder, writes the SPDX SBOM of the verification image, and fails on a critical vulnerability in that image. |
| Publish | Checks every file against its `.sha256` file, then tags the commit, pushes the client package to nuget.org and publishes the GitHub release. A dry run stops before this. |

Both installers use the pinned Playwright version, so one release carries one Chromium build. The
run fails when either package's manifest names another version.

One release holds the Windows ZIP, the Linux tarball, the client `.nupkg` and `.snupkg`, a
`.sha256` file for each of those four, and `pressroom.spdx.json`. A dry run keeps the same files
for 14 days as the artifacts `windows-install-packages`, `linux-install-packages` and
`nuget-packages`.

When a Playwright update needs a library that the `Dockerfile` does not install, the Linux job
fails and nothing is published. Update the apt list, then run the release again.

`Version` in `Directory.Build.props` and the `-Version` defaults of the build scripts set the
version of local builds only. A release takes its version from the input or from the newest
`v*` tag.

## Container images

Container images are published by hand, with **Actions → Container images → Run workflow**
(`containers.yml`).

| Input | Use |
|---|---|
| `version` | The application version, such as `1.2.3`. Required. |
| `publish` | Push the images to GitHub Container Registry. On by default. Turn it off to build and test only. |
| `update_latest` | Also move the `latest` tags. On by default. A prerelease version never moves them. |

The workflow selects the newest stable Playwright release, so an image can carry a newer Chromium
than the installers of the same version. The Linux job runs `Build-Container.ps1` with
`-TestNetworkIsolation`, writes an SBOM and fails on a critical vulnerability. The Windows job
runs `Build-WindowsContainer.ps1` with `-TestNetworkIsolation` on a Windows Server 2022 runner,
and fails on a critical vulnerability in a NuGet dependency. Both jobs keep their test results as
artifacts for 14 days, and push with the workflow's own token. The
[container guide](../container.md#images) lists the tags.

A package that GitHub Container Registry has just created can be private. Set its visibility to
public once to allow anonymous pulls.
