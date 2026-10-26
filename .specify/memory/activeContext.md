# Active context

## 1. Current focus

Refreshing `zpages` into a small, standard-library-only library for good
citizen applications on Kubernetes. The `v0.1.0` design is written in
`systemPatterns.md` section 2 and in `adr/`. It was updated with the
owner's answers to the design questions and waits for the owner's final
read. No product code has changed yet.

## 2. Recent changes on 2026-10-26

1. Added `CLAUDE.md` with build notes and the current architecture.
2. Fast-forwarded `develop` to `master`. Nothing pushed.
3. Ran an independent code review and a research pass. Results are in
   `review-20261026.md`.
4. Created this memory bank.

## 3. Next steps

1. The owner gives the design a final read. The name is decided before
   Phase 3.
2. Write the detailed implementation plan for the task list in
   `progress.md`.
3. Implement on a `feature/` branch, one task per commit.

## 4. Decisions

Stated by the owner:

1. Purpose: demonstrations of how Kubernetes and platform services
   handle application states, and a base for new projects.
2. Light, small and easy to use.
3. Upstream means the services the application calls; downstream means
   its callers.
4. Platforms: mostly GKE, but any Kubernetes with platform services.
   Istio matters, in both sidecar and ambient mode.
5. Work on the library now; the demo is a later project.
6. The memory bank is tracked in git.
7. Scope includes `go.mod`, vet fixes, the probe state defects and tests.
8. Operators can force check results, overload and a crash loop.
9. Platform behaviour is checked against current documentation and
   covered by unit tests. Tests on real clusters wait for the demo
   project.
10. Phase 0 is a build baseline with no tag. `go-fibo` and
    `rest-collector` stay on `350fc44` with the known issues.
11. The admin listener is reached through `kubectl port-forward` only.
    No token in `v0.1.0`.

Proposed during the design discussion, with no objection from the owner:

1. Standard library only (ADR 01).
2. Named state model, dependency checks, operator switches, 200 and 503
   codes, `/livez`, `/readyz`, `/startupz` and `/healthz` (ADR 02).
3. Admin endpoints on a separate listener that is off by default, POST
   only, no CORS; `restart` and `env` removed, `recover` added (ADR 03).
4. `Run` lifecycle helper with a drain delay (ADR 03).
5. In-flight limiter, trace header propagation and a hand-written
   `/metrics`.
6. The probe state defects are fixed by the redesign, not by patching the
   old flags. Tests are written for the new code.

Settled while writing the design, open to change:

1. Keep `/support/v1/` as the admin path prefix.
2. `DrainDelay` 5 seconds and `ShutdownTimeout` 20 seconds by default.
3. A check with no result yet counts as failing.
4. Probe bodies leave out check error text unless an option turns it on.
5. Overload rejections use 503 with `Retry-After: 1`.
6. The log format is fixed at start-up.
7. The in-flight limiter wraps only the application's handler, never the
   probe, admin or metrics handlers.
8. Admin actions answer 403 to cross-site browser requests, based on
   `Origin` and `Sec-Fetch-Site`.
9. Check forcing uses `POST checks/{name}/fail`, `pass` and `clear`;
   overload uses `POST overload` and `overload/clear`.
10. The crash loop is driven by the `ZPAGES_CRASH_ON_START` environment
    variable, read by `Run`.

Open:

1. Keep or change the name `zpages`, before Phase 3.

## 5. Preferences

1. No attribution lines anywhere: commits, PR descriptions, documents or
   code comments.
2. Keep the focus narrow. Do not widen the scope to the demo without
   asking.

## 6. Learnings

1. The kubelet reads only the status code; 203 and 501 carry no extra
   meaning for it.
2. Application code must never clear an operator switch.
3. JSON bodies go to `w.Write`, never into a format string.
4. A handler that the library does not route itself leaves method checks
   to every consumer, and both consumers got it wrong.
