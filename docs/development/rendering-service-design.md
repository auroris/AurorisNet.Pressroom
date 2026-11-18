# Rendering service design

How Pressroom processes a request, isolates a document, cancels work and supervises its browser.
Read this before you change `Pressroom.Rendering` or the request pipeline in `Pressroom.Server`:
each section names behavior that other parts, or the [security model](../security.md), rely on.

## Projects

| Project | Responsibility |
|---|---|
| `Pressroom.Contracts` | `RenderRequest`, `RenderOptions` and the other request records, `RenderJob`, `IPdfRenderer`, `RequestValidator`, and the JSON settings in `JsonDefaults`. Uses the base class library only. |
| `Pressroom.Client` | `PressroomClient`, the HTTP implementation of `IPdfRenderer`, and `PressroomException`. |
| `Pressroom.Rendering` | `BrowserRenderer`, which owns the browser; `DocumentValidator`; `FontCatalog`; the process ownership types; the packaged fonts. |
| `Pressroom.Server` | The routes, API-key authentication, admission, jobs, printing, logging and the publish targets. |

The server and the client share `RequestValidator`, so a request that the client accepts fails
on the server only for limits the client cannot know, such as `Render:MaxTimeoutMs`.

`src/Pressroom.Server/openapi.json` is written by hand and embedded in the server, which serves it
at `/openapi/v1.json`. In anonymous mode the server empties the document's top-level `security`
list. Change the file together with the routes.

## A synchronous request

`RenderEndpoints.HandleRenderAsync` runs these steps in order. The order keeps a rejected request
from costing memory or a browser context.

1. **Access.** `ApiKeyAuthenticationHandler` validates the `X-Api-Key` header. The authorization
   policy accepts a valid key, or in anonymous mode a request without the header. Under IIS,
   `IISServerOptions.AutomaticAuthentication` is `false`, so IIS supplies no Windows identity.
   That code is compiled for Windows targets only (`PRESSROOM_IIS`).
2. **Cheap checks.** A wrong content type or encoding gets 415, and a `Content-Length` over
   `Render:MaxBodyBytes` gets 413. A renderer that is not ready gets 503.
3. **Admission.** `AdmissionGate` has `Concurrency` plus `QueueLimit` permits. Without a free
   permit the request gets 429, before its body is read. The permit is held until the response
   is complete, so the number of PDFs held in memory for slow clients stays bounded.
4. **Deadline.** The deadline starts at admission with `Render:MaxTimeoutMs`. Once the request is
   parsed, it becomes `options.timeoutMs` measured from admission, so time spent reading the body
   counts.
5. **Body.** `BoundedBodyReader` counts bytes while it reads, for requests without
   `Content-Length`. `JsonDefaults.Options` rejects unknown properties and numbers written as
   strings.
6. **Preparation.** `RenderEndpoints.Prepare` sets the effective `javaScript` to the request's
   value AND the key's permission, fills in the default page size and timeout, runs
   `RequestValidator`, checks the printer against the allowlist, and rejects a `<base>` element.
   `DocumentValidator` finds `<base>` in the parsed HTML tree, because a text search misses
   markup that the parser repairs. Job submissions go through the same method.
7. **Render slot.** `RenderCoordinator` holds `Concurrency` slots for synchronous requests and
   jobs together. A job takes a slot only while no synchronous request waits.
8. **Render.** `BrowserRenderer.RenderAsync` creates a context for this request alone and closes
   it afterwards.
9. **Result.** The PDF goes to the print service or into the response. One log line records the
   request ID, the key identifier, the status, sizes, page count, duration and the number of
   blocked resources.

## How a document is isolated

`BrowserRenderer.RenderDocumentAsync` sets up the context before it creates a page:

- **The document is served from memory.** The page navigates to
  `https://pressroom.invalid/document`, and a route handler fulfills that one request with the
  submitted HTML. Nothing is fetched. This lets the policy arrive in response headers, where it
  applies before parsing and where the document's own markup cannot remove it.
- **Headers.** A Content Security Policy (`script-src 'none'` and `sandbox`, or inline and
  `data:` scripts with `sandbox allow-scripts` when scripts are on),
  `Connection-Allowlist: ();webrtc=block;report-to=pressroom-sandbox`,
  `X-DNS-Prefetch-Control: off` and `Referrer-Policy: no-referrer`. No reporting endpoint is
  set, so reports stay in the browser.
- **Routing.** Requests for `data:` URLs continue. Every other request is aborted and counted.
  WebSockets are closed with code 1008. The context is also offline.
- **Counting.** CSP blocks some resources before they reach the route handler, so a DevTools
  audit session counts CSP URL violations too. Only the count is kept, never a URL.
- **Navigation.** `blob:` navigations bypass routing. When the main frame leaves the document
  URL, the page is closed and the render fails with 422.
- **Fonts.** After the load, the packaged `@font-face` rules are added with a script evaluation.
  Playwright's `AddStyleTag` waits for a load event that Chromium suppresses while scripts are
  off.
- **Templates.** Chromium prints header and footer templates as separate documents, so each
  template gets the resource policy and `script-src 'none'` in a `<meta>` element in front of
  the caller's markup.
- **Output.** `PdfAsync` writes no file. PdfPig counts the pages of the PDF in memory.

Request routing does not stop WebRTC. The connection allowlist does, and Chromium must support
it. The warm-up render of each new browser therefore constructs an `RTCPeerConnection` in a
trusted page and waits for the browser's `connection-allowlist` report through a
`ReportingObserver`. It gathers no ICE candidates and contacts no server. Without the report
within one second, startup fails, so a browser that ignores the header never serves a request.

**Keep exception messages out of logs and responses.** Playwright's messages can contain the
caller's HTML and URLs. The renderer replaces them with fixed text, and the server logs the
exception type only.

## Deadlines and cancellation

Playwright's calls take no cancellation token, and `Task.WaitAsync` only ends the wait: the
browser keeps working. The renderer therefore stops work by closing the context:

- On a timeout, a dropped connection or shutdown, the renderer closes the context and waits up to
  `Render:CleanupTimeoutMs`.
- If the context does not close, the renderer ends the whole browser and asks the supervisor for
  a new one.
- The render slot is released only when the context is gone or the browser's processes have
  ended. If the processes cannot be ended, the slot stays taken, no new browser starts, and the
  service reports unavailable until it is restarted. A second browser beside a stuck one would
  break the memory budget.
- The renderer applies `options.timeoutMs` itself as well, for callers that pass no cancellation
  token.

## Browser supervision

One browser serves all renders of a process. `BrowserRenderer` starts it, runs the warm-up
render, and only then reports ready.

- **Replacement after `Render:RestartAfterRenders` attempts.** New renders wait, renders in
  progress finish, the browser stops and a new one starts. Waiting requests keep their deadlines.
  `/v1/health` returns 503 in between.
- **Crash.** When the browser disconnects, renders in progress fail with 503 and the supervisor
  starts a new browser. It tries three times, then leaves the service unavailable.
- **Shutdown.** `StopAsync` has `Render:ShutdownTimeoutMs` in all. It waits for renders in
  progress for that time less two cleanup timeouts and one second, then closes the browser and
  ends its processes. The host's own shutdown timeout is five seconds longer.

The browser is launched headless with no channel, which selects the Chromium headless shell that
the Playwright package installs, and with `ChromiumSandbox` set from the settings.
`ServerConfiguration` rejects `false`.

## Process ownership

A browser that fails to close must not leave processes behind, and Pressroom must never end a
process it did not start. `OwnedDriverProcess` finds the private Node driver by comparing the
process's children before and after `Playwright.CreateAsync` and matching the packaged
executable's path.

- **Windows.** The driver is put in a Job Object with `KILL_ON_JOB_CLOSE` before it launches the
  browser, so every descendant belongs to the job. If the job cannot be created, for example
  under a restrictive application pool, startup fails.
- **Linux.** `LinuxProcessTree` records the driver's descendants from `/proc` with their start
  times, sends SIGTERM and then SIGKILL, and signals through pidfds, so a reused process ID is
  never signaled.
- Other platforms, and processes that are not x64, are not supported.

## Hosting the renderer in another process

An application that references `Pressroom.Rendering` gets a renderer with these effects:

- `BrowserRenderer` sets `TEMP`, `TMP` and `TMPDIR` for the whole process to its instance folder
  under `<StateRoot>/temp`, so the driver and Chromium write their temporary files there. Every
  later `Path.GetTempPath()` in the process returns that folder, and `StopAsync` deletes it. It
  also sets `PLAYWRIGHT_BROWSERS_PATH` and `PLAYWRIGHT_NODEJS_PATH`.
- The application folder must hold the `.playwright` driver, the browser folder and the font
  manifest, as a published server does.
- A renderer starts once. Create a new instance to start again.
- The host does the authorization. `RenderOptions.JavaScript` defaults to `true`, so set it to
  `false` for documents you do not trust.

## Jobs

- **Submission** has its own gate of `Concurrency` plus `QueueLimit` permits, so uploads of a
  batch cannot use up the admission of synchronous requests.
- **Storage.** `LocalJobStore` keeps `request.json`, `job.json` and `result.pdf` in
  `<StateRoot>/jobs/<id>`. A new job is written to a `.pending-<id>` folder and renamed. Every
  file is written under a temporary name, flushed and renamed, and on Linux the folder is flushed
  too. One lock guards every state change and the storage count.
- **One instance.** The store holds `.instance.lock` open without sharing. A second instance
  fails at startup.
- **Recovery.** At startup the store reads every `job.json`, puts queued jobs back in the queue
  and fails jobs that were rendering as `interrupted`. It refuses symbolic links and junctions,
  metadata with another `SchemaVersion` than 1, and a queued job without its request. It still
  reads files of earlier releases that contain the retired `disposition` property.
- **Scheduling.** `JobScheduler` starts the oldest queued job of the next owner in turn, when
  `RenderCoordinator` gives it a slot. A storage error fails the one job, or is logged and tried
  again. It never stops the host, because that would end synchronous rendering too.
- **Idempotency keys** are stored as SHA-256 hashes.
