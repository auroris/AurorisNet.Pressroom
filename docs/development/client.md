# .NET client

`AurorisNet.Pressroom.Client` calls a running Pressroom service from a .NET 8 or .NET 10
application. It builds the requests, validates them before sending, streams the PDF and turns
error responses into exceptions. Applications in other languages call the [API](api.md) with any
HTTP client.

## Installing the package

```sh
dotnet add package AurorisNet.Pressroom.Client
```

Each [GitHub release](https://github.com/auroris/AurorisNet.Pressroom/releases) also holds
`AurorisNet.Pressroom.Client.<version>.nupkg`, its `.snupkg` symbols and their `.sha256` files.
To install from the file, put it in a local feed folder such as `./packages` and add
`--source ./packages` to the command.

The package holds two assemblies, `Pressroom.Client` and `Pressroom.Contracts`, and depends on no
other package. `Pressroom.Contracts` has the request and option records, `IPdfRenderer`,
`RequestValidator` and the JSON settings, in the namespace of the same name.

## Creating a client

`PressroomClient` wraps an `HttpClient` that you own. Its base address must end with a slash, and
the API key goes in a default header:

```csharp
using Pressroom.Client;
using Pressroom.Contracts;

builder.Services.AddHttpClient<PressroomClient>(http =>
{
    http.BaseAddress = new Uri(builder.Configuration["Pressroom:Url"]!);   // https://pressroom.example/
    http.DefaultRequestHeaders.Add("X-Api-Key", builder.Configuration["Pressroom:ApiKey"]!);
    http.Timeout = TimeSpan.FromSeconds(65);                               // above the server's longest deadline
});
```

Leave the header out when the server runs in anonymous mode. Keep the URL and the key in your
application's configuration.

## Rendering a PDF

`RenderAsync` returns the whole PDF in memory with the response metadata:

```csharp
RenderedPdf pdf = await pressroom.RenderAsync(new RenderRequest
{
    Html = completeHtml,
    FileName = "orders-preview.pdf",
    Options = new RenderOptions
    {
        PageSize = PageSize.Named("A4"),
        Margins = new PageMargins { Top = "15mm", Bottom = "15mm" }
    }
}, cancellationToken);

// pdf.Bytes, pdf.FileName, pdf.PageCount, pdf.DurationMs
```

For a large document, stream it. `RenderStreamAsync` returns a `PdfRenderResult` that owns the
HTTP response. Copy its `PdfStream`, then dispose it:

```csharp
await using PdfRenderResult result = await pressroom.RenderStreamAsync(request, cancellationToken);
await result.PdfStream.CopyToAsync(destination, cancellationToken);
```

Code written against the `IPdfRenderer` interface streams too, through
`RenderAsync(string html, RenderOptions options, CancellationToken)`.

The client validates a request with the server's rules before it sends it, and throws
`ArgumentException` that names the invalid fields. A test can so catch a misspelled unit or
margins that leave no printable area. The client accepts `options.timeoutMs` up to 300000,
because it does not know the server's `Render:MaxTimeoutMs`. The server still applies its own
maximum.

## Returning the PDF to a browser

Pressroom returns the bytes and a sanitized file name. Your web application decides, with its own
response headers, whether the browser downloads or shows the PDF. For a download, use the
framework's file result:

```csharp
// ASP.NET Core MVC
return File(pdf.Bytes, "application/pdf", pdf.FileName);
```

For a preview, set the header yourself:

```csharp
// ASP.NET Core MVC
var disposition = new Microsoft.Net.Http.Headers.ContentDispositionHeaderValue("inline");
disposition.SetHttpFileName(pdf.FileName);
Response.GetTypedHeaders().ContentDisposition = disposition;
return File(pdf.Bytes, "application/pdf");
```

The typed header encodes Unicode file names. Whether the browser shows the preview depends on its
settings and its PDF support.

## Printing

`PrintAsync` sends a request that has a `Printer` identifier, and returns the receipt. It rejects
a request without a printer, and the methods that return a PDF reject a request with one:

```csharp
PrintJobResult job = await pressroom.PrintAsync(new RenderRequest
{
    Html = completeHtml,
    Printer = "shipping"
}, cancellationToken);

// job.JobId, job.Printer, job.Status == "submitted", job.PageCount
```

The client never repeats a print request. Before your application does, check the Windows or CUPS
queue, because the first request may have reached the spooler.
[Printing](../printing.md) explains the receipt and the errors.

## Batch rendering

Turn on jobs on the server first. `SubmitAsync` takes the same `RenderRequest` and an optional
idempotency key of 1 to 128 visible ASCII characters. While the server retains the job, the same
caller and key get the existing job back, even when the later request has another body. Use a new
key for new work.

```csharp
RenderJob job = await pressroom.SubmitAsync(request, "statement-10231", cancellationToken);
RenderJob completed = await pressroom.WaitAsync(job.Id, cancellationToken, TimeSpan.FromMinutes(5));
if (completed.Status == JobStatus.Succeeded && completed.Printer is null)
{
    RenderedPdf pdf = await pressroom.DownloadAsync(job.Id, cancellationToken);
    await pressroom.CancelAsync(job.Id, cancellationToken); // deletes the stored result early
}
```

| Method | What it does |
|---|---|
| `SubmitAsync` | Creates a job and returns its document. |
| `GetJobAsync` | Reads a job's current state. |
| `WaitAsync` | Polls until the job reaches a final state, at the interval the server gives in `Retry-After`: one second when absent, never more than ten. Throws `TimeoutException` when `maxWait` runs out. Cancelling the wait leaves the job on the server. |
| `DownloadAsync`, `DownloadStreamAsync` | Return the PDF in memory, or as a `PdfRenderResult` to dispose, as `RenderStreamAsync` does. |
| `CancelAsync` | Cancels a queued or rendering job, or deletes a stored result. |
| `ListJobsAsync` | Pages through the caller's jobs, optionally by status, so an application can find its jobs again after a crash. |

`RenderManyAsync` runs a batch of PDF requests. It submits, polls, downloads and deletes each
job, with at most `maxInFlight` (1 to 1024) in progress:

```csharp
await foreach (RenderManyResult item in pressroom.RenderManyAsync(requests, 8, cancellationToken))
{
    if (item.Error is { } error)
    {
        // Inspect StatusCode, Code and Retryable, and apply your own retry policy.
        continue;
    }
    await File.WriteAllBytesAsync(item.Pdf!.FileName, item.Pdf.Bytes, cancellationToken);
}
```

- Results arrive as jobs complete, and each carries its request. Exactly one of `Pdf` and `Error`
  is set.
- The client never repeats a submission or a render that failed.
- A failure of the HTTP connection gives an error with the code `transport` and no `Retryable`
  value, because the server may have accepted the job. List your jobs before you submit again.
- Leaving the loop early, or cancelling, stops the local work. Jobs the server accepted stay
  there.
- A request with a printer gives an error. Use `SubmitAsync` and `WaitAsync` for print jobs,
  whose result is a receipt.

## Errors, retries, and fallback

An error response throws `PressroomException`, with the HTTP `StatusCode`, the problem `Detail`,
the validation `Errors`, the `RetryAfter` delay, and the `Code` and `Retryable` values of a job
error:

```csharp
try
{
    return await pressroom.RenderAsync(request, cancellationToken);
}
catch (PressroomException ex) when (ex.StatusCode is HttpStatusCode.TooManyRequests or HttpStatusCode.ServiceUnavailable)
{
    // Capacity or restart: wait ex.RetryAfter ?? TimeSpan.FromSeconds(1) and retry once, or fall back.
}
catch (PressroomException ex) when (ex.StatusCode == HttpStatusCode.UnprocessableEntity)
{
    // The document failed to render. Sending the same document again will not help.
}
```

A response that is neither an error nor the expected PDF or JSON throws `HttpRequestException`.

Retries belong to the calling application, within a total deadline that suits the document. A
policy to start from:

- Retry 429 and 503 once after `RetryAfter`, with a limit on the total wait.
- Do not retry 400, 401, 413, 422 or 504.
- Treat a connection failure, or a timeout of your own `HttpClient`, as an unavailable service.

Many applications keep an **HTML fallback**: when no Pressroom URL is configured, or a render
fails, they send the same HTML to the browser and let the user print it. `IsReadyAsync` reads
`/v1/health` and returns `false` while the service is unavailable, for dashboards and startup
checks.

## Local development

Point `Pressroom:Url` at a console run on `localhost`, as in
[Getting started](../getting-started.md#4-run-it), with a key of its own or anonymous mode.

An application can also render in its own process with the `Pressroom.Rendering` project. That
project is not published as a package: reference it from a checkout and supply its browser files.
[Rendering service design](rendering-service-design.md#hosting-the-renderer-in-another-process)
lists what the renderer changes in the hosting process.
