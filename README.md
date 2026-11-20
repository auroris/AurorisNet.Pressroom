# Pressroom

An HTML-to-PDF service for Windows, IIS, Linux and containers: an application posts one complete
HTML document and receives a PDF, or sends the pages to a configured printer.

![MIT license](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/license-mit.svg)
![.NET 10](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/dotnet-10.svg)
![Windows and IIS](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/windows-iis.svg)
![Linux](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/linux.svg)
![Renderer: Chromium headless shell](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/renderer-chromium.svg)
![Output: PDF or printer](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/output-pdf-printer.svg)
![Containers: Linux and Windows x64](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/containers.svg)

![A render request on the left, its PDF response headers below it, and the two rendered Letter pages of the bilingual sample document on the right](docs/images/hero.png)

Pressroom is a standalone ASP.NET Core 10 service. It renders with the Chromium headless shell,
which it drives through Playwright. A release is one folder that holds the server, its private
Playwright driver and Node runtime, the matching Chromium build, and the Noto Sans fonts, with a
manifest of file hashes. [Download the latest release](https://github.com/auroris/AurorisNet.Pressroom/releases/latest).

- **Rendering.** `POST /v1/render` takes a JSON body with `html` and returns `application/pdf`,
  with the page count and render time in response headers.
- **Paper options.** Named and custom page sizes, margins, scale, print or screen media, header
  and footer templates with page numbers, tagged PDF, and an outline built from the headings.
- **Fonts.** Noto Sans in regular, bold, italic and bold italic is added to every document. A
  document embeds any other font as a `data:` URL.
- **Document isolation.** Each render gets a new browser context in sandboxed Chromium. Only
  `data:` resources load. Scripts run only for API keys that have `AllowJavaScript`.
- **Printing.** A request with a `printer` identifier sends the pages to an allowlisted Windows
  printer through SumatraPDF, or to a Linux CUPS queue, and returns a submission receipt.
- **Access.** Named API keys in the `X-Api-Key` header, or anonymous mode for a trusted network.
  `/v1/health` is always anonymous.
- **Capacity.** Two renders run at once and eight requests wait by default. Further requests get
  429 with `Retry-After`. One deadline covers each request from admission to the finished PDF.
- **Jobs.** With `Jobs:Enabled`, callers submit a document, poll its job, and download or cancel
  it later. Jobs are stored on disk and queued jobs survive a restart.
- **.NET client.** The `AurorisNet.Pressroom.Client` package calls the API from .NET 8 and
  .NET 10 applications.

## Install

Choose a deployment and follow its guide:

| Deployment | Suited to | Guide |
|---|---|---|
| Linux container with outbound blocking | Internet-facing and intranet use | [Containers](docs/container.md) |
| Linux x64 under systemd, as a dedicated unprivileged user | Internet-facing and intranet use | [Linux](docs/linux.md) |
| Windows Server with IIS | Intranet use | [Deployment on Windows and IIS](docs/deployment.md) |
| Windows container | Intranet use | [Containers](docs/container.md#windows-containers) |

For an internet-facing service, use API keys, put an HTTPS reverse proxy in front of the
loopback-bound API, and block outbound connections from the host or container. The
[security model](docs/security.md) explains each control.

### Run a release on Windows

1. Download `Pressroom-<version>-win-x64-framework-dependent.zip` from the
   [latest release](https://github.com/auroris/AurorisNet.Pressroom/releases/latest), check it
   against its `.sha256` file, and extract the whole archive into a folder such as
   `C:\Sites\Pressroom`. Install the .NET 10 ASP.NET Core runtime, or the
   [.NET 10 Hosting Bundle](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/hosting-bundle?view=aspnetcore-10.0)
   for IIS.
2. Create `C:\ProgramData\Pressroom\pressroom.settings.json` with one API key:

   ```json
   {
     "Paths": { "StateRoot": "C:\\ProgramData\\Pressroom" },
     "Auth": {
       "Mode": "ApiKey",
       "ApiKeys": {
         "reporting-app": { "Secret": "REPLACE_WITH_A_RANDOMLY_GENERATED_SECRET" }
       }
     }
   }
   ```

3. Check the package, then run it:

   ```powershell
   & 'C:\Sites\Pressroom\Pressroom.Server.exe' --check
   & 'C:\Sites\Pressroom\Pressroom.Server.exe' --urls http://localhost:5177
   ```

   `--check` starts the packaged Chromium with its sandbox, renders a test page and prints one
   JSON line with `"status":"ready"`.
4. Render a document:

   ```powershell
   $body = '{"html":"<!doctype html><html><head><title>Hello</title></head><body><h1>Hello</h1></body></html>","fileName":"hello.pdf"}'
   Invoke-WebRequest -Method Post -Uri http://localhost:5177/v1/render -ContentType 'application/json' `
     -Headers @{ 'X-Api-Key' = 'REPLACE_WITH_YOUR_SECRET' } -Body $body -OutFile hello.pdf
   ```

[Getting started](docs/getting-started.md) explains each step.

### .NET client

Add the client package to a .NET 8 or .NET 10 application:

```shell
dotnet add package AurorisNet.Pressroom.Client
```

Each GitHub release also holds the `.nupkg` file. The
[client guide](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/client.md)
shows how to render, print and run batches. Applications in other languages call the HTTP API.

## API

| Endpoint | Access | Result |
|---|---|---|
| `POST /v1/render` | The configured access mode. An API key when `printer` is set. | The PDF, or a 202 print receipt |
| `GET /v1/health` | Always anonymous | 200 `{"status":"ready"}`, or 503 `{"status":"unavailable"}` |
| `GET /v1/info` | The configured access mode | Versions, limits, counters, page sizes and packaged fonts |
| `GET /v1/printers` | An API key in every mode | The configured printer identifiers |
| `GET /openapi/v1.json` | The configured access mode | The OpenAPI document |
| `/v1/jobs` routes | The configured access mode, when `Jobs:Enabled` is `true` | Submit, poll, download, list and cancel jobs |

The [API reference](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/api.md)
lists every request option, response header and status code.

## Settings

Pressroom reads the packaged `appsettings.json`, then one external JSON file:
`C:\ProgramData\Pressroom\pressroom.settings.json` on Windows, or
`/etc/pressroom/pressroom.settings.json` on Linux. It reads them once at startup, and it takes no
application settings from environment variables. [Configuration](docs/configuration.md) lists
every setting with its limits.

| Setting | Default | What it does |
|---|---|---|
| `Auth:Mode` | `ApiKey` | `ApiKey` requires a configured key on every request except `/v1/health`. `Anonymous` accepts requests without a key. **Anonymous callers render without scripts and cannot print.** |
| `Auth:ApiKeys` | none | Named keys: an identifier mapped to a `Secret` of 16 to 4096 characters and an optional `AllowJavaScript`. API-key mode needs at least one. |
| `Paths:StateRoot` | `C:\ProgramData\Pressroom` on Windows, `/var/lib/pressroom` on Linux | The writable folder for `temp`, `logs` and `jobs`. It must be outside the release folder. |
| `Render:Concurrency` | `2` | Renders in progress at once. |
| `Render:QueueLimit` | `8` | Requests that may wait for a render slot. Beyond that, requests get 429. |
| `Render:MaxBodyBytes` | `26214400` | The largest JSON request body, 25 MiB. |
| `Render:DefaultTimeoutMs`, `Render:MaxTimeoutMs` | `30000`, `60000` | The processing deadline when a request names none, and the longest a request may ask for. |
| `Printing:Printers` | none | The printer allowlist: caller-facing identifiers mapped to printer or CUPS queue names. |
| `Jobs:Enabled` | `false` | Adds the `/v1/jobs` routes and stores submitted HTML and finished PDFs under `StateRoot`. |

## Supported versions

| Component | Supported |
|---|---|
| Server runtime | .NET 10 with ASP.NET Core 10, x64 |
| Windows | Windows Server x64 with IIS and the .NET 10 Hosting Bundle. Kestrel for local runs. |
| Linux | Ubuntu 24.04 LTS or Debian 13, x64 |
| Containers | Linux x64, and Windows Server LTSC 2022 |
| Renderer | The Chromium headless shell that the pinned Microsoft.Playwright release installs |
| Printing | SumatraPDF on Windows. CUPS `lp` or `lpr` on a Linux host. |
| `AurorisNet.Pressroom.Client` | .NET 8 and .NET 10 |

## Documentation

These guides ship in the `docs` folder of every release:

| Guide | What it covers |
|---|---|
| [Getting started](docs/getting-started.md) | From a release archive to a first PDF on a local machine. |
| [Configuration](docs/configuration.md) | The settings file, access modes and API keys, every setting with its limits, and startup checks. |
| [Printing](docs/printing.md) | SumatraPDF and CUPS setup, the printer allowlist, receipts and errors. |
| [Deployment on Windows and IIS](docs/deployment.md) | Prerequisites, folders and permissions, IIS settings, updates and rollback. |
| [Linux](docs/linux.md) | Libraries, the sandbox and AppArmor, systemd, updates, and the acceptance checklist. |
| [Containers](docs/container.md) | Image tags, the Linux image with Compose or Kubernetes, outbound blocking, and Windows containers. |
| [Security model](docs/security.md) | Document isolation, authentication, retention, resource limits, and what the deployment must add. |
| [Windows and IIS acceptance](docs/windows-acceptance.md) | The checks that qualify a release on a target IIS server. |

These guides stay in the repository:

| Guide | What it covers |
|---|---|
| [API reference](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/api.md) | Endpoints, request options, responses, status codes and jobs. |
| [.NET client](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/client.md) | Installing and using `AurorisNet.Pressroom.Client`. |
| [Authoring documents](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/documents.md) | Writing HTML and CSS for print: resources, fonts, page breaks, headers, scripts. |
| [Developing](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/developing.md) | Building, testing, publishing, packaging and releasing. |
| [Rendering service design](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/rendering-service-design.md) | How a request is processed, isolated and cancelled, and how the browser is supervised. |

[`deploy/`](deploy/) holds the Compose files, the Kubernetes example, the seccomp profile and the
Windows container helpers.

## Developing

You need the .NET 10 SDK, and the .NET 8 runtime for the client tests. Publishing needs
Windows x64, or Linux x64 with PowerShell 7.

```shell
dotnet restore Pressroom.slnx
dotnet build Pressroom.slnx -c Release --no-restore
dotnet test --solution Pressroom.slnx -c Release --no-build
# The server folder with its browser, checked by a sandboxed render (Windows)
dotnet publish src/Pressroom.Server -c Release -p:PublishProfile=IisFolder
# The client package
dotnet pack src/Pressroom.Client -c Release -o artifacts/nuget --no-restore
```

`dotnet test` skips the tests that drive real Chromium. To run them, publish once and set
`PRESSROOM_RUN_BROWSER_TESTS=1` and `PRESSROOM_TEST_BROWSERS_PATH` to the published `browsers`
folder. `release.yml` runs them against the packaged browser on every release. The
[developer guide](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/developing.md)
covers publishing, the Linux package and the container images.

## Releases

Pressroom follows Microsoft.Playwright, which selects the Chromium build that every installer and
container ships. Its version is the `PressroomPlaywrightVersion` property in
`src/Pressroom.Rendering/Pressroom.Rendering.csproj`. Dependabot checks NuGet each week and gives
Playwright a pull request of its own. `ci.yml` tests it on Windows and Linux and merges it, and
`weekly-release.yml` publishes the week's merged updates as one release. Versions are `MAJOR.MINOR.PATCH`.

One release holds the Windows x64 ZIP, the Linux x64 tarball, the client `.nupkg` and `.snupkg`,
a `.sha256` file for each of those, and the SPDX SBOM. It also pushes the client package to
nuget.org. Container images are published by hand with `containers.yml`. The
[release guide](https://github.com/auroris/AurorisNet.Build/blob/main/docs/releasing.md)
describes the automation and how to start a release by hand.

## License

[MIT](https://github.com/auroris/AurorisNet.Pressroom/blob/main/LICENSE). The
[third-party notices](https://github.com/auroris/AurorisNet.Pressroom/blob/main/THIRD-PARTY-NOTICES.md)
list the redistributed components and their licenses.
