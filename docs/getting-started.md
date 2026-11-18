# Getting started

Run a release on a Windows machine and render a first PDF. The steps are the same on Linux, with
the paths and commands from the [Linux guide](linux.md). For production hosting, follow
[Deployment on Windows and IIS](deployment.md), [Linux](linux.md) or [Containers](container.md).

## 1. Download and extract a release

Download `Pressroom-<version>-win-x64-framework-dependent.zip` and its `.zip.sha256` file from the
[releases page](https://github.com/auroris/AurorisNet.Pressroom/releases). Compare the hashes,
which must match apart from letter case, then extract the whole archive into an empty folder such
as `C:\Sites\Pressroom`:

```powershell
(Get-FileHash .\Pressroom-<version>-win-x64-framework-dependent.zip -Algorithm SHA256).Hash
Get-Content .\Pressroom-<version>-win-x64-framework-dependent.zip.sha256
Expand-Archive .\Pressroom-<version>-win-x64-framework-dependent.zip -DestinationPath C:\Sites\Pressroom
```

The folder is the whole application: the server, its private Playwright driver and Node runtime
in the hidden `.playwright` folder, the matching Chromium headless shell in `browsers`, the Noto
Sans fonts, the third-party notices, these guides, and `release-manifest.json` with every version
and file hash. Pressroom needs all of it at startup, so keep the folder complete.

Install the .NET 10 ASP.NET Core runtime. For IIS, install the
[.NET 10 Hosting Bundle](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/hosting-bundle?view=aspnetcore-10.0),
which includes the runtime and the IIS module.

## 2. Create the settings file

Pressroom reads its packaged `appsettings.json`, then one external JSON file that holds the
settings of this server, including its API keys. The default location is
`C:\ProgramData\Pressroom\pressroom.settings.json`. Create it with one key:

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

Generate each secret from at least 32 random bytes, for example:

```powershell
$bytes = [byte[]]::new(48)
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)
```

The identifier, here `reporting-app`, is a label that appears in logs, so you can tell callers
apart. API-key mode needs at least one key before the server starts.
[Configuration](configuration.md) describes every setting, including anonymous mode for a trusted
network and the JavaScript permission for documents that draw charts.

## 3. Check the package

`--check` starts the packaged Chromium with its sandbox, renders a test page and exits without
opening a port:

```powershell
& 'C:\Sites\Pressroom\Pressroom.Server.exe' --check --config 'C:\ProgramData\Pressroom\pressroom.settings.json'
```

A working package prints one JSON line with `"status":"ready"` and the Pressroom, Chromium and
Playwright versions. A failure prints one line with the reason to standard error and exits with
code 1.

## 4. Run it

Start the executable. It hosts itself on Kestrel:

```powershell
& 'C:\Sites\Pressroom\Pressroom.Server.exe' --config 'C:\ProgramData\Pressroom\pressroom.settings.json' --urls http://localhost:5177
```

The log shows `Browser warm-up succeeded` with the Chromium version and the configured
concurrency. `/v1/health` then answers 200:

```powershell
Invoke-RestMethod http://localhost:5177/v1/health
```

## 5. Render your first PDF

The smallest request is the HTML alone:

```json
{
  "html": "<!doctype html><html><head><title>Hello</title></head><body><h1>Hello</h1></body></html>",
  "fileName": "hello.pdf"
}
```

The repository has a larger sample,
[`orders-bilingual.request.json`](https://github.com/auroris/AurorisNet.Pressroom/blob/main/samples/orders-bilingual.request.json):
two Letter pages with English and French text, all four packaged font styles, an inline vector
graphic and a page break. Download it, then send it:

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/auroris/AurorisNet.Pressroom/main/samples/orders-bilingual.request.json `
  -OutFile .\orders-bilingual.request.json
Invoke-WebRequest -Method Post -Uri http://localhost:5177/v1/render `
  -ContentType 'application/json' -Headers @{ 'X-Api-Key' = 'REPLACE_WITH_YOUR_SECRET' } `
  -InFile .\orders-bilingual.request.json -OutFile .\orders.pdf
```

With curl:

```sh
curl -sS -X POST http://localhost:5177/v1/render \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: REPLACE_WITH_YOUR_SECRET" \
  --data-binary @orders-bilingual.request.json \
  -D - -o orders.pdf
```

The response is `application/pdf`. Its `Content-Disposition: attachment` header carries the
sanitized file name, and `X-Render-Pages` and `X-Render-Duration-Ms` carry the page count and the
processing time. `orders.pdf` has two pages, with "Page 1 / 2" and "Page 2 / 2" at the bottom
right.

## 6. Connect the applications

Give each calling application the service URL and a key of its own. The running service publishes
its API contract at `/openapi/v1.json`, which needs a key in API-key mode. .NET applications can
use the `AurorisNet.Pressroom.Client` package from nuget.org or from the GitHub release.

## Where next

| To | Read |
|---|---|
| Set keys, the JavaScript permission, limits and paths | [Configuration](configuration.md) |
| Send output to a Windows or Linux printer | [Printing](printing.md) |
| Host it under IIS and keep it updated | [Deployment on Windows and IIS](deployment.md) |
| Run it on Linux under systemd | [Linux](linux.md) |
| Run it as a Linux or Windows container | [Containers](container.md) |
| Understand document isolation and what the deployment must add | [Security model](security.md) |
| Qualify a Windows server before go-live | [Windows and IIS acceptance](windows-acceptance.md) |
