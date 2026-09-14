# SESSION-2026-08-18 — Instrument auth bootstrap phase for intermittent first-load hang

**Date:** 2026-08-18

---

## 1. Requirement recap

User reported that the web app's first page load sometimes gets stuck forever on the "Đang tải dữ
liệu Vĩnh Long…" (loading Vĩnh Long data…) spinner. This is a recurring, intermittent symptom — the
codebase already carries several prior fixes for related hangs, documented in existing code comments:
a no-op auth lock in `supabaseClient.ts`, pinning a single Supabase client instance across HMR
(Hot Module Replacement) reloads, removing a redundant `getSession()` call from `AuthContext.tsx`, a
20s `Promise.all` timeout in the data-loading path, and per-query timing logs (a `timed()` wrapper)
around the ~23 parallel Supabase queries fired by `loadAppData()`. Despite all of that, the hang was
reported to still happen sometimes.

## 2. How it was implemented + docs used

No new library or pattern — pure diagnostic instrumentation using `console.log` plus
`performance.now()` for relative timestamps. The approach was to find the blind spot rather than
guess at a new fix: `AuthContext.tsx`'s bootstrap `useEffect` had **zero logging on its
success/normal path** — it only logged on the 10s timeout (`AUTH_BOOTSTRAP_TIMEOUT_MS`) or on error.
Since `App.tsx`'s `Shell` component gates `loadAppData()` behind `authLoading`, any hang happening
*before* `loadAppData()` is even called (i.e., during Supabase auth bootstrap) was completely
invisible in the console — none of the prior fixes (the 20s timeout, the `timed()` wrapper) could
have caught it, because those all instrument the data-loading phase, not the auth phase that gates it.

Options considered:
- **Add a hard timeout around auth bootstrap too**, mirroring the existing 20s `Promise.all` timeout
  — rejected for this session because it would mask the symptom (force progress after N seconds)
  without revealing *why* it hangs, and a wrong guess at the timeout mechanism could re-introduce a
  race similar to the ones already fixed and documented in `supabaseClient.ts`.
- **Add console logging only** — chosen. It changes no runtime behavior, carries no risk of
  introducing a new race, and gives direct visibility into `onAuthStateChange` firing (or not),
  which branch it takes, and how long `loadProfile()` takes, on the very next reproduction.

No existing logging pattern was reused verbatim (the file had none on the success path); the log
message format (`[auth] ...` / `[shell] ...` prefixes, `t+Nms` via `performance.now()`) was newly
introduced but kept consistent with the style of the existing timeout/error logs already in the file.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/context/AuthContext.tsx` | Auth bootstrap effect that subscribes to `supabase.auth.onAuthStateChange` and sets `loading` | Logged only on the 10s timeout or on error — the entire success path (guest, known user, new user → `loadProfile()`) was silent | Logs at effect start, on every `onAuthStateChange` fire (event, `hasSession`, `isFirstFire`, timestamp), and on each of the three success branches (guest, same known user, new user before/after `loadProfile()`) |
| `web/src/App.tsx` (`Shell` component) | Effect that waits for `authLoading` to clear, then calls `loadAppData()` | No logging at all in this effect | Logs when the effect (re)runs with the current `authLoading` value, right before calling `loadAppData()`, and in both `.then()`/`.catch()` showing whether `cancelled` was already set |

## 4. Code changes in detail

### 1. Log auth bootstrap effect start and every `onAuthStateChange` fire — `web/src/context/AuthContext.tsx`

**Before:**
```ts
  useEffect(() => {
    let cancelled = false;
    // Đánh dấu đã nhận lần fire đầu tiên của onAuthStateChange (nó luôn tự
    // bắn 1 lần với session hiện tại ngay lúc subscribe). Trước đây bootstrap
    // gọi thêm `supabase.auth.getSession()` thủ công song song với việc
    ...
    const { data: sub } = supabase.auth.onAuthStateChange(async (_event, newSession) => {
      if (cancelled) return;
      const isFirstFire = !initialized;
      initialized = true;
```

**After:**
```ts
  useEffect(() => {
    let cancelled = false;
    console.log(`[auth] AuthProvider effect bắt đầu, subscribe onAuthStateChange lúc t+${performance.now().toFixed(0)}ms`);
    // Đánh dấu đã nhận lần fire đầu tiên của onAuthStateChange (nó luôn tự
    ...
    const { data: sub } = supabase.auth.onAuthStateChange(async (_event, newSession) => {
      console.log(
        `[auth] onAuthStateChange fired: event=${_event} hasSession=${!!newSession} isFirstFire=${!initialized} lúc t+${performance.now().toFixed(0)}ms`,
      );
      if (cancelled) return;
      const isFirstFire = !initialized;
      initialized = true;
```

**What changed:** one `console.log` added at the top of the effect body (before the timeout/cleanup
setup); one `console.log` added as the first line inside the `onAuthStateChange` callback, before the
`cancelled` guard — so it fires and is visible even in the (currently impossible-to-observe) case
where the callback is called but immediately bails out.
**Why:** without this, there was no way to tell, from a stuck session, whether Supabase's client ever
called back into the app at all. This is the single biggest blind spot: `onAuthStateChange` is
documented by Supabase to always fire once immediately with the current session on subscribe — if
that never happens, the whole app is stuck before `AuthProvider` can ever call `setLoading(false)`.
**How it behaves now:** on every load, the console will show the effect starting and the exact
timestamp of the first (and any subsequent) `onAuthStateChange` fire, including the event name and
whether a session was present.

### 2. Log the three success branches inside `onAuthStateChange` — `web/src/context/AuthContext.tsx`

**Before:**
```ts
      if (!newSession) {
        knownUserIdRef.current = null;
        setProfile(null);
        if (isFirstFire) setLoading(false);
        return;
      }
      ...
      if (newSession.user.id === knownUserIdRef.current) {
        if (isFirstFire) setLoading(false);
        return;
      }

      knownUserIdRef.current = newSession.user.id;
      if (!isFirstFire) setLoading(true);
      await loadProfile(newSession.user.id);
      setLoading(false);
    });
```

**After:**
```ts
      if (!newSession) {
        knownUserIdRef.current = null;
        setProfile(null);
        if (isFirstFire) {
          console.log(`[auth] không có session -> guest, setLoading(false)`);
          setLoading(false);
        }
        return;
      }
      ...
      if (newSession.user.id === knownUserIdRef.current) {
        if (isFirstFire) {
          console.log(`[auth] session của user đã biết (không đổi) -> setLoading(false)`);
          setLoading(false);
        }
        return;
      }

      knownUserIdRef.current = newSession.user.id;
      if (!isFirstFire) setLoading(true);
      console.log(`[auth] user mới (${newSession.user.id}), gọi loadProfile()…`);
      await loadProfile(newSession.user.id);
      console.log(`[auth] loadProfile() xong lúc t+${performance.now().toFixed(0)}ms -> setLoading(false)`);
      setLoading(false);
    });
```

**What changed:** added one log line to each of the three code paths that end in `setLoading(false)`
(guest, unchanged known user, new user), plus a log immediately before and after the `await
loadProfile(...)` call specifically — this is the only `await` in the callback and therefore the only
place execution can genuinely suspend for an unbounded time.
**Why:** if the hang happens inside `loadProfile()` (e.g. a Supabase query that never resolves,
similar in shape to the already-fixed `Promise.all` hang in the data-loading phase), the "before" log
will appear but the "after" log never will — pinpointing the hang to that specific call versus, say,
`onAuthStateChange` never firing at all.
**How it behaves now:** each of the three branches is now distinguishable in the console, and a
`loadProfile()` hang is now observable as a "before" log with no matching "after" log, rather than
silence indistinguishable from the whole effect never running.

### 3. Log the `Shell` data-loading effect and `loadAppData()` settlement — `web/src/App.tsx`

**Before:**
```ts
    if (authLoading) return;

    let cancelled = false;
    loadAppData()
      .then((d) => { if (!cancelled) setData(d); })
      .catch((e) => { if (!cancelled) setError(String(e)); });
    return () => { cancelled = true; };
```

**After:**
```ts
    console.log(`[shell] effect chạy, authLoading=${authLoading} lúc t+${performance.now().toFixed(0)}ms`);
    if (authLoading) return;

    let cancelled = false;
    console.log(`[shell] authLoading=false -> gọi loadAppData()`);
    loadAppData()
      .then((d) => {
        console.log(`[shell] loadAppData() resolve, cancelled=${cancelled}`);
        if (!cancelled) setData(d);
      })
      .catch((e) => {
        console.log(`[shell] loadAppData() reject, cancelled=${cancelled}`, e);
        if (!cancelled) setError(String(e));
      });
    return () => { cancelled = true; };
```

**What changed:** a log at the very top of the effect (fires on every re-run, including the ones that
bail out early on `authLoading === true`); a log right before calling `loadAppData()`; logs inside
both `.then()` and `.catch()` reporting the `cancelled` flag at settlement time.
**Why:** ties the auth phase to the data phase in one timeline. Without this, even if the
`AuthContext` logs showed `setLoading(false)` was called, there was no confirmation that the `Shell`
effect actually re-ran afterward and called `loadAppData()` — a stale-closure or missed-dependency
bug in `authLoading` propagation would look identical to an auth hang from the console's previous
(silent) state.
**How it behaves now:** the console shows the exact moment `authLoading` flips to `false`, the moment
`loadAppData()` is invoked, and whether it ever settles — and if the effect's cleanup ran first
(`cancelled=true`), that's now visible instead of a silent no-op `setData`/`setError` skip.

## 5. How to find this again

- `grep -rn "\[auth\]" web/src/context/AuthContext.tsx`
- `grep -rn "\[shell\]" web/src/App.tsx`
- `grep -n "AUTH_BOOTSTRAP_TIMEOUT_MS" web/src/context/AuthContext.tsx` — the pre-existing 10s timeout
  this instrumentation sits alongside.
- `grep -rn "timed(" web/src/` — the pre-existing per-query timing wrapper for `loadAppData()`'s ~23
  parallel Supabase queries (the data-loading phase, downstream of what this session instrumented).
- Symptom string to search chat/issue history: "Đang tải dữ liệu Vĩnh Long".

## 6. Concepts introduced

### `onAuthStateChange`'s guaranteed initial fire
- **Plain definition:** Supabase's JS client, when you subscribe via
  `supabase.auth.onAuthStateChange(callback)`, always invokes `callback` once immediately with the
  current session state (even if there is no session), in addition to firing again on real auth
  events later.
- **Why it shows up here:** the app's whole bootstrap logic depends on this guaranteed first fire to
  ever call `setLoading(false)`. If it doesn't fire (for whatever reason), the app hangs forever with
  no visible error — this session's logging is aimed squarely at confirming or disproving that this
  fire happens on every load.

### `performance.now()` for relative timing in logs
- **Plain definition:** returns a high-resolution timestamp (milliseconds since the page started
  navigating), monotonic and unaffected by system clock changes — unlike `Date.now()`.
- **Why it shows up here:** used to compute elapsed time between the effect starting and each log
  point, so that a session reproduction gives a timeline (e.g. "onAuthStateChange fired 3ms after
  subscribe" vs. "never fired within N seconds") instead of just an ordering of events.

No other new concepts — this session reused patterns (the `timed()` wrapper style, the existing
Vietnamese-language inline comments explaining prior race-condition fixes) already established in the
codebase.

## 7. Where it got stuck

This session did not itself get stuck implementing the change — adding `console.log` statements is
low-risk and `npx tsc --noEmit` passed clean immediately. The "stuck" part is the **underlying
production bug**, which remains unsolved after this session:

- **Symptom:** intermittent hang on first page load, stuck on "Đang tải dữ liệu Vĩnh Long…"
  indefinitely (per user report — no fixed reproduction steps known).
- **Already ruled out / already fixed in prior sessions** (per existing code comments, not
  re-verified in this session): a no-op auth lock issue in `supabaseClient.ts`; multiple `GoTrueClient`
  instances being created across HMR reloads (fixed by pinning a single client instance); a redundant
  `getSession()` call racing with `onAuthStateChange` in `AuthContext.tsx` (removed); the 23 parallel
  Supabase queries inside `loadAppData()` hanging without bound (fixed with a 20s `Promise.all`
  timeout and per-query `timed()` logging).
- **Why those fixes were insufficient:** they all target either the Supabase client's internal locking
  or the data-loading phase (`loadAppData()`), which only runs *after* `authLoading` becomes `false`.
  None of them add any visibility into the auth-bootstrap phase itself. If the hang originates there
  — e.g. `onAuthStateChange` never firing, or `loadProfile()` hanging on its own Supabase query — none
  of the prior fixes could have caught or logged it, which is consistent with (but does not prove) why
  the hang reportedly still recurs.
- **Root cause: not identified this session.** This is explicitly diagnostic tooling, not a fix. No
  hang was reproduced or observed during this session — the change adds the missing instrumentation so
  that the *next* occurrence can be diagnosed from the browser console instead of requiring another
  guess-and-fix cycle. The next reproduction with DevTools open will show one of: (a) `onAuthStateChange`
  never fires at all, (b) it fires but `loadProfile()` never resolves, or (c) both auth logs complete
  normally but `[shell]` never logs `loadAppData()` being called — each pointing to a different,
  previously indistinguishable failure mode.

## 8. Verify

```bash
cd web && npx tsc --noEmit
```
Ran during this session — completed with no output, confirming no type errors were introduced by the
added logging.

Runtime verification (not yet done — requires an actual reproduction of the hang):
1. Open the app in a browser with DevTools console open.
2. On a normal (non-hung) load, expect to see, in order: `[auth] AuthProvider effect bắt đầu...`,
   `[auth] onAuthStateChange fired: ...`, one of the three `[auth]` branch logs ending in
   `setLoading(false)`, then `[shell] effect chạy, authLoading=false ...`, `[shell] authLoading=false
   -> gọi loadAppData()`, and finally `[shell] loadAppData() resolve, cancelled=false`.
3. On the next occurrence of the stuck-spinner symptom, capture the console output up to the point it
   stops — the last log line before silence identifies which of the three failure modes in section 7
   occurred.

## 9. Gotchas

- These are `console.log` statements left in production code, not behind a debug flag — they will
  appear in every user's browser console on every load. This is intentional for now (diagnostic
  phase), but if the hang is eventually root-caused and fixed, a follow-up session should remove or
  gate these logs behind an environment check to avoid leaving debug noise in production long-term.
- The `[shell]` effect log fires on every re-run of the effect, including the early-return branch when
  `authLoading` is still `true` — this is expected and by design (needed to confirm the effect *did*
  re-run when `authLoading` flips), not a sign of an extra unwanted effect execution.
- `isFirstFire` logging only reflects the state at the moment the callback starts executing, before
  `initialized = true` is set later in the same callback — do not use these logs to infer the value of
  `initialized` at any point other than the moment the log line printed.
- This instrumentation only covers the auth-bootstrap phase and the hand-off into `loadAppData()`. It
  does **not** duplicate or replace the existing `timed()` per-query logging inside `loadAppData()`
  itself — if a future hang shows all `[auth]`/`[shell]` logs completing normally but the app still
  spins, the existing `timed()` logs (not this session's changes) are the next place to look.
