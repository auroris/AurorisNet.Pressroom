# Windows and IIS release acceptance

Qualify a release on the target IIS server by working through this checklist with the published
files. The build checks each release on the build machine only, so the server's prerequisites,
the application pool's permissions and the rendering there are checked here. The checklist covers
direct IIS hosting and physical printing. For a container, use
[Check a running container](container.md#check-a-running-container).

## Record the environment

Record these with the result, and keep the exact release archive with them:

- the release manifest, the browser version and the font manifest;
- the Windows Server edition and build;
- the .NET, ASP.NET Core and Hosting Bundle versions;
- the IIS configuration and the pool identity;
- the Visual C++ runtimes installed on the machine, and whether the Media Foundation feature is
  installed;
- the configured rendering limits.

Prepare the server as follows:

- Start from a clean, supported Windows Server with IIS and the .NET 10 Hosting Bundle. Record the
  components that the Hosting Bundle itself installs separately from the rest.
- Leave Media Foundation out for the standard qualification.
- If the test fails for a missing dependency, record exactly which one before you install a
  machine-wide Visual C++ runtime to make it pass.
- Use a location that has no Playwright browser cache of a developer and no
  dependency-validation markers.
- Give the application no Node, npm, .NET SDK, PowerShell 7 or download access.
- Grant the real pool identity read and execute on the release, and write on its external state
  folders only.
- Leave Playwright's dependency validation on and `Render:ChromiumSandbox` `true`.

## Cold start and published payload

- Start the IIS application and confirm that `/v1/health` returns 200 after warm-up, and that
  `/v1/info` reports the expected release, browser and font versions.
- Send the sample request
  [`orders-bilingual.request.json`](https://github.com/auroris/AurorisNet.Pressroom/blob/main/samples/orders-bilingual.request.json)
  through the site. Verify a PDF response with two pages, Letter portrait dimensions, page
  numbers, the vector graphic, bold and italic faces, French accents and ligatures, an outline,
  and tags. Look at the PDF, and inspect it with your PDF tool.
- Verify `application/pdf` and a `Content-Disposition: attachment` header with a safe file name,
  also for a Unicode file name. A request with a property the API does not define, such as
  `disposition`, must return 400.
- Verify an image and a font that the caller embeds, beside the packaged Noto Sans faces.
- Verify header and footer text separately, with a generic font family. The page's CSS and fonts
  do not reach Chromium's header and footer templates, and fonts embedded in a template are not
  supported.
- Restart the pool and repeat. Confirm that the browser and driver processes of the old worker
  exit, and that other browsers on the machine are untouched.
- Confirm that the Windows Job Object works under the real IIS identity, including any nesting
  that IIS adds. Pressroom puts its private Node process in a job before the browser starts and
  keeps every descendant in it. If it cannot, startup must fail. Exercise shutdown and a browser
  crash, and confirm that the job ends with no active process.
- Confirm that the pool identity cannot write to the release folder, and that startup still
  succeeds without writing a dependency-validation marker there.

## Authentication through IIS

Enable IIS Anonymous Authentication and disable Windows Authentication. Set each access mode in
the external JSON file, and restart between modes. Check the status codes and the
`WWW-Authenticate` headers.

| Mode and request | Expected |
|---|---|
| `ApiKey`: valid configured key | 200 on `/v1/info`, and a successful render |
| `ApiKey`: no key | 401, also for a caller with Windows credentials |
| Either mode: invalid, empty, repeated or oversized key | 401, with no Negotiate or NTLM challenge |
| `Anonymous`: no key | A successful render, `/v1/info` and the OpenAPI document |
| `Anonymous`: valid configured key | Success |
| `ApiKey`: no keys configured | Startup fails with a configuration error |
| Invalid key settings, or two keys with one secret | Startup fails with a configuration error |
| Either mode: key without `AllowJavaScript`, request `javaScript` omitted or `true` | Static render. Document scripts do not run. |
| `Anonymous`: no key, request `javaScript` omitted or `true` | Static render. Document scripts do not run. |
| Either mode: key with `AllowJavaScript: true`, request `javaScript` omitted or `true` | Chart scripts and the readiness expression run |
| Either mode: key with `AllowJavaScript: true`, request `javaScript: false` | Static render. Document scripts do not run. |

Also verify:

- `/v1/health` is anonymous in both modes, and `/v1/info` and the OpenAPI document need a valid
  key in API-key mode.
- With scripts off, a document with inline scripts, `data:` scripts, event handlers and child
  frames runs none of them, while its static text, embedded images, CSS and packaged fonts
  render.
- A readiness function with scripts off returns 400, also when the request says
  `javaScript: true` and the key lacks the permission.
- After you change a rendering limit and a key in the external file and restart, both changes
  apply.
- The application reads the file that `ConfigurationFile` names, fails to start when that file is
  missing, and takes no application setting from an environment variable.

## Printer output

**Desktop test.** Set `Printing:SumatraPdfPath` to an installed or portable SumatraPDF, and add
`Printing:Printers:pdf-test` with the value `Microsoft Print to PDF`. Start Pressroom from a
console, with an API key. Send the sample request with `"printer": "pdf-test"` and a timeout long
enough to complete the printer's **Save Print Output As** dialog. Save to an absolute path
outside the temporary job folder. Expect 202 with JSON: `status: submitted`, a `jobId`,
`printer: pdf-test` and `pageCount: 2`. Inspect both pages of the saved file. Use the `jobId` to
match Pressroom's log line with the Windows queue.

**Production.** Repeat with a real printer under the dedicated IIS identity, which needs print
permission and access to SumatraPDF. `Microsoft Print to PDF` asks for a file name, so it does not
suit IIS.

- Verify an unavailable queue, denied permission, a cancelled request and a request timeout.
  After each, the print command has exited and the temporary PDF is gone.
- Pages already queued can still print after a cancellation. Look at the Windows queue before you
  send again.
- Verify that `GET /v1/printers` and print requests need a valid key in anonymous mode too, and
  that PDF requests still work.

## Limits, cancellation and resources

- **Body size.** Send bodies near the 25 MiB application limit and above the 32 MiB host limit.
  Measure the whole UTF-8 JSON body, with the escaped HTML and templates. Over the application
  limit, expect 413 with problem details. Over the host limit, IIS may answer with its own
  response.
- **Capacity.** Fill the active and waiting capacity. Verify 429 with `Retry-After`, and that
  `/v1/health` stays 200.
- **Cancellation.** Cancel waiting and active requests, including an expensive script and a print
  request with a key that allows scripts. Confirm that the capacity comes back and that later
  renders complete.
- **Failures.** A failed readiness condition returns 422. A passed deadline returns 504 and ends
  the request's work in the browser.
- **Blocked resources.** From test documents, try HTTP and HTTPS, local files, frames and popups,
  CSS imports, fetch and XHR, WebSockets and service workers. Send the script-based probes with a
  key that allows scripts, so they run. Confirm that each is blocked, and that the logs hold no
  URL, document text, page error, console text, PDF content or key.

A requirement that the server makes no outbound connection at all also needs a network policy
enforced outside Pressroom.

## Recycle, update, rollback and soak

- **Soak.** Run at least 1,000 representative renders at the configured concurrency, past the
  point where Pressroom replaces the browser. Record errors and retries, peak memory, process
  counts, response times and memory after idling. Include slow clients, so that responses still
  being downloaded count against admission. Confirm that memory and process counts stay bounded.
- **Recycle.** Recycle the pool during traffic, and verify readiness, recovery and cleanup. Test a
  browser crash and its replacement. Allow memory for overlapped worker processes, or choose
  recycling without overlap.
- **Update and rollback.** Deploy an update with your normal procedure, then restore the whole
  previous release the same way. After each, verify readiness, the version, the access mode and a
  PDF. Settings, logs and temporary state stay outside the site folder. Do not count a partial
  copy or a failed warm-up as a deployment.

## Asynchronous jobs, when enabled

- Submit a batch of several hundred documents while you measure the 95th-percentile latency of
  synchronous requests. Aim for at most twice the idle latency, with no added 429 on synchronous
  requests.
- Restart with queued and active jobs. Queued jobs resume, and the interrupted render fails as
  retryable. After an interrupted print job, check the Windows spooler.
- Verify that two keys cannot list, read, download or delete each other's jobs.
- Exercise the global queue limit and the limit of one key, the storage budget, cancellation,
  expiry and deletion on download. Expired content is gone within two minutes. A result that is
  being downloaded stays until the download ends.
- Inspect the access lists on the job folders, which name the service identity only. Confirm that
  the logs hold metadata only and that a second instance cannot open the store.
- Disable overlapped recycling. Test updates and rollback by stopping and starting, without
  replacing the jobs folder.

## Result

Record **passed**, **failed** or **not run** for each section, with the exact release archive, the
machine inventory, the measurements and the limits that remain. A release is qualified when these
checks pass on the target server with Playwright's dependency validation on. Qualify again after
a change of browser or of native runtime components.
