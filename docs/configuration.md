# Configuration

Pressroom takes its settings from JSON files. This page describes where the files are, every
setting with its default and limits, and the checks that run at startup.

## The external settings file

The packaged `appsettings.json` holds the defaults. One external settings file holds everything
that belongs to a server, including its API keys.

| Rule | Behavior |
|---|---|
| Default location | `C:\ProgramData\Pressroom\pressroom.settings.json` on Windows. `/etc/pressroom/pressroom.settings.json` on Linux. |
| Another file | Set `ConfigurationFile` in `appsettings.json`, or pass `--config <path>` once on the command line. `--config` wins over `ConfigurationFile`. |
| Relative paths | Resolve from the application folder. |
| Missing file | The default file may be absent. A file selected with `ConfigurationFile` or `--config` must exist, or startup fails. |
| Layering | A value in the external file replaces the packaged default for the same key. |
| Reading | Once, at startup. Restart the application after a change. |

```json
{
  "ConfigurationFile": "D:\\PressroomConfig\\pressroom.settings.json"
}
```

Pressroom takes application settings from these two files only. It ignores environment variables
and command-line arguments other than `--config` and `--check`. Hosting switches of ASP.NET Core,
such as `--urls` and `ASPNETCORE_URLS`, still apply.

Keep the external file outside the release folder and outside source control, readable by the
service identity and administrators only. Keep secrets out of `web.config`, `appsettings.json`
and publish profiles. The repository has a
[sample settings file](https://github.com/auroris/AurorisNet.Pressroom/blob/main/samples/pressroom.settings.example.json).

## Access mode and API keys

`Auth:Mode` is `ApiKey` by default. Callers send a configured secret in the `X-Api-Key` header on
every request except `GET /v1/health`, which is always anonymous.

```json
{
  "Auth": {
    "Mode": "ApiKey",
    "ApiKeys": {
      "reporting-app": { "Secret": "REPLACE_WITH_A_RANDOMLY_GENERATED_SECRET" },
      "charts":        { "Secret": "ANOTHER_RANDOM_SECRET", "AllowJavaScript": true }
    }
  }
}
```

| Setting | Default | What it does |
|---|---|---|
| `Auth:Mode` | `ApiKey` | `ApiKey` requires a valid key. `Anonymous` also accepts requests that send no key. |
| `Auth:ApiKeys:<id>` | none | One key. `<id>` is a nonsecret identifier of 1 to 128 characters that appears in logs and owns the key's jobs. |
| `Auth:ApiKeys:<id>:Secret` | none | The value callers send, 16 to 4096 characters. Two identifiers cannot share a secret. Generate it from at least 32 random bytes. |
| `Auth:ApiKeys:<id>:AllowJavaScript` | `false` | Lets this key's documents run scripts. See [JavaScript permission](#javascript-permission). |

API-key mode needs at least one key, or startup fails. Pressroom compares secrets in constant time
and writes only the identifier to its logs.

A request that sends an invalid key gets 401 in either mode. So does a request with an empty
`X-Api-Key` header, more than one such header, or a value longer than 4096 characters.

**Anonymous mode.** Select it explicitly, for a trusted network:

```json
{
  "Auth": { "Mode": "Anonymous", "ApiKeys": {} }
}
```

Requests without a key can then render, read `/v1/info` and the OpenAPI document, and use jobs.
They render with scripts disabled, and they cannot print or list printers. A caller can still
send a configured key to use that key's permissions. All anonymous callers share one job owner, so
each can list, read and delete the others' jobs.

Under IIS, enable Anonymous Authentication and disable Windows Authentication, so requests reach
Pressroom's own access check. Windows identities and domain groups grant no access.

## JavaScript permission

Documents run scripts only when the request comes with a key that has `"AllowJavaScript": true`.
The request option `options.javaScript` defaults to `true` and can turn scripts off for one
render. It cannot turn them on for a key without the permission. A request that sets
`options.readiness.function` while scripts are off gets 400.

## Rendering limits

| Setting | Default | What it does |
|---|---|---|
| `Render:Concurrency` | `2` | Renders in progress at once, 1 to 64. Size it from the memory that representative documents use. |
| `Render:QueueLimit` | `8` | Further requests that may wait for a render slot, 0 to 1024. A request beyond `Concurrency` plus `QueueLimit` gets 429 with `Retry-After`. |
| `Render:MaxBodyBytes` | `26214400` | The largest JSON request body, 25 MiB, including the escaped HTML and templates. Must stay below 33554432, the 32 MiB limit that IIS and Kestrel apply. |
| `Render:DefaultTimeoutMs` | `30000` | The processing deadline when a request omits `options.timeoutMs`. At least 100. |
| `Render:MaxTimeoutMs` | `60000` | The largest `options.timeoutMs` a request may ask for. From `DefaultTimeoutMs` to 300000. It also limits browser startup. |
| `Render:ShutdownTimeoutMs` | `30000` | The time to finish renders and close the browser at shutdown, 1000 to 50000. Must be more than twice `CleanupTimeoutMs`. The host waits 5 seconds longer than this value. |
| `Render:CleanupTimeoutMs` | `5000` | The time to close one browser context, or to end the browser's processes, before Pressroom treats the browser as failed. 100 to 30000. |
| `Render:RestartAfterRenders` | `500` | Pressroom replaces the browser after this many render attempts. Renders in progress finish first, and waiting requests keep their deadlines. |
| `Render:DefaultPageSize` | `Letter` | The page size when a request omits `options.pageSize`: `Letter`, `Legal`, `Tabloid`, `A3`, `A4` or `A5`, spelled exactly so. |
| `Render:ChromiumSandbox` | `true` | Must be `true`. With `false`, startup fails. |
| `Render:BrowsersPath` | `browsers` | The packaged browser folder, relative to the application folder. |
| `Render:ChromiumPath` | none | Another Chromium executable to launch in place of the packaged one. Startup still runs the sandbox and connection checks on it. Releases are tested with the packaged browser only. |

The processing deadline covers the wait for a render slot, loading the document, the readiness
conditions, PDF generation and page counting. Sending the response has no deadline, but the
request keeps its place in the admission count until the response is complete, so slow downloads
count against capacity.

Under IIS, `web.config` gives the application 120 seconds to start and 60 seconds to stop.
Container and systemd stop timeouts must be longer than `ShutdownTimeoutMs` plus 5 seconds.

## Paths, fonts and logs

| Setting | Default | What it does |
|---|---|---|
| `Paths:StateRoot` | `C:\ProgramData\Pressroom` on Windows, `/var/lib/pressroom` on Linux | The writable folder for `temp`, `logs` and `jobs`. Must be an absolute path outside the release folder. Write it out in full: placeholders such as `%ProgramData%` are not expanded. |
| `Fonts:Manifest` | `fonts/manifest.json` | The list of packaged fonts, relative to the application folder. Part of the release. |

Pressroom creates `temp`, `logs` and `jobs` under `StateRoot` when it may. Each running instance
works in its own folder under `temp` and removes it at shutdown.

Logging is fixed. Pressroom writes to the console and to `pressroom-<date>.log` files in
`<StateRoot>/logs`. A new file starts each day and at 10 MiB, and the 14 newest files are kept.
Log lines hold request identifiers, key identifiers, sizes, timings, page counts, status codes and
counts of blocked resources. They never hold document content, PDF bytes, secrets, blocked URLs or
text from the page's console.

## Printing

```json
{
  "Printing": {
    "SumatraPdfPath": "C:\\Tools\\SumatraPDF\\SumatraPDF.exe",
    "Printers": {
      "shipping": "Warehouse Laser Printer",
      "pdf-test": "Microsoft Print to PDF"
    }
  }
}
```

| Setting | Default | What it does |
|---|---|---|
| `Printing:Printers` | none | The allowlist. Each key is the identifier callers send: 1 to 128 ASCII letters, digits, hyphens and underscores, matched without regard to case. Each value is the Windows printer name or CUPS destination, up to 256 characters. |
| `Printing:SumatraPdfPath` | none | Windows: the absolute path of `SumatraPDF.exe`. Required when `Printers` has entries. Ignored on Linux. |
| `Printing:CupsCommandPath` | `/usr/bin/lp` | Linux: the absolute path of the CUPS `lp` or `lpr` command. Ignored on Windows. |

[Printing](printing.md) covers the host setup.

## Asynchronous jobs

`"Jobs": { "Enabled": true }` adds the `/v1/jobs` routes. Pressroom then stores submitted HTML and
finished PDFs in `<StateRoot>/jobs`. Run one instance for each state root: a second instance
that opens the same job folder fails. Durations are written as `hh:mm:ss` or `d.hh:mm:ss`.

| Setting | Default | What it does |
|---|---|---|
| `Jobs:Enabled` | `false` | Turns on the routes, the scheduler and the store. |
| `Jobs:Workers` | `1` | Jobs that render at once, 1 to 64. At most `Render:Concurrency` minus `ReservedSyncSlots`. |
| `Jobs:ReservedSyncSlots` | `1` | Render slots that jobs leave free for synchronous requests. At least 1, and less than `Render:Concurrency` when jobs are on. |
| `Jobs:MaxQueued` | `200` | Jobs that may wait, 1 to 100000. |
| `Jobs:MaxQueuedPerKey` | `100` | Jobs of one owner that may wait. At most `MaxQueued`. |
| `Jobs:MaxQueueAge` | `00:10:00` | A job that waits longer than this expires. |
| `Jobs:ResultRetention` | `01:00:00` | How long a finished job and its PDF stay. An expired job's record stays for the same time again. |
| `Jobs:RequestRetention` | `01:00:00` | How long the HTML of a failed job stays, up to the job's expiry. |
| `Jobs:MaxStoredBytes` | `2147483648` | The storage budget, 2 GiB by default, from 1 MiB to 1 TiB. It counts stored HTML and PDFs plus 16 KiB for each job. |
| `Jobs:DeleteResultOnDownload` | `false` | Expires a job after its PDF has been downloaded once in full. |

The three durations must be between one second and 30 days.

**Scheduling.** Synchronous requests go first. A job starts only while no synchronous request
waits and more than `ReservedSyncSlots` render slots are free. A job's processing deadline starts
when it starts to render. `MaxQueueAge` limits the wait before that.

**Capacity.** `POST /v1/jobs` returns 429 with `Retry-After` when the queue is full, when the
owner's share of it is full, or when the request would exceed `MaxStoredBytes`. A finished PDF
that would exceed the budget fails its job with the code `storage_full`. Pressroom writes each
file under a temporary name first, so the folder can briefly need room for one more result. Logs
and the browser's temporary files are outside the budget.

**Retention.** Pressroom deletes a job's HTML when the job succeeds. It keeps the HTML of a
failed job for `RequestRetention`, or until the job expires if that is sooner. When a job expires,
its content is deleted and its record stays for one more `ResultRetention`, so the result route
returns 410. After that the ID returns 404. A PDF that is being downloaded stays until the
download ends. Expired jobs are removed at startup, whenever a job is submitted or read, and once
a minute.

When a job's files cannot be removed, for example because a backup holds one open, Pressroom logs
a warning and tries again on a later pass. A storage error while a job runs fails that job where
it can, and synchronous rendering continues. With `Jobs:Enabled` set back to `false`, nothing
removes stored jobs: wait for them to expire, or delete them, before you turn jobs off.

## Startup checks

Pressroom stops at startup, with the reason on standard error, when:

- a setting is outside its limits, or `Render:ChromiumSandbox` is `false`;
- a settings section (`Render`, `Paths`, `Fonts`, `Auth`, `Printing` or `Jobs`) has a key that
  matches no setting, such as the misspelled `Render:Concurency`;
- `Auth:Mode` is not `ApiKey` or `Anonymous`, a key entry is invalid, two keys share a secret, or
  API-key mode has no keys;
- `Paths:StateRoot` is not absolute, is inside the release folder, or cannot be written;
- the packaged driver, browser or fonts are missing, or a font does not match its checksum;
- the sandboxed browser does not start, its test render fails, or the browser does not enforce
  the connection policy that blocks document-initiated connections.

`--check` runs the same checks except those on the `Auth` section, then exits without listening.
It passes with no API key configured.
