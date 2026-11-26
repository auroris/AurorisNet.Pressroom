# Pressroom

Self-hosted HTML-to-PDF and printing service for Windows, IIS, Linux, and containers. Post a complete HTML document and get back a PDF—or send pages straight to an allowlisted printer.

![MIT license](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/license-mit.svg)
![.NET 10](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/dotnet-10.svg)
![Windows and IIS](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/windows-iis.svg)
![Linux](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/linux.svg)
![Renderer: Chromium headless shell](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/renderer-chromium.svg)
![Output: PDF or printer](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/output-pdf-printer.svg)
![Containers: Linux and Windows x64](https://raw.githubusercontent.com/auroris/AurorisNet.Build/main/badges/containers.svg)

![Render request, PDF response headers, and rendered Letter pages](docs/images/hero.png)

Built on ASP.NET Core 10 and Playwright's Chromium headless shell. Each release bundles the server, Playwright driver, Node runtime, Chromium, and Noto Sans fonts in one folder.

- **Isolated rendering:** Every `POST /v1/render` runs in a fresh sandboxed browser context (`data:` resources only; JavaScript off unless enabled per API key).
- **Print options:** Named/custom paper sizes, margins, scale, headers/footers with page numbers, tagged PDF, and heading outlines.
- **Direct printing:** Optional `printer` target prints via SumatraPDF (Windows) or CUPS (Linux) against an allowlist.
- **Queueing & async jobs:** Bounded concurrency/queue with `429 Retry-After`, per-request deadlines, and optional disk-backed `/v1/jobs`.
- **Auth:** API keys (`X-Api-Key`), or anonymous mode for trusted networks (`/v1/health` is always anonymous).

## Install

Download the latest release from [GitHub Releases](https://github.com/auroris/AurorisNet.Pressroom/releases/latest) or choose a deployment guide:

- [Containers (Linux & Windows)](docs/container.md) — recommended for internet-facing deployments with outbound blocking
- [Linux (systemd)](docs/linux.md)
- [Windows Server & IIS](docs/deployment.md)
- [Getting started](docs/getting-started.md) — local setup, `--check`, and first render

**.NET client (does not install Playwright or Chromium):**

```shell
dotnet add package AurorisNet.Pressroom.Client
```

Targets .NET 8 and .NET 10. See [src/Pressroom.Client/README.md](src/Pressroom.Client/README.md) or [client guide](docs/development/client.md).

## API overview

| Endpoint | Result |
|---|---|
| `POST /v1/render` | Returns `application/pdf`, or `202` print receipt when `printer` is set. |
| `GET /v1/health` | Always anonymous; `200 ready` or `503 unavailable`. |
| `GET /v1/info` / `GET /v1/printers` / `GET /openapi/v1.json` | Service info, configured printers (API key required), OpenAPI schema. |
| `/v1/jobs` | Submit, poll, download, list, and cancel async jobs when `Jobs:Enabled` is `true`. |

## Key settings

Reads packaged `appsettings.json` plus `C:\ProgramData\Pressroom\pressroom.settings.json` (Windows) or `/etc/pressroom/pressroom.settings.json` (Linux) on startup (no env-var overrides):

| Setting | Default | Effect |
|---|---|---|
| `Auth:Mode` | `ApiKey` | `ApiKey` or `Anonymous` (anonymous disables JS and printing). |
| `Auth:ApiKeys` | none | Named keys (`Secret` 16–4096 chars, optional `AllowJavaScript`). |
| `Paths:StateRoot` | `C:\ProgramData\Pressroom` / `/var/lib/pressroom` | Writable folder outside the release directory for temp, logs, and jobs. |
| `Render:Concurrency` / `QueueLimit` | `2` / `8` | Parallel renders and queued requests before `429`. |
| `Render:MaxBodyBytes` | `26214400` | Max JSON body size (25 MiB). |
| `Render:DefaultTimeoutMs` / `MaxTimeoutMs` | `30000` / `60000` | Request processing deadlines. |
| `Printing:Printers` | none | Allowlist mapping caller printer IDs to OS/CUPS printer names. |
| `Jobs:Enabled` | `false` | Enables disk-backed `/v1/jobs`. |

## Documentation

| Guide | Covers |
|---|---|
| [Getting started](docs/getting-started.md) & [Configuration](docs/configuration.md) | First PDF, all settings and limits |
| [Deployment (Windows/IIS)](docs/deployment.md), [Linux](docs/linux.md), [Containers](docs/container.md) | Hosting, sandboxing, `deploy/` files |
| [Printing](docs/printing.md), [Security model](docs/security.md), [Windows acceptance](docs/windows-acceptance.md) | SumatraPDF/CUPS, threat model, release checks |
| [API reference](docs/development/api.md), [.NET client](docs/development/client.md), [Authoring documents](docs/development/documents.md) | Endpoints, client usage, print HTML/CSS |
| [Developing](docs/development/developing.md) & [Design](docs/development/rendering-service-design.md) | Building, browser tests, packaging, releases |

## License

[MIT](https://github.com/auroris/AurorisNet.Pressroom/blob/main/LICENSE) ([third-party notices](https://github.com/auroris/AurorisNet.Pressroom/blob/main/THIRD-PARTY-NOTICES.md)).
