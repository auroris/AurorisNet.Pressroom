# Pressroom.Contracts

The request and response types of the Pressroom HTML-to-PDF API, for .NET 8 and .NET 10:
`RenderRequest`, the page and readiness options, `IPdfRenderer`, `PdfRenderResult`, the job
records, `RequestValidator` and `JsonDefaults`. The project uses the base class library only.

The client, the server and the renderer share this project. It is not published as a package of
its own: `AurorisNet.Pressroom.Client` holds `Pressroom.Contracts.dll` for each target framework.

```shell
dotnet add package AurorisNet.Pressroom.Client
```

The [client README](https://github.com/auroris/AurorisNet.Pressroom/blob/main/src/Pressroom.Client/README.md)
shows how to use the package, and the
[API reference](https://github.com/auroris/AurorisNet.Pressroom/blob/main/docs/development/api.md)
describes each field.
