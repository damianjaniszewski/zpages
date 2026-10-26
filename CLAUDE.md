# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## 1. Overview

`zpages` is a Go library (package `zpages`, no `main`) with HTTP handlers for
health and readiness probes and for operator "support" endpoints. Services
import it as `github.com/damianjaniszewski/zpages` and mount the handlers on
their own router. The library does not register routes or start a server.

Facts that are not visible from this directory:

1. Consumers in this GOPATH are `../go-fibo` and `../rest-collector`. Both pin
   `v0.0.0-20200216160603-350fc44dad5a` (commit `350fc44`) in `go.mod`, so a
   change here reaches them only after a push and a version bump in each
   consumer. Their `main` files show the expected wiring: route paths, HTTP
   methods and a separate listener port.
2. `$GOPATH/src/github.com/damianjaniszewski/zpages` is a second, older
   checkout (at `3c8cc74`). Edit this directory, not that one.
3. Branch `develop` is identical to `master`. Branch `origin/SWAGGERHUB`
   contains only a SwaggerHub-generated OpenAPI spec
   (`yaml-resolved/openapi.yaml`). The spec lists `/metrics`,
   `/support/v1/info` and `/support/v1/icon`, which are not implemented, and
   omits `loglevel` and `crash`. The unused `Info` type in `schemas.go` belongs
   to that unimplemented `info` endpoint.
4. `.git.orig/` is an ignored backup of an older `.git`. Do not use or delete
   it.

## 2. Build and checks

The repository has no `go.mod`, no tests, no Makefile and no CI. In-place
builds fail:

1. Module mode: `go build ./...` stops with "directory prefix . does not
   contain main module".
2. GOPATH mode (`GO111MODULE=off`): the logrus copy in
   `$GOPATH/src/github.com/sirupsen/logrus` is corrupt and `golang.org/x/sys`
   is missing.

To check a change without adding files to the repository, build a copy under
a temporary module:

```bash
D=$(mktemp -d) && cp *.go "$D" && cd "$D"
go mod init github.com/damianjaniszewski/zpages
go mod tidy
go build ./...
go vet ./...
go test -vet=off ./...
gofmt -l .
```

If a `go.mod` is ever added to the repository, the module path must stay
`github.com/damianjaniszewski/zpages`, because consumers import that path.

Results at `350fc44` (Go 1.27.3, logrus v1.10.2):

1. `go build` passes.
2. `go vet` fails on two checks. `printf`: every
   `fmt.Fprintf(w, string(out))` uses a non-constant format string.
   `copylocks`: `json.Marshal(z.ServiceStatus)` copies the `sync.Mutex`
   fields; passing `&z.ServiceStatus` avoids it.
3. Plain `go test` reports "build failed" although there are no test files,
   because it runs the `printf` vet check. Use `-vet=off` until the `Fprintf`
   calls are fixed. Run one test with `go test -vet=off -run '^TestName$' .`.
4. `gofmt -l` lists `zpages.go` (trailing blank line).

## 3. Architecture

### 3.1. Files

1. `zpages.go`: constants (service types, status strings, log formats), the
   `Handler` type and `Init`. `appName` and `appVersion` are the library's own
   name and version, printed by `Init`. The repository has no git tags.
2. `schemas.go`: JSON response types. `ServiceStatus` also carries the
   unexported probe flags and their two mutexes.
3. `handlers-probes.go`: `Healthz` and `Readyz`.
4. `handlers-support.go`: the `Support*` handlers, the exported state setters,
   `setLogLevel` and `responseHeader`.

### 3.2. Handler life cycle

Consumers build `&zpages.Handler{...}` with `Logger` (`*logrus.Logger`),
`ShutdownChannel` and `Version.Module`/`Version.Version` set, then call
`Init`.

1. `Init` logs through `z.Logger`, so a nil logger panics.
2. `Init` replaces `ServiceStatus` and `LogLevel` and appends the application
   to `Version.Dependencies`. It does not call `setLogLevel`: the logger keeps
   the level and formatter the consumer gave it until a
   `PUT /support/v1/loglevel`.
3. Consumers create `ShutdownChannel` with `make(chan os.Signal, 1)` and
   register the same channel with `signal.Notify`. `SupportQuit` sends
   `os.Interrupt` on it before it writes the response body, so a nil or full
   channel blocks the request.

### 3.3. Probe state

Health and readiness are separate state machines in `ServiceStatus`, each
under its own mutex (`mutexHealth`, `mutexReady`). Each has three flags: the
plain state (`statusHealth`, `statusReady`), forced-ok and forced-not-ok.

1. The flags set the HTTP code. The exported `Health.Status` and
   `Readiness.Status` strings set what the JSON body says. A setter must update
   both under the matching mutex.
2. Probe code precedence, first match wins: forced-not-ok 501, forced-ok 203,
   plain flag true 200, otherwise 503. The `// 202` comment in `Healthz` is
   wrong; the code returns 203.
3. The exported setters are `SetNotReady`, `UnsetNotReady`,
   `SetNotReadyForced`, `UnsetNotReadyForced` and `SetUnhealthyForced`.
   Nothing sets `statusHealthyForced` or `statusReadyForced` to true, so 203
   is unreachable. Nothing clears forced-unhealthy, so after
   `POST /support/v1/fail` health stays 501 until the process restarts.
4. `UnsetNotReadyForced` sets the body to `ready` without checking
   `statusReady`. After `SetNotReady`, the body says `ready` while the probe
   returns 503.

### 3.4. Support endpoints

Paths and allowed methods are set by consumers, not here:

1. `POST /support/v1/{quiesce,resume,quit,restart,fail,crash}` change state.
   The handlers do not check the method. `restart` sets and clears
   forced-not-ready in one call and restarts nothing. `crash` writes the
   response, then calls `log.Fatalf`, which exits the process.
2. `GET /support/v1/{version,env}` and `GET|PUT /support/v1/loglevel` act only
   on `GET`/`PUT`. Other methods, such as CORS `OPTIONS`, get the headers and
   an empty 200.
3. `env` returns the whole process environment, secrets included. It splits
   on every `=`, so it truncates values that contain `=`.
4. `loglevel` accepts `PANIC`, `FATAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`
   and `TRACE` (anything else means `INFO`) and the formats `text` and `json`.
   A debug level above 0 turns on full timestamps for `text`. On a body read
   error it sends 204 with a body; on a JSON error it sends 200 with the error
   body.
5. The handlers do not report errors from `json.Marshal` of `ServiceStatus`.
   The support handlers marshal it without locking the mutexes.

## 4. Code conventions

1. The receiver is `z *Handler`.
2. A handler starts with
   `log := z.Logger.WithFields(logrus.Fields{"app": ..., "version": ...,
   "instance": ..., "guid": ...})`, then calls `z.responseHeader(w)` (JSON
   content type and `Access-Control-Allow-Origin: *`).
3. Log messages start with `"%s <FuncName>: "` and `appName`. State-changing
   support calls log at `Warn`. Probe logging is gated by
   `z.LogLevel.Debug >= 4`.
4. Errors go back as JSON `Error{Message: err.Error(), ErrorSource: appName}`.
5. The allowed status and type strings are constants in `zpages.go` and are
   repeated in field comments in `schemas.go`. Change both together.
