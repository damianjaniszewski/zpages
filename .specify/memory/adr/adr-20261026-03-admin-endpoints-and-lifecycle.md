# ADR: admin endpoints and lifecycle

## 1. Status

Proposed on 2026-10-26. Waiting for the owner's final read. The owner
decided that the admin listener is reached through
`kubectl port-forward` only, with no token in `v0.1.0`.

## 2. Context

1. The handlers do not check the request method, and both consumers
   route OPTIONS to them, so a CORS preflight runs `crash`, `fail` or
   `quit`.
2. The support endpoints have no protection and send
   `Access-Control-Allow-Origin: *`. `/env` returns the whole process
   environment.
3. `quit` sends on a channel the consumer passes in. A nil or full
   channel blocks the request.
4. Both consumers listen only for `os.Interrupt`, while Kubernetes sends
   SIGTERM, so rollouts drop in-flight requests.
5. `quit` and `crash` exit before the response reaches the caller.
6. Spring Boot exposes only health over HTTP by default and keeps its
   shutdown endpoint disabled.

## 3. Decision

1. The library registers the admin routes itself, POST only for actions,
   and sends no CORS headers.
2. The admin handler runs on a separate listener that the application
   enables on purpose. The suggested address binds to `127.0.0.1`.
3. Endpoints: `quiesce`, `resume`, `fail`, `recover`, `quit`, `crash`,
   `version`, `loglevel`, check forcing and `overload`. `restart` and
   `env` are removed.
4. A crash loop is driven by the `ZPAGES_CRASH_ON_START` environment
   variable, because a switch kept in memory is lost on every restart.
5. Action endpoints answer 403 to cross-site browser requests.
6. `quit` and `crash` answer 202 and flush before they act.
7. A `Run` helper owns the lifecycle: open listeners, mark the service
   started, wait for a signal or `quit`, drain, shut down with a deadline.
   It handles SIGTERM and SIGINT. `ShutdownChannel` is removed.

## 4. Consequences

1. Each service needs one more port.
2. Presenters reach a localhost-bound admin listener through
   `kubectl port-forward`. That this works must be checked.
3. Services that need their own shutdown steps hook into `Run`, or do not
   use it.
4. Without a token, protection depends on the separate listener, the
   localhost binding and network policy.

## 5. Alternatives considered

1. Keep the admin endpoints on the probe port and add a token: not
   chosen, because kubelet probes and operator actions would share one
   exposure.
2. Leave routing to consumers: rejected, because both consumers got the
   method checks wrong.
3. Keep `env` with masked values: possible later. It is removed for now,
   because nothing in the design needs it.
