# AurorisNet.Pressroom.Client

.NET 8 and .NET 10 HTTP client for [Pressroom](https://github.com/auroris/AurorisNet.Pressroom) (HTML-to-PDF and printing). Zero external dependencies—does not install Playwright or Chromium (requires a running Pressroom server).

## Install

```shell
dotnet add package AurorisNet.Pressroom.Client
```

## Quick start

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

Methods: `RenderAsync`, `RenderStreamAsync`, `PrintAsync`, `SubmitAsync`, `GetJobAsync`, `WaitAsync`, `DownloadAsync`, `DownloadStreamAsync`, `CancelAsync`, `RenderManyAsync`, and `IsReadyAsync`. Invalid requests throw `ArgumentException`; HTTP errors throw `PressroomException` (no automatic retries).

See the [client guide](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/client.md), [API reference](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/api.md), and [MIT License](https://github.com/auroris/AurorisNet.Pressroom/blob/main/LICENSE).
