# Tech context

## 1. Technologies

1. Go. Local toolchain on 2026-10-26: go1.27.3 darwin/arm64.
2. Current dependency: `github.com/sirupsen/logrus`, resolved to v1.10.2
   in a scratch build. Its README says it is in maintenance mode.
3. Target: standard library only, mainly `net/http`, `log/slog`,
   `encoding/json`, `os/signal` and `runtime/debug`.
4. Both consumers route with `github.com/gorilla/mux` v1.8.0. The library
   does not import it.

## 2. Development setup

Until a `go.mod` is committed:

1. `go build ./...` fails in place. `CLAUDE.md` section 2 has the
   scratch-copy recipe.
2. There are no tests and no CI.

Commands once `go.mod` exists:

```bash
go build ./...
go vet ./...
go test -race ./...
go test -race -run '^TestName$' ./...
gofmt -l .
```

## 3. Constraints

1. Consumers import the module path `github.com/damianjaniszewski/zpages`.
2. Consumers pin `v0.0.0-20200216160603-350fc44dad5a`, and the repository
   has no tags. `v0.1.0` may break the API, because Go modules give no
   compatibility guarantee before v1.
3. Go 1.22 is the minimum for method patterns in `http.ServeMux`.
   `log/slog` needs Go 1.21.
4. An older checkout at `$GOPATH/src/github.com/damianjaniszewski/zpages`
   (commit `3c8cc74`) must not be edited.

## 4. Platform facts the design relies on

Source: the research pass on 2026-10-26, see `review-20261026.md`
section 4.

1. The kubelet treats HTTP probe codes from 200 to 399 as success and
   reads at most 10 KiB of the body. The defaults are `timeoutSeconds: 1`
   and `failureThreshold: 3`.
2. A liveness failure restarts the container. A readiness failure removes
   the pod from EndpointSlices. Liveness and readiness probes wait until
   the startup probe succeeds.
3. On pod deletion the kubelet runs preStop, then sends SIGTERM. The
   default grace period is 30 seconds.

Not yet verified. Phase 2 starts by checking them against current
documentation; tests on real clusters wait for the demo project:

1. GKE Ingress with container-native load balancing builds its health
   check from the readiness probe; Gateway uses `HealthCheckPolicy`.
2. GKE adds a readiness gate for pods behind network endpoint groups.
3. Istio rewrites HTTP probes so the kubelet calls the Istio agent.
4. Istio ambient mode is generally available since Istio 1.24, and gives
   request-level metrics only where a waypoint proxy is deployed.
5. Kubernetes native sidecar containers fix the start and stop order
   between the application and an Istio sidecar.
6. `kubectl port-forward` reaches a port bound to `127.0.0.1` inside the
   pod.
7. The exact trace headers Istio expects applications to pass on.

## 5. Tool usage

1. Branches: `master` and `develop`. Local `develop` tracks local
   `master`, not `origin/develop`.
2. Implementation happens on `feature/<name>` branches.
3. Commit subjects are written as commands, such as `Add go.mod`, in at
   most 50 characters; bodies wrap at 72.
4. No attribution lines in commits, PR descriptions, documents or code
   comments.
