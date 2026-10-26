# Progress

## 1. What works

At commit `2eeb805`; the Go code is unchanged since `350fc44`:

1. Health and readiness endpoints with forced states, used by `go-fibo`
   and `rest-collector`.
2. Support endpoints: quiesce, resume, fail, quit, restart, crash,
   version, env and loglevel.
3. `go build` passes in a scratch copy with a temporary module.

## 2. Known issues

`review-20261026.md` section 2 has the full list with file and line
references.

1. With the consumers' routing, an OPTIONS request runs the support
   actions.
2. The support endpoints have no protection and send
   `Access-Control-Allow-Origin: *`; `/env` exposes secrets.
3. Data races between probes, setters and log level changes. Probes write
   the response while a mutex is locked.
4. Application setters clear operator overrides.
5. JSON bodies are passed to `Fprintf` as format strings.
6. No `go.mod`, `go vet` fails and there are no tests.

## 3. Task list

### 3.1. Phase 0: baseline

A build baseline only, with no tag. `go-fibo` and `rest-collector` stay
on `350fc44` with the known issues.

1. [ ] Add `go.mod` and `go.sum` for the current code.
2. [ ] Fix the `go vet` and `gofmt` findings in the current code.

### 3.2. Phase 1: state and probes

1. [ ] State type with phase, operator switches, check results, heartbeat
   and the precedence rules, with tests.
2. [ ] Dependency checks with background runs, timeouts, cached results
   and `Report`, with tests.
3. [ ] Probe handler for `/livez`, `/readyz`, `/startupz` and `/healthz`
   with the JSON body, with tests.
4. [ ] Heartbeat for hang detection, with tests.
5. [ ] Forced check results, stored apart from the real results, with
   tests.

### 3.3. Phase 2: operator and lifecycle

1. [ ] Check the unverified platform facts in `techContext.md` section 4
   against current documentation, and adjust the defaults below.
2. [ ] Admin handler: quiesce, resume, fail, recover, quit, crash,
   version, check forcing and overload, with the cross-site check, with
   tests.
3. [ ] Log level endpoint on `slog.LevelVar`, with tests.
4. [ ] `Run`: signals, drain delay, shutdown deadline and server errors,
   with tests.
5. [ ] `ZPAGES_CRASH_ON_START` in `Run`, with tests.

### 3.4. Phase 3: traffic helpers and metrics

1. [ ] In-flight limiter as a handler method, with the overload switch,
   with tests.
2. [ ] Trace header propagation middleware and `RoundTripper`, with
   tests.
3. [ ] `/metrics` in Prometheus text format, with tests.

### 3.5. Phase 4: release

1. [ ] Remove the old API and logrus.
2. [ ] Package documentation and runnable examples.
3. [ ] Update `CLAUDE.md` and this memory bank.
4. [ ] Tag `v0.1.0`, when the owner decides.

### 3.6. Open decisions

1. [ ] Library name, before Phase 3.
2. [x] Optional token for the admin listener: not in `v0.1.0`. Access is
   through `kubectl port-forward` only.

### 3.7. Future work, not scheduled

1. [ ] Demo project: applications and manifests for plain Kubernetes, GKE,
   and Istio in sidecar and ambient mode.
2. [ ] Switches that reproduce common mistakes for the demo, such as
   liveness tied to a dependency.
3. [ ] A gRPC health service driven by the same state.
4. [ ] Move `go-fibo` and `rest-collector` to `v0.1.0`, and make them
   handle SIGTERM.
5. [ ] CI that runs `go vet`, race tests and `gofmt`.
6. [ ] A token, so the admin listener can bind to the pod IP for access
   from other pods.
7. [ ] Tests of the platform behaviour on GKE with Istio in sidecar and
   ambient mode, as part of the demo project.

## 4. Evolution of decisions

1. 2019 to 2020: the original library, with logrus and forced-state
   flags.
2. 2026-10-26: refresh started. A redesign was chosen over patching the
   flag model; see `adr/`.
3. 2026-10-26: the owner chose forced checks, forced overload and a crash
   loop; documentation and unit tests for platform behaviour; Phase 0
   without a tag; admin access through port-forward only.
