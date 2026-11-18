# API reference

Pressroom has a small HTTP API under `/v1`. This page lists the routes, the request options, the
responses and the status codes. A running service publishes the same contract as an OpenAPI
document at `/openapi/v1.json`.

Requests and responses are UTF-8. A request body is uncompressed `application/json`. Every
response has `Cache-Control: no-store`.

## Endpoints

| Endpoint | Access | Result |
|---|---|---|
| `POST /v1/render` | The configured access mode. An API key when `printer` is set. | `application/pdf`, or a 202 JSON print receipt |
| `GET /v1/health` | Always anonymous | 200 `{"status":"ready"}`, or 503 `{"status":"unavailable"}` |
| `GET /v1/info` | The configured access mode | Versions, limits, counters, page sizes and packaged fonts |
| `GET /v1/printers` | An API key in every mode | The configured printer identifiers |
| `GET /openapi/v1.json` | The configured access mode | The OpenAPI document |
| `/v1/jobs` routes | The configured access mode. Present only when jobs are enabled. | See [Asynchronous jobs](#asynchronous-jobs-opt-in) |

In API-key mode, send the secret in the `X-Api-Key` header. In anonymous mode the header may be
left out, but a key that is sent must be valid, or the response is 401.
[Configuration](../configuration.md#access-mode-and-api-keys) describes both modes.

## POST /v1/render

### Request

```json
{
  "html": "<!doctype html><html lang=\"en\"><head><title>Orders</title></head><body>…</body></html>",
  "fileName": "orders-2026-10.pdf",
  "printer": null,
  "options": {
    "pageSize": "Letter",
    "landscape": false,
    "margins": { "top": "20mm", "right": "15mm", "bottom": "20mm", "left": "15mm" },
    "preferCssPageSize": false,
    "printBackground": true,
    "scale": 1.0,
    "displayHeaderFooter": true,
    "headerTemplate": "<div></div>",
    "footerTemplate": "<div style=\"font-size: 10px; text-align: right; width: 100%; padding-right: 20px;\">Page <span class=\"pageNumber\"></span> / <span class=\"totalPages\"></span></div>",
    "tagged": true,
    "outline": true,
    "media": "print",
    "javaScript": true,
    "locale": "en-CA",
    "timeZone": "America/Edmonton",
    "readiness": { "selector": null, "function": null, "delayMs": 0, "timeoutMs": 5000 },
    "timeoutMs": 30000
  }
}
```

Only `html` is required, and the example shows the default of every other value. Property names
are case-sensitive, a property the API does not define is rejected, and numbers must be JSON
numbers. `options`, `margins` and `readiness` may be left out but cannot be `null`.

| Field | Type | Default | Rules |
|---|---|---|---|
| `html` | string | required | A complete HTML document. A `<base>` element is rejected. |
| `fileName` | string | `document.pdf` | The file name Pressroom returns with the PDF. Sanitized: control characters and `< > : " / \ | ? *` become `_`, the name is cut to 120 characters, a reserved Windows name gets a `_` prefix, and `.pdf` is added when missing. |
| `printer` | string or null | `null` | A configured printer identifier. The response is then a print receipt. See [Printing](../printing.md). |
| `options.pageSize` | name or object | `Letter`, or the server's `Render:DefaultPageSize` | `Letter`, `Legal`, `Tabloid`, `A3`, `A4` or `A5`, in any letter case, or `{ "width": "8.5in", "height": "11in" }`. |
| `options.landscape` | boolean | `false` | Swaps the paper's width and height. |
| `options.margins` | object | 20 mm top and bottom, 15 mm left and right | `top`, `right`, `bottom` and `left` are lengths in `px`, `mm`, `cm` or `in`, up to 200 inches. They must leave a printable area. |
| `options.preferCssPageSize` | boolean | `false` | Lets the document's `@page` size override `pageSize`. |
| `options.printBackground` | boolean | `true` | Prints background colors and images. |
| `options.scale` | number | `1` | 0.1 to 2. |
| `options.displayHeaderFooter` | boolean | `true` | Prints the header and footer templates. |
| `options.headerTemplate`, `options.footerTemplate` | string | An empty header, and a right-aligned `Page N / M` footer | HTML in which Chromium fills elements with the classes `date`, `title`, `url`, `pageNumber` and `totalPages`. A template needs its own styles. A `<base>` element is rejected. |
| `options.tagged` | boolean | `true` | Asks for a tagged PDF. |
| `options.outline` | boolean | `true` | Builds a document outline from the headings. |
| `options.media` | `"print"` or `"screen"` | `print` | The CSS media type for the render. |
| `options.javaScript` | boolean | `true` | `false` turns scripts off for this render. `true` runs scripts only when the API key allows them. |
| `options.locale` | string | `en-CA` | A culture name that .NET knows, up to 64 characters. |
| `options.timeZone` | string | `America/Edmonton` | An IANA time zone identifier, up to 128 characters. |
| `options.readiness.selector` | string or null | `null` | Waits until an element that matches is visible. 1 to 4096 characters. |
| `options.readiness.function` | string or null | `null` | Waits until a JavaScript expression is truthy. 1 to 16384 characters. Needs scripts to be on, or the response is 400. |
| `options.readiness.delayMs` | integer | `0` | A pause after the other conditions. At most `readiness.timeoutMs`. |
| `options.readiness.timeoutMs` | integer | `5000` | The time for all readiness conditions together, from 1 to the server maximum. |
| `options.timeoutMs` | integer or null | The server's `Render:DefaultTimeoutMs`, 30000 | The processing deadline, from 100 to the server's `Render:MaxTimeoutMs`, 60000 by default. |

Pressroom validates the whole request before it opens a browser context. A request that fails
validation gets 400 with an `errors` object keyed by field.

### PDF response

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename=orders-2026-10.pdf; filename*=UTF-8''orders-2026-10.pdf
Cache-Control: no-store
X-Render-Pages: 2
X-Render-Duration-Ms: 1091
```

`X-Render-Pages` is the page count. `X-Render-Duration-Ms` is the processing time from admission
to the counted PDF, including time spent waiting for a render slot. `Content-Disposition` is
always `attachment`, and carries the sanitized file name to the calling application. The
application decides whether its own response to a browser shows or downloads the PDF: see
[Returning the PDF to a browser](client.md#returning-the-pdf-to-a-browser).

### Print receipt

A request with `printer` and a valid API key returns **202 Accepted** with `application/json`
once the print command has accepted the document:

```json
{ "jobId": "64e5b7e07c8e4bdb86521bed5cd54d4c", "printer": "shipping", "status": "submitted", "pageCount": 1 }
```

Follow the printing itself in the Windows or CUPS queue. [Printing](../printing.md) explains the
receipt.

### Status codes

Errors are `application/problem+json` with `title`, `status`, `detail` and `requestId`. A
validation error adds an `errors` object keyed by field path.

| Status | Meaning | Notes |
|---|---|---|
| 200 | Rendered | The PDF and the headers above |
| 202 | Print job submitted | The JSON receipt |
| 400 | Invalid JSON, a missing or unknown field, an invalid option, a `<base>` element, an unknown printer, or a readiness function with scripts off | `errors` names the fields |
| 401 | The `X-Api-Key` header is missing in API-key mode, or is invalid, empty or repeated | Also a print request without a key |
| 413 | The body is over the application limit, 25 MiB by default, or the host's 32 MiB limit | A rejection by IIS itself may not be JSON |
| 415 | The body is not uncompressed UTF-8 `application/json` | |
| 422 | A readiness condition was not met, the page raised a script error, or Chromium could not load or print the document | A fixed message, with no text from the page |
| 429 | The render capacity is in use | `Retry-After: 1`. `/v1/health` stays 200. |
| 502 | The print command reported a failure | Pages may already be in the print queue |
| 503 | The renderer is starting, restarting or shutting down, or the print command is unavailable | `Retry-After: 1` for the renderer |
| 504 | The processing deadline passed | Pages may already be in the print queue |

When the caller disconnects, the log records status 499. A response from IIS itself, or a process
that is killed, may not produce problem details.

### Deadlines and capacity

One deadline, `options.timeoutMs`, covers reading the request, waiting for a render slot, loading
the document, the readiness conditions, PDF generation and page counting. Sending the response is
outside it.

By default two renders run and eight more requests wait, so the eleventh request at the same time
gets 429. A request holds its place until its response has been sent, so slow clients count
against capacity. [Configuration](../configuration.md#rendering-limits) lists the settings.

The complete JSON body counts against the body limit, with the escaped HTML and the templates.
Large embedded images are the usual reason to come near it. Use SVG, or raster images sized for
print.

## GET /v1/health

Returns 200 `{"status":"ready"}` once the browser has completed its warm-up render and while it
is available. Returns 503 `{"status":"unavailable"}` during startup, browser replacement, failure
and shutdown. A full queue does not change it. The route is anonymous for IIS Application
Initialization and load balancer probes.

## GET /v1/info

```json
{
  "version": "1.2.1+d57eddc…",
  "playwrightVersion": "1.64.1",
  "chromiumVersion": "156.0.8078.4",
  "active": 0, "queued": 0, "admitted": 0, "admissionLimit": 10,
  "concurrency": 2, "queueLimit": 8,
  "maxBodyBytes": 26214400, "maxTimeoutMs": 60000,
  "rendersSinceStartup": 42,
  "lastBrowserRestartUtc": "2026-10-31T06:36:10Z",
  "pageSizes": ["Letter", "Legal", "Tabloid", "A3", "A4", "A5"],
  "fonts": [{ "family": "Noto Sans", "weight": "400", "style": "normal" }, …],
  "jobs": { "enabled": false, "queued": 0, "rendering": 0, "stored": 0, "storedBytes": 0, "workers": 1, "maxQueued": 200 }
}
```

`active` and `queued` count renders in progress and waiting. `admitted` counts synchronous
requests from admission to the end of their response, and `admissionLimit` is `concurrency` plus
`queueLimit`. `fonts` lists the packaged faces. `jobs.storedBytes` includes 16 KiB for each
stored job.

## GET /v1/printers

Returns the configured identifiers in order, such as `{"printers":["pdf-test","shipping"]}`, to a
caller with a valid API key. It does not report whether a queue works.

## Asynchronous jobs (opt-in)

With `Jobs:Enabled` set to `true` on the server, the routes below exist. Otherwise they return
404. They follow the configured access mode.

A job belongs to the API key that submitted it. Another key gets 404 for that job, also on
deletion. Callers without a key share one owner and can list, read and delete each other's jobs,
so use API-key mode for private work.

| Endpoint | Response |
|---|---|
| `POST /v1/jobs` | Takes the body of `/v1/render`, with the same validation. 202 with the job document, a `Location` header and `Retry-After`. 429 with `Retry-After` when the queue, the owner's share of it, or the storage budget is full. |
| `GET /v1/jobs/{id}` | The job document, with `Retry-After` while the job is queued or rendering. 404 for an unknown ID or another owner's job. |
| `GET /v1/jobs/{id}/result` | The PDF, with `Content-Disposition`, `X-Render-Pages` and `X-Render-Duration-Ms`. 409 before the job has succeeded, for a failed job, for a print job, and while another download of the result is in progress. 410 after cancellation or expiry. |
| `DELETE /v1/jobs/{id}` | 204. Cancels a queued or rendering job, or deletes a stored result. Repeating it returns 204 while the job's record exists. 404 for an unknown ID or another owner's job. |
| `GET /v1/jobs?status=&limit=&after=` | `{ "items": [...], "nextAfter": ... }` with the caller's jobs, newest first. `status` is one status name in lower case. `limit` is 1 to 100, 50 by default. `after` is the ID of the last job of the previous page, which `nextAfter` gives while more jobs follow. |

### Submitting

A job ID is 128 random bits, written as 32 lower-case hexadecimal characters.

A submission may carry one `Idempotency-Key` header of 1 to 128 visible ASCII characters. While
the job is retained, in any state, the same owner and key get the existing job back, and the body
of the later request is ignored. Use a new key for new work.

A print job needs an API key and a configured printer identifier, as synchronous printing does.

### The job document

```json
{
  "id": "9f1c2e4d6b7a4c3e8d5f0a1b2c3d4e5f",
  "status": "queued",
  "queuePosition": 12,
  "createdUtc": "2026-11-02T03:10:00Z",
  "expiresUtc": "2026-11-02T03:20:00Z",
  "fileName": "statement-10231.pdf",
  "links": { "self": "/v1/jobs/9f1c2e4d6b7a4c3e8d5f0a1b2c3d4e5f", "result": "/v1/jobs/9f1c2e4d6b7a4c3e8d5f0a1b2c3d4e5f/result" }
}
```

| Field | Present | Meaning |
|---|---|---|
| `id`, `createdUtc`, `expiresUtc`, `fileName`, `links` | Always | `links` holds the paths of the job and its result, including the application's base path under IIS. |
| `status` | Always | `queued`, `rendering`, `succeeded`, `failed`, `cancelled` or `expired`. |
| `queuePosition` | While queued | The position in submission order across all owners. An estimate, because the scheduler takes owners in turn. |
| `startedUtc`, `completedUtc` | Once reached | When rendering started and when the job ended. |
| `pageCount`, `durationMs` | When succeeded | The page count and the processing time. |
| `printer`, `printJob` | Print jobs | The printer identifier, and the receipt once the job has succeeded. |
| `error` | When failed or expired | `type`, `title`, `status`, `detail`, `code` and `retryable`. |

Decide whether to submit a failed job again from its `error`, not from `Retry-After`:

| `error.code` | Meaning | `retryable` |
|---|---|---|
| `document` | A readiness condition or the document itself failed. | `false` |
| `deadline` | The processing deadline passed. | `true`, or `false` when pages may have reached the printer |
| `interrupted` | The service stopped while the job was rendering. | `true`, or `false` when pages may have reached the printer |
| `renderer_unavailable` | The browser restarted or stopped. | `true` |
| `storage_full` | The PDF did not fit in the storage budget. | `true` |
| `printing_unavailable` | The print command is unavailable. | `true` |
| `print_submission` | The print command reported a failure. | `false` |
| `internal` | The job could not be started or completed. | `false` |
| `queue_expired`, `expired`, `downloaded` | The job expired while waiting, after its retention, or after its download with `DeleteResultOnDownload`. | `false` |

When `retryable` is `false` for a print job, check the print queue before you submit it again:
pages may have printed.

### Scheduling and restarts

A job's processing deadline starts when the job starts to render, and `Jobs:MaxQueueAge` limits
the wait before that. The scheduler starts jobs in submission order and takes the owners in turn.
It uses the render slots that the reserve for synchronous requests leaves free, and waits while
a synchronous request waits.

After a restart, queued jobs resume, and a job that was rendering is failed with the code
`interrupted`. Pressroom never runs a job a second time by itself.

### Retention

Pressroom deletes the submitted HTML when a job succeeds. PDFs, and the HTML of failed jobs,
follow the [retention settings](../configuration.md#asynchronous-jobs). An expired job's record
stays for one more `ResultRetention`, during which the result route returns 410. After that the
ID returns 404.

With `Jobs:DeleteResultOnDownload`, a job expires after one complete download. The server cannot
know whether the client kept every byte, so leave the setting off when a download may need to be
repeated.
