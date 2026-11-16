# Worked example: T2 login rate limiter

For a compact T1 bug-fix example, see [`SKILL.md`](../SKILL.md). Below is a full pre-gate T2 walkthrough.

**Task:** "Add failed-login rate limiting to `/login` per `docs/auth-policy.md` (5/15m per email+IP) behind our reverse proxy. Make no mistake."

## Contract

- **Goal:** enforce the 5-failures/15m `(email, client_ip)` sliding-window policy on `/login` without global account lockouts.
- **Oracle:** `docs/auth-policy.md:14-29`. Using Redis `TIME` as the single clock across replicas, it specifies atomic failed-attempt slot reservation in `(now_ms - 900_000, now_ms]` up to 5 entries, 429 with `Retry-After` on the 6th without inserting or extending TTL, and a `last_clear_ms` marker on login success so concurrent in-flight failures started at or before a successful login cannot re-populate the cleared bucket.
- **Done when:**
  - 5 failed attempts in `(now_ms - 900_000, now_ms]` (by Redis `TIME`) are allowed and the 6th returns 429 with `Retry-After`;
  - blocked attempts neither insert nor refresh TTL;
  - a successful login clears the `(email, client_ip)` key and stamps `last_clear_ms` so an overlapping failure started earlier is ignored while subsequent failures count normally;
  - proxied clients with distinct forwarded IPs get separate buckets, while untrusted peers cannot spoof `X-Forwarded-For`;
  - existing auth tests and new limiter tests pass.
- **Must not:** globally lock an account, collapse all proxied users into one bucket, or edit unrelated endpoints.
- **Assumptions:**
  - production sits behind L7 proxies matching `TRUSTED_PROXY_CIDRS`;
  - the `REDIS_URL` reference in `deploy/prod.yaml:48` resolves to one shared Redis keyspace across replicas without per-pod/env overrides.
  - If either is wrong, client IPs collapse or per-replica counters diverge.
- **Stakes: T2** — authentication/rate-limit security posture; local change only (pre-gate).

## Mistake map

- Trusting `X-Forwarded-For` from direct peers lets attackers rotate headers to bypass limits.
- Keying on `remote_addr` behind a trusted proxy collapses all users into one bucket.
- Non-atomic prune/count/insert or success-clear vs. concurrent-failure races under bursts; clock skew across app pods if caller wall-clock is used instead of Redis `TIME`.
- Same-millisecond timestamps overwrite each other in a sorted set.
- Refreshing TTL on blocked/prune-only calls extends lockouts indefinitely, whereas setting TTL only on key creation drops expiry for later entries.
- Production replicas might not resolve to the same Redis endpoint/keyspace.
- Production LB hop count may differ from `TRUSTED_PROXY_CIDRS`.
- `/reset-password` may remain an unthrottled adjacent surface.

## Prediction

- Behind a trusted proxy, two clients with distinct forwarded IPs get separate buckets.
- A direct untrusted peer rotating `X-Forwarded-For` lands in its `remote_addr` bucket and gets 429 on the 6th attempt.
- Under the spec's serialized failed-attempt slot reservation, two concurrent failures starting at 4 active entries yield one 401 and one 429, with the count remaining 5.

Concrete expectations must follow from the formal algorithm and contract semantics — window bounds, operation ordering, and reservation point — not from an implementation label like "sliding window".

## Verify and correct

1. Record a clean baseline: `pytest tests/test_auth.py -q` → 18 passed on `feat/login-rate-limit` in repo root.
2. Write `test_trusted_proxy_distinct_clients` and `test_forged_header` (one account, wrong password, rotating `X-Forwarded-For` each attempt); test expectations were derived directly from the policy before inspecting the implementation.
3. The first draft reads `X-Forwarded-For` unconditionally and returns six 401s on `test_forged_header`, exposing the bypass.
4. Fix: trust `X-Forwarded-For` only when `remote_addr` is in `TRUSTED_PROXY_CIDRS`, walk right-to-left to the first untrusted hop, and reject empty or malformed chains with 400 rather than sharing the proxy IP bucket.
5. Re-run the auth and limiter suites.

## Fresh eyes

An isolated-context subagent challenges the contract against `docs/auth-policy.md:14-29`, runs the suites, and finds two issues:

1. The store passes pod-local wall-clock time instead of Redis `TIME`, uses a non-atomic read/modify/write (racing concurrent failures against login-success `DEL`), keys sorted-set members by timestamp alone (overwriting same-millisecond events), and sets TTL only on key creation in a separate call.
2. `/reset-password` has no limiter.

Corrections:

- Replace the store logic with atomic Lua scripts using `redis.call('TIME')` (`now_ms`) as the single clock across pods:
  - pre-auth check reads `req_ms` from Redis `TIME` and rejects immediately with 429 when active count is already 5;
  - on auth failure, ignore stale in-flight attempts where `req_ms <= last_clear_ms`, prune entries `<= now_ms - 900_000`, check cardinality `< 5`, insert a unique `(now_ms, nonce)` member only when allowed, and refresh `PEXPIRE` to `900_000ms` **only on insertion** (never on blocked 429 or prune-only calls, so retries cannot extend lockout and empty keys vanish immediately);
  - on auth success, atomically delete the failure set and record `last_clear_ms = now_ms` with `PEXPIRE 900_000ms`.
- Prove the same-timestamp, boundary-expiry, concurrency-barrier, concurrent success-vs-failure ordering, and insert-only TTL checks red against the old store, then green (test expectations derived directly from `docs/auth-policy.md:14-29`).
- **Resubmit the updated diff** to a second fresh-eyes pass. Pass 2 runs the full suite and breaks the proxy guard in a disposable `0700` `/tmp` copy (cleaned up afterward) to confirm `test_forged_header` fails red.
- Keep `/reset-password` out of the diff per contract and surface it under `Residual risk`.

## Ledger

Test expectations for rows #1–#3 were derived directly from the policy before inspecting the implementation:

| # | Risk | Severity | Check | Evidence (command/source, oracle, provenance) | Status |
| - | ---- | -------- | ----- | --------------------------------------------- | ------ |
| 1 | Forged `X-Forwarded-For` bypasses client-IP key | high | Untrusted peer rotates `X-Forwarded-For` 6 times; 6th returns 429 with `Retry-After` | `pytest tests/test_rate_limit.py::test_forged_header -q` → 1 passed; oracle: `docs/auth-policy.md:14-29`; `feat/login-rate-limit` @ `abc1234`, repo root, local, exit 0 | passed |
| 2 | Trusted proxy collapses distinct clients into one bucket | high | Two requests via trusted proxy with distinct client IPs increment separate keys | `pytest tests/test_rate_limit.py::test_trusted_proxy_distinct_clients -q` → 1 passed; oracle: `docs/auth-policy.md:14-29`; `feat/login-rate-limit` @ `abc1234`, repo root, local, exit 0 | passed |
| 3 | Sliding-window race, clock skew, same-ms collision, success-vs-failure race, or broken TTL | high | Redis `TIME` clock, boundary prune, unique `(now_ms, nonce)`, barrier at count 4 (one 401 + one 429, count=5), `last_clear_ms` ordering on concurrent login success, and insert-only `PEXPIRE` | `pytest tests/test_rate_limit.py -k "window or concurrent or clear or ttl" -q` → 5 passed; oracle: `docs/auth-policy.md:14-29`; `feat/login-rate-limit` @ `abc1234`, repo root, local Redis, exit 0 | passed |
| 4 | Prod replicas do not share one Redis keyspace | high | Check repo deploy manifests for shared `REDIS_URL`; verify live prod pods resolve identical Redis endpoint/DB | `deploy/prod.yaml:48` and `deploy/overlays/prod/kustomization.yaml:12` reference one `REDIS_URL` with no repo override; live prod observation blocked (`access`: no prod cluster access; owner: platform on-call) | inferred |
| 5 | Production proxy hop count shifts client IP | high | Compare live LB `X-Forwarded-For` chain against `TRUSTED_PROXY_CIDRS` | Blocked (`access`) — external LB hops not in repo and no prod read-only access; owner: platform on-call | unverified |

## Report (pre-gate)

```text
Tier: T2 — authentication/rate-limit security posture; T2 complete (pre-gate), no gate required
Outcome: Sliding-window `/login` rate limiter implemented and verified locally; not deployed
Changed: app/rate_limit.py, app/auth.py, tests/test_rate_limit.py, tests/test_auth.py
Interpretation: none
Verified: `feat/login-rate-limit` @ `abc1234`, repo root, local + disposable Redis (oracle: `docs/auth-policy.md:14-29`, test expectations derived directly from policy before inspecting implementation): `pytest tests/test_auth.py tests/test_rate_limit.py -q` → 30 passed, exit 0; `deploy/overlays/prod/kustomization.yaml:12` adds no per-replica Redis override, and `deploy/prod.yaml:48` references a shared `REDIS_URL`; actual production endpoint equality remains unverified.
Inferred: prod replicas share one Redis keyspace under repo deployment config (open high-severity risk #4 — `inferred` does not close it) — inferred from `deploy/prod.yaml:48` and `deploy/overlays/prod/kustomization.yaml:12`; reverses if out-of-repo cluster secrets or runtime env overrides point pods at distinct Redis endpoints or DB indexes; blocked (`access`: no prod cluster access); next check: verify resolved Redis endpoint/DB across live prod pods (owner: platform on-call)
Unverified: actual live production `REDIS_URL` endpoint equality (absence of out-of-repo secret/env overrides) and live LB `X-Forwarded-For` hop count vs. `TRUSTED_PROXY_CIDRS` — both blocked (`access`: prod read-only access unavailable); next checks: verify resolved Redis endpoint/DB across live prod pods and compare one prod request's `X-Forwarded-For` hop count against `TRUSTED_PROXY_CIDRS` (owner: platform on-call)
Residual risk: high-severity risks #4 (live prod Redis endpoint/keyspace equality) and #5 (live LB hop count) remain open until platform on-call runs the blocked pre-deploy checks; `/reset-password` has no rate limiter (left untouched per contract; recommend follow-up); distributed multi-IP sprays against one account or many accounts from one NAT IP are outside the `(email, ip)` policy in `docs/auth-policy.md`
Post-action state: n/a (pre-gate)
Verifier: fresh-context subagent (`prompt-scoped`; local pytest + disposable Redis); ran `pytest tests/test_auth.py tests/test_rate_limit.py -q` → 30 passed (`feat/login-rate-limit` @ `abc1234`, repo root, exit 0), inverted proxy trust guard in a cleaned-up `0700` `/tmp` copy (`test_forged_header` → 1 failed, exit 1), static trace of `app/rate_limit.py:42-96` inferred
Rollback/compensation: n/a (pre-gate local work only)
Stop reason: all feasible local high-severity checks passed; two live prod checks (#4 endpoint equality, #5 LB hop count) blocked by named prod access
Blocked: access — platform on-call verification of live prod `REDIS_URL` endpoint equality and `TRUSTED_PROXY_CIDRS` hop count before any deploy
```

## If deployment is later requested

Stop before deploy and emit a T2 gate packet (see [`references/t2.md`](t2.md)) with:

- `Approval: pending`;
- `Action: deploy abc1234 to prod`;
- baseline 5xx and login-success rates;
- exercised vs. untested rollback steps (feature-flag disable tested in ephemeral staging; prod Redis keyspace flush untested);
- numeric abort thresholds over a 60-minute canary window.

The canary is execution *after* approval, not the approval.
