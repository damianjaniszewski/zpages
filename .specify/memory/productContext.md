# Product context

## 1. Why the library exists

The original `zpages`, written in 2019 and 2020, gave two of the owner's
services, `go-fibo` and `rest-collector`, health and readiness probes and
operator "support" endpoints. Both services still pin commit `350fc44`.

In 2026 the owner is refreshing it to show how Kubernetes, and platform
services such as Istio and GKE load balancing, should react to the states
an application goes through, and to use it in new projects.

## 2. Problems it solves

An application that reports its state badly pushes the platform into the
wrong decision. Common cases:

1. Liveness tied to a database: one database outage restarts every pod.
2. Readiness that fails on every pod when a shared dependency is down: a
   partial outage becomes a total one.
3. No drain on SIGTERM: rolling updates drop in-flight requests.
4. Status codes the platform does not understand, such as 203 or 501.
5. Admin endpoints that anyone on the pod network can call.
6. Traces that break at every hop because the application does not pass
   trace headers on.

## 3. States and expected platform reactions

1. Starting: start-up and readiness fail. The platform sends no traffic
   and does not restart the pod.
2. Ready: all probes pass. The platform routes traffic.
3. Degraded, an optional dependency is down: readiness passes and the
   body says `warn`. The platform keeps routing; the application runs
   with reduced function.
4. Unavailable, a critical dependency is down: liveness passes and
   readiness fails. The platform stops routing to this pod and does not
   restart it.
5. Overloaded by downstream callers: the application rejects excess
   requests with 503 and `Retry-After`. Autoscaling or mesh outlier
   detection reacts.
6. Draining, for an upgrade, scale-in or node drain: readiness fails,
   liveness passes, in-flight requests finish, the process exits.
7. Quiesced by an operator: readiness fails and the pod keeps running for
   debugging.
8. Failed, hung or broken: liveness fails and the kubelet restarts the
   container.
9. Crashed: the process exits and the kubelet restarts it. If it keeps
   crashing, the pod shows `CrashLoopBackOff` while the kubelet backs off.

Mark a dependency critical only when this pod cannot work without it
while other pods still can, for example a local file or a per-pod
connection. A dependency that every pod shares, such as one database, is
usually better as optional: the pod stays ready, fails those requests
fast and lets mesh timeouts and circuit breaking act.

## 4. How it should work for a developer

A service creates one handler, registers its checks, mounts the probe and
admin handlers on its own servers and passes the servers to a lifecycle
helper. The sketch below is a proposal, not implemented code. Names can
change during planning.

```go
z := zpages.New(zpages.Options{DrainDelay: 5 * time.Second})
z.AddCheck("cache", zpages.Optional, checkCache)
z.AddCheck("db", zpages.Critical, checkDB)

api := z.LimitInFlight(100, zpages.PropagateTrace(mux))

app := &http.Server{Addr: ":8080", Handler: api}
mgmt := &http.Server{Addr: ":8081", Handler: z.ProbeHandler()}
admin := &http.Server{Addr: "127.0.0.1:8082", Handler: z.AdminHandler()}

if err := z.Run(ctx, app, mgmt, admin); err != nil {
    log.Fatal(err)
}
```

## 5. Experience goals

1. Developers: a few lines of wiring, no new dependencies, any router.
2. Operators and presenters: every state can be set on purpose and seen
   in probe results, logs and metrics. That includes forced check
   results, forced overload and a crash loop.
3. Platforms: plain 200 and 503 codes, and a JSON body people can read.
