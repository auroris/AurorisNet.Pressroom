# AurorisNet.Pressroom.Client

The .NET client for [Pressroom](https://github.com/auroris/AurorisNet.Pressroom), a self-hosted
service that turns a complete HTML document into a PDF or sends it to a configured printer.

The package calls a running Pressroom server over HTTP from a .NET 8 or .NET 10 application. It
holds the `Pressroom.Client` and `Pressroom.Contracts` assemblies and depends on no other package.
Rendering happens on the server: set one up first with the
[getting started guide](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/getting-started.md),
from a [server release](https://github.com/auroris/AurorisNet.Pressroom/releases/latest).

## Install

```shell
dotnet add package AurorisNet.Pressroom.Client
```

Each [GitHub release](https://github.com/auroris/AurorisNet.Pressroom/releases) also holds the
`.nupkg` file, for a local feed.

## Use

Give the client an `HttpClient` with your server's URL, ending in a slash, and your API key:

```csharp
using Pressroom.Client;
using Pressroom.Contracts;

using var http = new HttpClient { BaseAddress = new Uri("https://pressroom.example/") };
http.DefaultRequestHeaders.Add("X-Api-Key", apiKey);
var pressroom = new PressroomClient(http);
var pdf = await pressroom.RenderAsync(new RenderRequest
{
    Html = "<!doctype html><html><body><h1>Hello</h1></body></html>",
    FileName = "hello.pdf"
});
await File.WriteAllBytesAsync(pdf.FileName, pdf.Bytes);
```

Reuse one `HttpClient`, or register a typed client with `AddHttpClient<PressroomClient>`.

| Method | What it does |
|---|---|
| `RenderAsync` | Returns the PDF in memory with its file name, page count and render time. |
| `RenderStreamAsync` | Returns a `PdfRenderResult` that streams the PDF. Dispose it after copying. |
| `PrintAsync` | Sends the document to a configured Windows or Linux printer and returns the receipt. |
| `SubmitAsync`, `GetJobAsync`, `WaitAsync` | Create a job on a server with jobs enabled, read its state, and wait for it to finish. |
| `DownloadAsync`, `DownloadStreamAsync`, `CancelAsync` | Fetch a job's PDF, and cancel a job or delete its result. |
| `RenderManyAsync` | Runs a batch through jobs with a limit on jobs in progress, and yields a PDF or an error for each request. |
| `IsReadyAsync` | Reads the server's health route. |

The client validates a request before it sends it and throws `ArgumentException` for invalid
values. An error response throws `PressroomException` with the HTTP status, the problem details
and the `Retry-After` delay. The client never retries a failed request: retries belong to your
application.

The [client guide](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/client.md)
covers streaming, printing, batches and error handling, and the
[API reference](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/api.md)
lists every request option.
