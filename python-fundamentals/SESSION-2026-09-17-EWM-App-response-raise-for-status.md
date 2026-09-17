# Concept note: `response.raise_for_status()` in `login_cookie()`

Not a bug session — a concept question during Phase 4 (rain/water-level crawler migration). The user
asked what `response.raise_for_status()` actually does in `login_cookie()`, around line 64 of
`app/engine_service/crawler/rain_crawler.py`. No code was changed for this note; it explains an
already-existing line.

## 1. Requirement recap

Explain `response.raise_for_status()` — what it checks, what it raises, and why it is used right
after the `requests.post()` call in `login_cookie()`.

## 2. How it was implemented + docs used

No implementation was needed; this was a walkthrough of existing code using the `requests` library's
documented behavior (`Response.raise_for_status()`). No alternative library or pattern was evaluated
— the explanation covers why this line matters given the code that already surrounds it, and contrasts
it with the other common styles (`response.ok`, no check at all) purely for context.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `app/engine_service/crawler/rain_crawler.py` | `login_cookie()` — logs into the vrain.vn portal and returns a `sid=...` cookie string | (unchanged — line already present) | (unchanged — explained, not modified) |

## 4. Code changes in detail

No diff exists for this note — it is a pure explanation of code that was already implemented in an
earlier part of the Phase 4 session. Shown here as-is, since there is no "before" state to contrast.

### 1. `raise_for_status()` guards the login HTTP call — `app/engine_service/crawler/rain_crawler.py`

**Before:** N/A — nothing changed; this is existing code being explained.

**After (unchanged, shown for reference):**
```python
def login_cookie() -> str | None:
    """Log in and return a new `sid=...` cookie; None if no credentials configured."""
    username = os.environ.get("VRAIN_USERNAME")
    password = os.environ.get("VRAIN_PASSWORD")
    if not username or not password:
        logger.warning("Missing VRAIN_USERNAME/VRAIN_PASSWORD env vars; cannot auto-login")
        return None
    headers = {
        **REQUEST_HEADERS,
        "content-type": "application/json",
        "origin": "https://kttv.vrain.vn",
    }
    response = requests.post(
        "https://kttv.vrain.vn/api/kttv/public/v1/login",
        json={"username": username, "password": password},
        headers=headers,
        timeout=30,
    )
    response.raise_for_status()
    sid = response.cookies.get("sid")
    if not sid:
        raise RuntimeError("Login succeeded but no sid cookie was returned")
    return f"sid={sid}"
```

**What changed:** Nothing — no diff. This is a read of the existing line
`response.raise_for_status()` between the `requests.post()` call and the `sid = response.cookies.get(...)`
line.

**Why this line exists:** `requests.post()` does **not** raise an exception just because the server
returned an HTTP error status (4xx/5xx). It only raises for connection-level failures — DNS
resolution failure, connection refused, timeout (`ConnectionError`, `Timeout`) — which happen
**before** a `Response` object even exists. If the vrain.vn login endpoint returns, say, `401
Unauthorized` (wrong credentials) or `500 Internal Server Error`, `requests.post()` returns a normal
`Response` object with `status_code=401` and no exception. Without a check, the next line
(`response.cookies.get("sid")`) would just silently find no `sid` cookie and fall through to the
generic `raise RuntimeError("Login succeeded but no sid cookie was returned")` — which is misleading,
because login did **not** succeed.

`response.raise_for_status()` closes that gap: it inspects `response.status_code`, and if it is in
the 4xx or 5xx range, raises `requests.HTTPError` with the status code and reason in the message
(e.g. `401 Client Error: Unauthorized for url: ...`). On 2xx it does nothing and execution continues
normally.

**How it behaves now (i.e. what this line causes at runtime):**
- On a successful login (HTTP 200), `raise_for_status()` is a no-op; execution proceeds to read the
  `sid` cookie.
- On a failed login (e.g. HTTP 401 from bad `VRAIN_USERNAME`/`VRAIN_PASSWORD`), `raise_for_status()`
  raises `requests.HTTPError` immediately, with the actual status code in the message.
- That exception propagates up to the caller, `crawl_day()` (around line 162-190), which wraps calls
  to `login_cookie()` in a `try/except` and logs a message such as `"re-login also failed"` together
  with the caught exception. Because `raise_for_status()` raised a specific `HTTPError` (carrying the
  401), the log line reflects the real cause — a credentials/auth failure — instead of the vaguer
  `RuntimeError("Login succeeded but no sid cookie was returned")` that would otherwise fire and
  mislabel an auth failure as a "missing cookie" issue.

## 5. How to find this again

- Grep: `raise_for_status`, `login_cookie`, `HTTPError`
- File: `app/engine_service/crawler/rain_crawler.py`, function `login_cookie()` (~line 46-68)
- Caller: `crawl_day()` in the same file (~line 162-190), the `try/except` around re-login

## 6. Concepts introduced

- **`requests.post()`/`.get()` never auto-raises on HTTP error status codes.** A 404, 401, or 500
  response is returned as a normal `Response` object with that `status_code` set — no exception. Only
  connection-level failures (DNS lookup failure, connection refused, timeout) raise before you get a
  `Response` at all (`requests.exceptions.ConnectionError`, `requests.exceptions.Timeout`).
- **`response.raise_for_status()`** — checks `response.status_code`; raises `requests.HTTPError` if it
  is 4xx or 5xx, does nothing on 2xx (and typically 3xx, since `requests` follows redirects by
  default). This is why calling it turns a "silent bad response" into an explicit, catchable
  exception with the real status code in the message.
- **`response.ok`** — an alternative style: a boolean property (`True` for 2xx, `False` for 4xx/5xx)
  that you check manually with `if not response.ok: ...`. Same underlying status-code check as
  `raise_for_status()`, but you write your own branching/error instead of getting `HTTPError`
  automatically. `login_cookie()` uses `raise_for_status()` rather than this style because it wants
  the exception to propagate straight into the existing `try/except` in `crawl_day()` without extra
  branching code.

## 7. Where it got stuck

Genuinely nothing — this was a straight explanation of an existing, already-working line during
Phase 4 review, not a debugging session. No error was reproduced, no false lead was chased.

## 8. Verify

No verification needed — no code changed. To see this behavior live in the codebase, the reasoning
can be confirmed by reading `crawl_day()`'s `try/except` around its `login_cookie()` call and
observing that an `HTTPError` raised from a bad login attempt is caught there and logged, rather than
crashing the crawler job.

## 9. Gotchas

- `raise_for_status()` does **not** catch connection-level failures — a request that never reaches
  the server (DNS failure, connection refused, timeout) raises before a `Response` object exists, so
  `raise_for_status()` is never called at all in that case. Callers relying on it as their only error
  handling will miss those cases; `crawl_day()`'s `try/except` catches them anyway since it wraps the
  whole call, but a narrower `except requests.HTTPError` around just this line would not.
- 3xx redirects are followed automatically by `requests` by default (`allow_redirects=True` for
  `.get()`/`.post()`), so `raise_for_status()` normally never sees a 3xx status to raise on unless
  redirects are disabled.
- If this endpoint's auth contract changes (e.g. it starts returning 200 with an error payload
  instead of a proper 401), `raise_for_status()` would no longer catch it — the check is purely on
  the HTTP status code, not the response body.
