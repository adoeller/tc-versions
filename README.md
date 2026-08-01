# Versions WFX Plugin (Total Commander)

`Versions` is a Total Commander file system plugin that checks program/version strings on web pages and shows the current status directly in TC.

This repository contains an updated FPC/Lazarus-compatible codebase with reliability, parsing, and UX improvements.

![lists](lists.png)

![results](results.png)

## Key Features

- Checks version entries from `.list` files.
- Supports HTTPS and modern HTTP behavior.
- Keeps list entry order in output.
- Shows localized status labels (`NEW`, `UNCHANGED`, `ERROR`, etc.) based on selected language file.
- Works with Unicode and 64-bit builds.

## Recently Added / Improved

- Parallel page fetching with configurable worker count.
  - New INI key: `MaxParallelRequests` (default `8`).
  - Results are processed and shown after all list entries are finished.
  - Return values are evaluated per entry.
- Configurable connection timeout.
  - New INI key: `ConnectTimeout` in milliseconds (default `5000`).
- Windows-native HTTPS stack.
  - HTTP transport now uses WinHTTP instead of OpenSSL sockets.
  - Removes OpenSSL runtime DLL dependency for HTTPS in this plugin.
- Optional request logging.
  - New INI keys: `RequestLog` and `RequestLogFile`.
  - Thread-safe logging with timestamp, URL, status, and error code.
- Better handling of `.list` files.
  - Empty lines are ignored.
  - Lines starting with `;` are treated as comments and ignored.
  - In the root view, `.list` file size shows the number of valid entries (non-empty, non-comment).
- Parser improvements for markers.
  - Marker matching works across line breaks.
  - Version text is extracted with original casing preserved.
  - Found version text is limited to `64` UTF-8 bytes.
- Progress display improvements.
  - Total Commander progress now includes current count and the entry name (first field in `.list`).
- Improved request robustness.
  - Firefox-like User-Agent and browser-like headers.
  - Retry handling for transient HTTP/IO errors.
- Automatic protection-page handling.
  - Detects Cloudflare challenge/block pages, Anubis challenges, and simple JavaScript-cookie challenges from HTTP headers and HTML signatures.
  - Transparently decompresses gzip/deflate HTTP responses before protection detection and marker parsing.
  - Does not mistake Cloudflare's JavaScript detector embedded in an otherwise complete page for an active challenge.
  - Keeps WinHTTP as the normal transport and starts Chrome only for a detected protection page.
  - Uses a Chrome-version-matched normal browser User-Agent instead of the detectable HeadlessChrome identifier.
  - If an interactive Cloudflare or Anubis challenge remains, temporarily starts a normal Chrome window off-screen and reads its final DOM through a localhost DevTools connection.
  - Uses a separate persistent Chrome profile so challenge cookies can be reused.
  - Serializes Chrome calls while regular WinHTTP requests remain parallel.
- Localized error output.
  - Errors are shown with localized error extension (language-dependent), while keeping technical details in logs.

## `.list` Format

A standard entry uses six fields:

`Name|URL|StartMarker|LeftMarker|RightMarker|CurrentVersion`

A search-only entry uses three fields:

`Name|URL|Parameters`

Notes:

- Empty lines are ignored.
- Lines starting with `;` are comments.
- Marker text can span multiple lines in downloaded HTML source.
- 3-field entries are shown as `.SEARCH` (localized) and are not fetched for version parsing.

## Status And Error Codes

Result lines are shown like:

- `ProductName (value).NEW`
- `ProductName (value).UNCHANGED`
- `ProductName (CODE).ERROR` (localized extension text)

Meaning of common codes inside `(...)`:

- `403 - Forbidden`, `404 - Not Found`, `503 - Service Unavailable`, ...  
  HTTP status code returned by the server, followed by its standard reason phrase when known.
- `PARSE`  
  The configured markers could not be found/matched in the fetched page.
- `EMPTY`  
  Request finished, but no response body was returned.
- `ABORT`  
  User canceled while progress dialog was active.
- `WIN12002`  
  WinHTTP timeout (`ERROR_WINHTTP_TIMEOUT`).
- `WIN12007`  
  Name resolution failed (`ERROR_WINHTTP_NAME_NOT_RESOLVED`).
- `WIN12029`  
  Cannot connect (`ERROR_WINHTTP_CANNOT_CONNECT`).
- `WIN12030`  
  Connection error (`ERROR_WINHTTP_CONNECTION_ERROR`).
- `WIN12032`  
  Request retry required by WinHTTP (`ERROR_WINHTTP_RESEND_REQUEST`).
- `WIN12005`  
  Invalid URL (`ERROR_WINHTTP_INVALID_URL`).
- `ERROR`  
  Generic fallback when no more specific code is available.
- `403 - Forbidden - Cloudflare Protection - Chrome not found`  
  A Cloudflare protection page was detected, but Chrome is not installed or could not be located.
- `403 - Forbidden - Cloudflare Protection - Chrome still Cloudflare Protection`  
  Chrome loaded the page, but Cloudflare still returned a challenge or block page.
- `200 - OK - Anubis Challenge - Chrome background timeout`  
  Anubis was detected and the normal, off-screen Chrome fallback did not finish within the configured time.

Notes:

- `HTTP`/`EHTTPClient` style errors are from older implementations; current builds use WinHTTP and show HTTP status codes or `WINxxxxx`.
- The extension after the code is localized (for example `.ERROR` in English, translated labels in other language files).
- If `RequestLog=1`, technical details are written to `RequestLogFile` with timestamp, URL, status, and error code.

## Configuration (`versions.ini`)

The plugin reads configuration from `versions.ini` in the plugin directory.

Supported keys:

- `Language`  
  Language file without extension, e.g. `ver_English`.
- `UseLowerCase`  
  `1`/`0`, default `1`. Controls case-insensitive marker search behavior.
- `DontValidateCertificate`  
  `1`/`0`, default `0`. Disables SSL certificate validation when enabled.
- `MaxParallelRequests`  
  Integer, default `8`. Number of parallel fetch workers.
- `ConnectTimeout`  
  Integer milliseconds, default `5000`. Connection timeout per request.
- `RequestLog`  
  `1`/`0`, default `0`. Enables request logging.
- `RequestLogFile`  
  Log filename/path, default `versions_http.log`.  
  Relative paths are resolved against plugin directory.
- `ChromeFallback`  
  `1`/`0`, default `1`. Enables the automatic Chrome fallback for detected Cloudflare/Anubis pages.
- `UseChromeHeadless`  
  `1`/`0`, default `1`. Enables the first Chrome attempt with `--headless=new` and DOM output.
- `UseChromeHidden`  
  `1`/`0`, default `1`. Enables the fallback normal Chrome window, positioned off-screen, for challenges that need a full browser. Set this to `0` to prevent such a Chrome window from being started.
- `ChromePath`  
  Optional full path to `chrome.exe`. If empty, the plugin checks the Windows App Paths registry and the standard per-machine/per-user Chrome locations.
- `ChromeProfileDir`  
  Optional profile directory used only by the Chrome fallbacks. Default: `%LOCALAPPDATA%\VersionsWFX\ChromeProfile`.
- `ChromeTimeout`  
  Integer milliseconds, default `30000`, minimum `5000`. Timeout for each Chrome fallback; the headless attempt uses it additionally as JavaScript virtual-time budget.

The Chrome fallback is intentionally limited to detected protection pages. A regular HTTP error, a normal JavaScript page, or a page merely hosted behind Cloudflare continues to use WinHTTP only. `UseChromeHeadless` and `UseChromeHidden` can be set independently; with both set to `0`, detected protection pages remain WinHTTP errors with the detected protection noted.

## Build

This project is configured for Free Pascal / Lazarus (`{$mode objfpc}`).

Example build command:

```powershell
lazbuild versions.lpi
```

Typical output binary:

- `Versions.wfx64`

## Original Credits

- Original plugin author: Fabio Chelly
- Later maintenance: ProgMan13

See `readme.txt` for original historical notes and legacy changelog.
