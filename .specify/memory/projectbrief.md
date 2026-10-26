# Project brief

## 1. Purpose

`zpages` is a small Go library that makes a service behave as a good
citizen on Kubernetes. It reports liveness, readiness and start-up, drains
traffic before shutdown, lets an operator force states on purpose, and
changes the log level at runtime.

The owner is refreshing it for two uses:

1. Demonstrations of how Kubernetes, and platforms built on Kubernetes,
   can and should handle application states: upgrades, degraded mode,
   upstream failures, downstream failures and other states that need
   handling.
2. A base for new projects.

The owner's quality bar: light, small and easy to use.

## 2. Terms

1. Upstream: the services the application calls, its dependencies.
2. Downstream: the clients that call the application.
3. Good citizen: an application whose probes, shutdown and errors let the
   platform make the right decision without guessing.

## 3. Goals

1. Named application states, with one rule set that maps each state to
   probe results.
2. Dependency checks marked optional or critical, never tied to liveness.
3. Graceful shutdown on SIGTERM with a drain delay.
4. Operator endpoints that trigger each platform reaction on purpose,
   including forced check results, forced overload and a crash loop.
5. Runtime log level through `log/slog`.
6. Small helpers for downstream overload and trace header propagation.
7. State metrics that Prometheus can scrape without a client library.
8. The same behaviour with no mesh, with Istio sidecars and with Istio
   ambient mode, on GKE and on other Kubernetes distributions.

## 4. Non-goals

1. A tracing SDK or a metrics client library.
2. Service mesh or platform configuration.
3. Demo applications and manifests. They are a separate, later project.

## 5. Constraints

1. Go standard library only; logrus is removed.
2. The module path stays `github.com/damianjaniszewski/zpages`.
3. Minimum Go version 1.22.
4. The refresh is released as `v0.1.0` and breaks the current API.
5. Wiring a service takes a few lines and works with any router.
6. The admin listener is reached through `kubectl port-forward` only.
   There is no token in `v0.1.0`.
7. Platform behaviour is checked against current documentation and
   covered by unit tests. Tests on real clusters wait for the demo
   project.

## 6. Open decisions

1. Keep the name `zpages`, or rename the library because of the clash
   with OpenCensus and OpenTelemetry zPages. Decide before Phase 3,
   because the metric prefix and the documentation depend on it. A rename
   means a new module path, since the package name should match the last
   element of the path.

## 7. Related documents

1. `productContext.md`: why the library exists and how it should feel to
   use.
2. `systemPatterns.md`: current and target architecture.
3. `review-20261026.md`: code review and research behind the design.
4. `adr/`: architecture decision records.
5. `progress.md`: task list.
