# Security model

Pressroom renders HTML that your applications send, in a browser on a server you control. This
page describes the controls Pressroom applies to documents, callers and resources, and the
controls the deployment must add.

## Who is trusted

- **Calling applications** are trusted and identified by API keys. They own their templates, the
  authorization of their users, and what goes into a document.
- **Documents** are not trusted. Every document, including templates and any user input inside
  them, renders under the controls in the next section.
- **The server** holds the keys and the browser. Run it behind HTTPS under an identity with the
  least rights it needs.

## Document isolation

Each render gets its own browser context, with its own cookies and storage, and Pressroom closes
the context when the request ends.

| Control | Effect |
|---|---|
| Resource policy | Only `data:` resources load. Network, local-file and `blob:` URLs are blocked, also inside nested documents. Pressroom serves the document from memory with its policy in response headers, so the policy applies before the browser parses any markup. |
| Content Security Policy | Allows inline styles and `data:` resources. Denies workers, objects, forms and base URLs. Sandboxes the document without same-origin or popup rights. |
| Connection allowlist | An empty allowlist blocks connections that the document starts outside ordinary requests, including WebRTC, WebTransport, DNS prefetch and preconnect. At startup Pressroom checks that the browser enforces it, and stops when it does not. |
| Offline context and interception | The context is offline, every request that is not `data:` is aborted, and WebSockets are closed. |
| Disabled features | Downloads, popups, forms, service workers, and storage tied to an origin. |
| `<base>` elements | A request whose document or templates contain one gets 400, so relative URLs cannot point at another origin. |
| JavaScript | Off unless the API key has `AllowJavaScript: true`. A request can turn scripts off, never on. |
| Navigation | When the main document navigates away from the submitted HTML, Pressroom closes the page and the render fails. |
| Chromium sandbox | Always on. Pressroom refuses to start with `Render:ChromiumSandbox` set to `false`, or when the sandboxed browser cannot start. |

One case fails a render in the packaged Chromium: a header or footer template that tries to load
an external resource is either rejected or runs into the request deadline. The caller gets an
error without document content, the context closes, and later renders work.

A WebRTC attempt is blocked by the connection policy, in the document and inside a `srcdoc` child
frame alike. It sends nothing, and the render goes on.

## Authentication

- API-key mode requires a valid `X-Api-Key` on the render, information, OpenAPI, printer and job
  routes. `/v1/health` is always anonymous, so IIS and probes can reach it.
- Pressroom compares secrets in constant time and requires each key to have a secret of its own.
  Logs name a key by its identifier.
- An invalid, empty, repeated or oversized key header gets 401 in either mode. The 401 carries no
  `WWW-Authenticate` challenge. Under IIS, Pressroom turns off automatic Windows authentication,
  so a Windows identity grants nothing.
- Anonymous mode must be selected explicitly. Callers without a key render without scripts and
  cannot print.

## Retention and logging

- A synchronous PDF request keeps its HTML and PDF in memory, page counting included.
- A print request writes the PDF to a new folder under `<StateRoot>/temp` for SumatraPDF or CUPS
  and removes it after submission. Only the service identity can open the folder: mode 0700 on
  Linux, and on Windows a protected access list for that identity. The operating system's spooler
  can keep the data longer.
- Logs hold metadata only: identifiers, sizes, timings, page counts, status codes and counts of
  blocked resources. They never hold document content, PDF bytes, secrets, blocked URLs, or the
  page's console and error text. A 422 response carries a fixed message with no text from the
  page.
- Chromium can write temporary files in its private working folder under `<StateRoot>/temp`, and
  the operating system can page memory to disk.

## Job storage

With jobs enabled, the HTML and PDFs live in `<StateRoot>/jobs/<id>` for the
[configured retention](configuration.md#asynchronous-jobs). The folders and files are for the
service identity only: modes 0700 and 0600 on Linux, and a protected access list on Windows. Use
volume encryption if the content must be encrypted at rest. One instance at a time can open the
store.

- **Lifetime.** Pressroom deletes a job's HTML when the job succeeds. The HTML of a failed job and
  the PDF of a finished one follow the retention settings. With storage that survives a restart,
  queued jobs resume, and a job that was rendering becomes failed with the code `interrupted`. An
  interrupted print job may already have reached the spooler, and cancelling a job cannot take
  spooled pages back.
- **Owners.** A job belongs to the identifier of the API key that submitted it. Another key gets
  404 for that job and cannot list it. A new secret under the same identifier keeps the access,
  and an identifier that is used again gets the jobs of its earlier use.
- **Anonymous jobs.** All callers without a key share one owner, so each of them can list, read
  and delete those jobs. Use API-key mode for jobs.

## Containers

The Linux container runs as a non-root user with a read-only root file system and memory-backed
state. The Windows container runs as `ContainerUser`, and its helper removes the container and
its writable layer when it stops. [Containers](container.md) explains both. Blocking outbound
connections is the work of a firewall or a NetworkPolicy.

## Resource protection

- The application limits the JSON body to 25 MiB by default and the host to 32 MiB. Pressroom
  admits a request before it reads the body, so a burst of requests cannot make it hold
  unlimited HTML.
- By default two renders run and eight requests wait. A request beyond that gets 429 with
  `Retry-After`, and `/v1/health` stays 200.
- Each request has one deadline for all its processing: 30 seconds by default and at most 60. A
  timeout or a dropped connection closes the browser context.
- When a context does not close in time, Pressroom ends the browser's processes and starts a new
  browser. If it cannot confirm that the old processes are gone, it starts none, and the service
  stays unavailable until it is restarted.
- Pressroom replaces the browser after a set number of renders, once the renders in progress have
  finished.

## Deployment recommendations

- **Internet-facing use.** Use the [Linux container](container.md#optional-linux-outbound-isolation)
  with its outbound blocking, API keys, and an HTTPS reverse proxy in front of the loopback-bound
  API. A [Linux host installation](linux.md) also suits, under a dedicated unprivileged user with
  API keys and HTTPS.
- **Intranet use.** Windows with IIS and Windows containers are meant for internal networks. Both
  Linux options work there too. Use HTTPS and a key for each application.
- Under IIS, give Pressroom an application pool identity of its own, with read and execute rights
  on the release folder and the state folder rights in [Deployment](deployment.md).
- Keep the settings file outside the release folder, readable by the service identity and
  administrators only. Keep secrets out of `web.config`, `appsettings.json` and publish profiles.
- Give each calling application its own key, and set `AllowJavaScript` only where documents need
  scripts.
- Before each deployment, check the archive against its checksum. `release-manifest.json` lists
  the hash of every file in the release.

<a id="what-is-not-guaranteed"></a>

## Deployment responsibilities

- **Outbound connections of the server's processes.** Pressroom's controls work inside the
  browser and the application. To stop the processes themselves from connecting out, use a host
  network policy that you have tested. [Containers](container.md) gives iptables rules for Linux
  and HNS rules for Windows.
- **Browser updates and host isolation.** Chromium's sandbox and the separate context of each
  render contain a document. Keep the release current and isolate the server, to limit what a
  browser exploit could reach.
- **HTML sanitization.** Validate and sanitize content in the calling application. Pressroom
  rejects `<base>` and blocks resource loading. It does no other sanitization.
- **PDF/UA conformance.** Pressroom asks Chromium for a tagged PDF by default. Whether the result
  conforms depends on the source HTML, so check the HTML and the PDF against your accessibility
  requirements.

The release workflow runs automated tests of these controls against the packaged Chromium on
every release. The [Windows and IIS acceptance checklist](windows-acceptance.md) and the
[Linux acceptance checklist](linux.md#acceptance-checklist) list the checks to repeat on the
target server.
