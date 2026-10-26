# ADR: state model and probes

## 1. Status

Proposed on 2026-10-26. Waiting for the owner's review.

## 2. Context

1. Health and readiness each use three unexported flags. Some states
   cannot be reached, some cannot be undone, and application setters
   clear operator overrides.
2. The flags set the HTTP code while separate strings set the body, and
   the two can disagree.
3. The kubelet only distinguishes success, codes 200 to 399, from
   failure. The custom 203 and 501 codes carry no extra meaning for it.
4. The Kubernetes API server moved from `/healthz` to `/livez` and
   `/readyz`. Kubernetes and Spring Boot both advise against liveness
   checks that depend on external systems.
5. The kubelet's default probe timeout is 1 second, too short for checks
   that call other services inline.

## 3. Decision

1. One state value with a lifecycle phase, two operator switches, check
   results and an optional heartbeat, guarded by one mutex. One function
   derives the reported state from it, by the precedence in
   `systemPatterns.md` section 2.2.
2. Only the admin endpoints can change the operator switches.
3. Dependency checks are optional or critical. An optional failure makes
   the service degraded and still ready; a critical failure makes it not
   ready. No check affects liveness.
4. Checks run in the background and cache their result. Probes never run
   checks inline.
5. Endpoints: `/livez`, `/readyz`, `/startupz`, and `/healthz` as an alias
   of `/livez`. Codes: 200 and 503 only.
6. The JSON body uses IETF health-check draft field names where they fit,
   with `pass`, `warn` and `fail`.
7. An operator can force a check to pass or fail. The forced result is
   stored apart from the real one and wins until the operator clears it.

## 4. Consequences

1. Consumers change their probe paths, or keep `/healthz` for liveness.
2. The body format changes. Tools that parse the old body break.
3. Choosing optional or critical becomes a design decision for each
   dependency. `productContext.md` section 3 gives guidance.
4. The patching tasks for the old flags are dropped; tests target the
   new state model.

## 5. Alternatives considered

1. Patch the existing flags: rejected. The model would stay hard to
   reason about, and the setters would still mix operator and
   application intent.
2. Keep 203 and 501: rejected, because no platform component reads them
   differently from 200 and 503.
3. Run checks inline on each probe: rejected, because a slow dependency
   would turn into a probe timeout.
4. Use the draft's `application/health+json` media type: not chosen for
   now, because the draft has expired and the type is not registered.
   The body keeps `application/json`.
