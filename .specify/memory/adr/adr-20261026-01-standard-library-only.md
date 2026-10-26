# ADR: standard library only

## 1. Status

Proposed on 2026-10-26. Waiting for the owner's review.

## 2. Context

1. The owner wants the library light, small and easy to use.
2. The current code depends on logrus. The logrus README says it is in
   maintenance mode, with no new features apart from interoperability
   with `log/slog`.
3. Since Go 1.21, `log/slog` has `slog.LevelVar`, a level that can change
   at runtime and is safe for concurrent use.
4. Since Go 1.22, `http.ServeMux` matches methods and path patterns and
   answers 405 when the method does not match.
5. Consumers route with gorilla/mux. A library that returns plain
   `http.Handler` values works with any router.

## 3. Decision

1. Remove logrus. Log through a `*slog.Logger` that the application
   passes in, with `slog.Default()` as the fallback.
2. Expose a `slog.LevelVar` for runtime level changes.
3. Route the library's own endpoints with `http.ServeMux` patterns.
4. Import nothing outside the standard library.
5. Set the minimum Go version to 1.22.

## 4. Consequences

1. The API breaks: the `Logger` field type changes and `LogLevel` takes a
   new shape.
2. The log format, text or JSON, is chosen at start-up. Changing it at
   runtime would need a custom slog handler, so the feature is dropped.
3. The application must pass the same `LevelVar` to its own slog handler
   for level changes to reach its logs.
4. Metrics are written by hand in Prometheus text format, because there
   is no client library.
5. `go.mod` has no `require` lines.

## 5. Alternatives considered

1. Keep logrus: rejected, because it adds a dependency in maintenance
   mode and no feature the design needs.
2. Build on a maintained probe library such as `alexliesenfeld/health`:
   rejected for now. It adds a dependency, and the operator endpoints,
   which no library found in the research offers, would still have to be
   written here.
