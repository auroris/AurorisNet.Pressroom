# Printing

Pressroom can render a document and hand its pages to a printer on the server. The request is an
ordinary render request with a `printer` identifier and an API key, and the response is a JSON
receipt. This page covers the host setup, the allowlist, and what the receipt and the errors mean.

## Requirements

- **Windows: SumatraPDF.** Install [SumatraPDF](https://www.sumatrapdfreader.org/free-pdf-reader),
  or place its portable executable in a folder the service can run it from. Pressroom does not
  include it.
- **Linux: CUPS and its command-line client**, with a working queue that accepts PDF. Pressroom
  runs `/usr/bin/lp` by default, or `lpr` when you configure it. The container image has neither,
  so print from a Pressroom installed on the host.
- **An API key.** Every configured key may print. A caller without a key cannot print or list
  printers, also when `Auth:Mode` is `Anonymous`.
- **Printers that the service identity can use.** A printer installed for an interactive user
  only may be missing for the IIS application pool or the Linux service user.

## Configure the allowlist

The allowlist starts empty. Each key is the identifier callers send: 1 to 128 ASCII letters,
digits, underscores and hyphens, matched without regard to case. Each value is the Windows printer
name or the CUPS destination. Add printers in the external settings file, then restart Pressroom.

### Windows

Set the SumatraPDF path and the Windows printer names:

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

`SumatraPdfPath` must be an absolute path that ends in `.exe`. Startup fails when printers are
configured without it.

Grant the application pool identity the right to run SumatraPDF, print permission on the queues,
and Modify on `temp` under `Paths:StateRoot`. Set paper, tray, duplex and the other device
defaults on the Windows printer: the render options decide what is on each page, and SumatraPDF
places the pages on paper with the printer's own settings. Check the output on the device you will
use.

### Linux host

Install CUPS and its client tools from your distribution, then set up and test the queue. Run
`lpstat -p -d` as the Pressroom service user to see which queues that user can reach.

```json
{
  "Printing": {
    "CupsCommandPath": "/usr/bin/lp",
    "Printers": {
      "shipping": "Warehouse_Laser"
    }
  }
}
```

`CupsCommandPath` defaults to `/usr/bin/lp`. Set it to `/usr/bin/lpr` to use `lpr`. It must be an
absolute path to an executable named `lp` or `lpr`, without arguments. `SumatraPdfPath` is ignored
on Linux, and `CupsCommandPath` on Windows.

The service user must be able to run the command, submit to the queue without a password prompt,
and write under `Paths:StateRoot`. Set paper, tray, duplex and other defaults in CUPS for that
user. Pressroom submits the PDF through the queue's normal filters, so the queue must be able to
convert PDF for the device. Before you use the API, print a PDF as the service user:

```sh
sudo -u pressroom lp -d Warehouse_Laser /absolute/path/to/test.pdf
```

## Submit a print job

Send a valid `X-Api-Key` and add `printer` to a render request:

```json
{
  "html": "<!doctype html><html><body><h1>Shipping document</h1></body></html>",
  "printer": "shipping",
  "options": { "pageSize": "Letter" }
}
```

`fileName` has no effect on printed output. Without `printer`, or with `"printer": null`, the
response is the PDF as usual.

Pressroom renders the document, writes the PDF to a new folder under `<StateRoot>/temp`, and
runs the print command on it:

| Host | Command |
|---|---|
| Windows | SumatraPDF with [`-print-to`](https://www.sumatrapdfreader.org/docs/Command-line-arguments) and the printer name, `-silent`, and a private settings folder |
| Linux, `lp` | [`lp -d <destination> -- <file>`](https://www.cups.org/doc/man-lp.html) |
| Linux, `lpr` | [`lpr -P <destination> <file>`](https://www.cups.org/doc/man-lpr.html) |

The printer name comes from the allowlist, never from the request, and Pressroom passes it and the
file path as separate arguments without a shell. Only the service user can open the folder: it
has mode 0700 on Linux, and on Windows an access list that names that identity alone. Pressroom
removes the PDF after the command exits.

When the command exits with success, the response is **202 Accepted** with `application/json`:

```json
{
  "jobId": "64e5b7e07c8e4bdb86521bed5cd54d4c",
  "printer": "shipping",
  "status": "submitted",
  "pageCount": 1
}
```

`submitted` means the print command accepted the document. Follow the progress, an offline
device or a later spooler error in the Windows or CUPS queue. `jobId` is Pressroom's own
identifier, which also appears in its log. It is not the spooler's job number.

`GET /v1/printers` returns the configured identifiers to a caller with a key, for example
`{"printers":["pdf-test","shipping"]}`. It does not report whether a queue works.

With asynchronous jobs enabled, `POST /v1/jobs` accepts the same `printer` field under the same
key requirement. A print job that succeeds holds a `printJob` receipt and no PDF to download.

## Errors and retries

| Status | Meaning |
|---|---|
| 400 | The printer identifier is not in the allowlist. Checked before rendering. |
| 401 | The API key is missing or invalid. |
| 502 | The print command exited with an error. |
| 503 | The print command is missing or cannot start, or the temporary folder cannot be written. |
| 504 | The processing deadline passed during rendering or submission. |

The admission limit and the processing deadline cover rendering and submission together. On a
timeout or a cancelled request, Pressroom stops the print command and removes the temporary file
once the command has exited.

**A timeout, a dropped connection or a failure can come after some or all pages have reached the
spooler.** Check the queue before you send the request again. Pressroom never repeats a print
request, and neither does the .NET client's `PrintAsync`.

## Local test

`Microsoft Print to PDF` works as a test printer on a desktop. Configure it as `pdf-test`, and
complete its **Save Print Output As** dialog before the request times out. It asks for a file
name, so it needs a person at the screen. Use a queue that needs no input under IIS.

The [Windows and IIS acceptance checklist](windows-acceptance.md) and the
[Linux acceptance checklist](linux.md#acceptance-checklist) hold the printer checks for a
deployment.
