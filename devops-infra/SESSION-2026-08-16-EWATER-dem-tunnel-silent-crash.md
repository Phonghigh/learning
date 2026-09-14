# SESSION-2026-08-16 — DEM tunnel silent crash after reboot, and extracting the service out of the repo

**Date:** 2026-08-16 · **Project:** EWATER · **Type:** bug fix + refactor (service extraction)

---

## 1. Requirement recap

User reported that after restarting Windows, the DEM elevation layer no longer loaded — the tile
URL `https://urgent-choose-inch-gets.trycloudflare.com/...` was failing. Their words: *"có vẻ như
script nó vẫn chưa hoạt động, tôi vừa restart lại máy, check xem nó đang bị gì sao nó lại không
work?"*

Follow-up requests during the same session:
1. Re-run the Vercel production deploy so production picks up the new tunnel URL.
2. Add a retry/wait for `docker` readiness.
3. Delete the unrelated `Cloudflared` Windows service to avoid confusion.
4. Wrap the whole script in try/catch so it can never fail silently again.
5. Then: move the whole thing out of the repo into `E:\Monitoring\FRIMS` as a standalone service.

## 2. How it was implemented + docs used

**Diagnosis first, no code changes until the cause was understood.** Evidence gathered in order:
`Get-ScheduledTask` / `Get-ScheduledTaskInfo` (task state and last result), directory listing of
`scripts/logs/` (which log files exist and when), `Get-CimInstance Win32_Process` (what
`cloudflared` process is actually running and its command line), `Get-CimInstance Win32_OperatingSystem`
(boot time), and `curl` against both the local TiTiler and the reported dead tunnel URL.

Two fixes to `start-dem-tunnel.ps1`:
- A **wait loop for the `docker` command to appear in PATH** at the top of `Wait-DockerReady()`,
  deliberately separate from the existing "is the engine up" poll.
- A **try/catch wrapping the entire 8-step body**, logging both the exception message and
  `$_.InvocationInfo.PositionMessage`.

Also considered and rejected: adding a fixed `Start-Sleep` delay at the top of the script, or
setting Task Scheduler's built-in "Delay task for 30 seconds". Both would trade a guaranteed delay
on every boot for an unreliable guess, and neither produces a log line when the assumption is wrong.
The wait-loop approach costs nothing when Docker is already ready and leaves evidence when it isn't.

Service extraction reused the existing script structure verbatim; the only structural change was
splitting the single `$RepoRoot` variable into `$EwaterRepoRoot` (where `services\tile-server` and
`web\` live) and `$ServiceRoot` (where the script and its logs live).

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `scripts/start-dem-tunnel.ps1` | orchestrates tile server + tunnel + env update | called `docker info` directly; any unforeseen error killed the script with no log | waits for `docker` in PATH (60s cap); whole body in try/catch that always logs. **Now deleted from repo** — lives at `E:\Monitoring\FRIMS\` |
| `scripts/register-dem-tunnel-startup.ps1` | registers the Scheduled Task | pointed at the in-repo `.vbs` | **deleted from repo**; new copy points at `E:\Monitoring\FRIMS\start-dem-tunnel-hidden.vbs` |
| `scripts/start-dem-tunnel-hidden.vbs` | hidden wrapper for Task Scheduler | launched the in-repo `.ps1` | **deleted from repo**; new copy launches `E:\Monitoring\FRIMS\start-dem-tunnel.ps1` |
| `scripts/README.md` | service documentation | in-repo paths | **deleted from repo**; updated copy at `E:\Monitoring\FRIMS\README.md` |
| `.gitignore` | repo ignore rules | had `scripts/logs/` | line removed — the directory no longer exists |
| `scripts/check-i18n.mjs` | i18n key checker (unrelated to tunnel) | — | **kept in repo** deliberately |

Commits: `35668d9` (script fixes), `d46e6d7` (service extraction).

## 4. Code changes in detail

### 1. Wait for the `docker` command to exist before calling it — `scripts/start-dem-tunnel.ps1`

**Before:**
```powershell
function Wait-DockerReady([int]$TimeoutSec = 180) {
    docker info *> $null
    if ($LASTEXITCODE -eq 0) { return $true }
```

**After:**
```powershell
function Wait-DockerReady([int]$TimeoutSec = 180) {
    # Cho lenh `docker` xuat hien trong PATH truoc khi goi. Ngay sau reboot, task
    # AtLogOn co the kich hoat chi vai giay sau khi dang nhap, luc PATH cua tien
    # trinh moi sinh chua kip gom thu muc Docker CLI -`docker info` se nem loi
    # "term khong duoc nhan dien", lam crash toan script truoc ca dong log dau tien.
    $cmdDeadline = (Get-Date).AddSeconds(60)
    while (-not (Get-Command docker -ErrorAction SilentlyContinue)) {
        if ((Get-Date) -ge $cmdDeadline) {
            Write-Log "LOI: lenh 'docker' khong xuat hien trong PATH sau 60s."
            return $false
        }
        Start-Sleep -Seconds 2
    }

    docker info *> $null
    if ($LASTEXITCODE -eq 0) { return $true }
```

**What changed:** 11 lines inserted at the top of the function, before the existing `docker info`
call. Nothing was removed; the original body is untouched below the new loop.

**Why:** `docker info` is a call to an external program. If that program is not findable via PATH,
PowerShell raises a *command-not-found* error — and because this script sets
`$ErrorActionPreference = "Stop"` at the top, that error terminates the entire script instantly,
before any `Write-Log` line executes. That is the mechanism behind "the task said success but no log
file exists" (see section 7, snag 1). `Get-Command ... -ErrorAction SilentlyContinue` is the safe
probe: it returns empty instead of throwing, so it can be polled in a loop.

**How it behaves now:** if `docker` is already on PATH (the normal case), the `while` condition is
false immediately and the loop costs nothing. If it is missing, the script waits up to 60 seconds in
2-second steps, and on timeout writes an explicit `LOI:` line and returns `$false` — a diagnosable
failure instead of a silent death.

### 2. Wrap the whole script body in try/catch — `scripts/start-dem-tunnel.ps1`

**Before:**
```powershell
# 1) Don dep tien trinh cloudflared quick-tunnel cu (neu co) truoc khi chay lai
Get-Process cloudflared -ErrorAction SilentlyContinue |
    ...
Write-Log "Hoan tat. Nho trigger redeploy (vercel --prod hoac push git) de production ap dung URL moi: $TunnelUrl"
```

**After:**
```powershell
try {

# 1) Don dep tien trinh cloudflared quick-tunnel cu (neu co) truoc khi chay lai
Get-Process cloudflared -ErrorAction SilentlyContinue |
    ...
Write-Log "Hoan tat. Nho trigger redeploy (vercel --prod hoac push git) de production ap dung URL moi: $TunnelUrl"

} catch {
    # Bat moi loi chua luong truoc (kem ca loi xay ra truoc dong Write-Log dau tien)
    # de khong bao gio "im lang that bai" nua - luon co it nhat 1 dong LOI trong log.
    $err = $_
    $where = $err.InvocationInfo.PositionMessage -replace "`r?`n", " | "
    Write-Log "LOI CHUA BAT DUOC: $($err.Exception.Message)"
    Write-Log "Vi tri loi: $where"
    exit 1
}
```

**What changed:** a `try {` opened just before step 1 and a `catch` block appended after the final
success log. The eight steps in between are unchanged — only their indentation context moved.

**Why:** change 1 fixes *one specific* cause of a silent crash. This one fixes the **class** of
problem: any unforeseen error anywhere in the script now produces at least one log line. It captures
`$_.InvocationInfo.PositionMessage`, which reports the file and line where the error was raised —
the difference between "something failed" and "line 147 failed".

**How it behaves now:** an unexpected exception writes `LOI CHUA BAT DUOC: <message>` plus
`Vi tri loi: <file:line>` to the run log, then exits 1. Previously the process just vanished.

### 3. Fallback log directory when the primary one can't be created — `scripts/start-dem-tunnel.ps1`

**Before:**
```powershell
New-Item -ItemType Directory -Force -Path $LogDir | Out-Null
```

**After:**
```powershell
# Neu khong tao duoc thu muc log chinh (vd. o dia chua san sang ngay sau reboot),
# fallback sang %TEMP% de van con noi ghi loi thay vi crash hoan toan im lang.
try {
    New-Item -ItemType Directory -Force -Path $LogDir -ErrorAction Stop | Out-Null
} catch {
    $LogDir = Join-Path $env:TEMP "ewater-dem-tunnel-logs"
    New-Item -ItemType Directory -Force -Path $LogDir -ErrorAction SilentlyContinue | Out-Null
}
```

**What changed:** one line became an 8-line try/catch with a `%TEMP%` fallback.

**Why:** the try/catch from change 2 is useless if `Write-Log` itself cannot write — and `$LogDir`
sits on drive `E:`, which may not be mounted yet moments after boot. This closes the gap where the
error handler would fail while handling an error.

**How it behaves now:** if `E:` is unavailable, logs go to `%TEMP%\ewater-dem-tunnel-logs\` instead
of the script dying with nowhere to report it.

### 4. Split one repo root into two roots — `E:\Monitoring\FRIMS\start-dem-tunnel.ps1`

**Before:**
```powershell
$RepoRoot     = "E:\FRIMS_VINH_LONG\EWATER"
$TileServerDir = Join-Path $RepoRoot "services\tile-server"
$WebDir       = Join-Path $RepoRoot "web"
$EnvLocalPath = Join-Path $WebDir ".env.local"
$LogDir       = Join-Path $RepoRoot "scripts\logs"
```

**After:**
```powershell
# Service nay nam ngoai repo EWATER (E:\Monitoring\FRIMS), nhung van thao tac len
# services\tile-server va web\.env.local ben trong repo do -nen tach 2 root rieng.
$EwaterRepoRoot = "E:\FRIMS_VINH_LONG\EWATER"
$ServiceRoot    = "E:\Monitoring\FRIMS"
$TileServerDir = Join-Path $EwaterRepoRoot "services\tile-server"
$WebDir       = Join-Path $EwaterRepoRoot "web"
$EnvLocalPath = Join-Path $WebDir ".env.local"
$LogDir       = Join-Path $ServiceRoot "logs"
```

**What changed:** `$RepoRoot` became two variables. Only `$LogDir` changed which root it derives
from; `$TileServerDir` and `$WebDir` still point into the EWATER repo.

**Why:** the script moved out of the repo, but the things it *operates on* (Docker Compose stack,
`web/.env.local`) did not. One variable could no longer express both "where I live" and "what I
control". Keeping them separate makes the dependency explicit rather than accidental.

**How it behaves now:** logs are written next to the script at `E:\Monitoring\FRIMS\logs\`, while
Docker and env updates still target the repo — the repo no longer accumulates log churn.

### 5. Repoint the Task Scheduler wrapper — `E:\Monitoring\FRIMS\start-dem-tunnel-hidden.vbs`

**Before:**
```vbscript
objShell.Run "powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File ""E:\FRIMS_VINH_LONG\EWATER\scripts\start-dem-tunnel.ps1""", 0, False
```

**After:**
```vbscript
objShell.Run "powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File ""E:\Monitoring\FRIMS\start-dem-tunnel.ps1""", 0, False
```

**What changed:** the path only. `register-dem-tunnel-startup.ps1` got the equivalent one-line change
to `$VbsPath`.

**Why:** after the move, the old path no longer exists. Until the Scheduled Task was re-registered
(which needed an elevated shell — section 7, snag 6), the task pointed at a deleted file and would
have silently done nothing on the next reboot.

**Note on the `Run(..., 0, False)` call:** the third argument `False` means *do not wait* — this is
precisely what made `LastTaskResult: 0` meaningless. It was left as-is intentionally: waiting would
keep a `wscript.exe` process alive for the whole run, and the real fix (logging) now covers the gap.

### 6. Drop the now-meaningless ignore rule — `.gitignore`

**Before:**
```gitignore
shared/data/canal-polygon.geojson
scripts/logs/
.vercel
```

**After:**
```gitignore
shared/data/canal-polygon.geojson
.vercel
```

**What changed:** removed `scripts/logs/`.

**Why:** that directory no longer exists in the repo — the logs live at `E:\Monitoring\FRIMS\logs\`
now. A leftover ignore rule for a nonexistent path is a small lie about the repo's structure.

## 5. How to find this again

- `grep -r "Wait-DockerReady" E:\Monitoring\FRIMS`
- Scheduled Task name: `EWATER-DEM-Tunnel-Startup`
- Env var written by the script: `VITE_DEM_TILE_SERVER_URL` (in `web/.env.local` and Vercel Production)
- Service now lives entirely outside this repo: `E:\Monitoring\FRIMS\`

## 6. Concepts introduced

### PATH readiness vs. service readiness are two different failures
- **Plain definition:** "the `docker` command cannot be found" and "the `docker` command runs but
  the engine isn't up" are distinct conditions that PowerShell surfaces at different levels — the
  first is a PowerShell-level error, the second is just a non-zero `$LASTEXITCODE`.
- **Why it shows up here:** the old code only handled the second. With `$ErrorActionPreference = "Stop"`
  set at the top of the file, the first condition terminates the entire script instantly — before
  any `Write-Log` call runs, which is exactly why no log file existed.

### `$ErrorActionPreference = "Stop"` does not apply to external program exit codes
- **Plain definition:** `Stop` turns PowerShell-level errors into terminating errors, but a native
  `.exe` returning a non-zero exit code is not a PowerShell error — the script continues.
- **Why it shows up here:** it explains the asymmetry above, and why `$LASTEXITCODE` must be checked
  by hand after `docker info` while a missing `docker` needs no check at all (it just kills the script).

### Fire-and-forget wrappers make exit codes meaningless
- **Plain definition:** `objShell.Run(cmd, 0, False)` in VBScript starts a process without waiting
  for it; the caller returns success as soon as the process is launched.
- **Why it shows up here:** Task Scheduler reported `LastTaskResult: 0` for a run that accomplished
  nothing. The 0 only confirmed `wscript.exe` started, not that the PowerShell script inside succeeded.

### Operational tooling has a different lifecycle than application code
- **Plain definition:** scripts that run on machine boot are maintained and versioned on a different
  rhythm than the app they support.
- **Why it shows up here:** the tunnel service ran daily and wrote logs constantly, while the repo
  is versioned per release — keeping them together meant log churn inside a git repo and confusion
  about what the repo actually contains.

## 7. Where it got stuck

**Snag 1 — "The task succeeded" was misleading.**
*Symptom:* `Get-ScheduledTaskInfo` showed `LastTaskResult: 0`, `LastRunTime: 11:01:55`.
*Cause:* the `.vbs` wrapper uses fire-and-forget `Run(..., 0, False)`, so the exit code reflects
only `wscript.exe` launching.
*Fix (diagnostic):* stopped trusting the task result and looked for independent evidence instead —
the absence of any log file for that run.

**Snag 2 — A `cloudflared` process was running, but was the wrong one.**
*Symptom:* `Get-Process cloudflared` showed a live process, suggesting the tunnel was fine.
*Cause:* a separate Windows **service** named `Cloudflared` (installed 2026-08-13, running as
`LocalSystem` with `tunnel run --token-file C:\ProgramData\cloudflared\token`) — a named tunnel
unrelated to DEM, auto-starting with Windows.
*Fix:* ruled it out via `sc.exe qc Cloudflared` showing a completely different command line. This
false lead is exactly why the user later asked to delete that service.

**Snag 3 — Root cause is inferred, not directly observed.**
The script crashed before writing anything, so the actual error was never captured. The inference —
`docker` not yet in PATH 13 seconds after boot — rests on three facts that fit together: the task
fired 13s after boot, no log exists for that run, and `$ErrorActionPreference = "Stop"` plus a bare
`docker info` call would produce exactly this signature. **Stated as a likely cause, not a proven
one.** The new `Write-Log "LOI: lenh 'docker' khong xuat hien trong PATH sau 60s."` line means a
recurrence will finally be observable.

**Snag 4 — Vercel CLI deploy failed twice, two different reasons.**
*Symptom A:* `vercel --prod` from `web/` → `The specified Root Directory "web" does not exist`.
*Cause A:* Vercel Project Settings has Root Directory = `web` (relative to repo root), so running
the CLI from inside `web/` makes Vercel look for `web/web`.
*Symptom B:* deploying from repo root → `File size limit exceeded (100 MB)` after uploading 4.4GB.
*Cause B:* the root has large data outside `web/`'s ignore scope.
*Fix:* abandoned the CLI path entirely and used `git push` to let Vercel's Git integration clone and
build with the configured Root Directory — no CLI upload limits involved. Cleaned up the stray
`.vercel` folder created at repo root during the attempt.

**Snag 5 — Upstream had already modified the same file.**
*Symptom:* `git pull --ff-only` refused: local changes to `scripts/start-dem-tunnel.ps1` would be
overwritten.
*Cause:* PR #11 (`fail-fast on Docker fatal log + verify Vercel env write`) had merged to
`origin/master` and touched the same function.
*Fix:* `git stash push -- <file>` → `git pull --ff-only` → `git stash pop`. Auto-merge succeeded
because the two changes were independent; re-ran the script end-to-end after merging before committing.

**Snag 6 — Two operations blocked by missing Administrator rights.**
*Symptom:* `sc.exe delete Cloudflared` and `Register-ScheduledTask -Force` both returned
`Access is denied`.
*Cause:* the session's PowerShell is not elevated; the service runs as `LocalSystem` and the task
registration touches a machine-level store.
*Fix:* verified the failed registration had **not** corrupted the existing task, then handed the
exact commands to the user to run in an elevated shell. User confirmed the task now points at
`E:\Monitoring\FRIMS\start-dem-tunnel-hidden.vbs`.

## 8. Verify

```powershell
# 1. Script runs clean from its new home
E:\Monitoring\FRIMS\start-dem-tunnel.ps1

# 2. Scheduled Task points at the new location
(Get-ScheduledTask -TaskName "EWATER-DEM-Tunnel-Startup").Actions.Arguments

# 3. Repo no longer carries the service
git -C E:\FRIMS_VINH_LONG\EWATER ls-files scripts/
```

Expected: (1) exits 0, ends with `Hoan tat.`, and the log lands in `E:\Monitoring\FRIMS\logs\`;
(2) prints `"E:\Monitoring\FRIMS\start-dem-tunnel-hidden.vbs"`; (3) prints only
`scripts/check-i18n.mjs`. Tunnel liveness: `curl` the new `*.trycloudflare.com` URL with
`/cog/info?url=/data/DEM_updated_VL_0m5_cog.tif` → HTTP 200.

## 9. Gotchas

- **The tunnel URL changes every restart.** Any manual note of a `*.trycloudflare.com` URL is stale
  the moment the tunnel restarts. Always read the current value from `web/.env.local`.
- **The script does not trigger a Vercel redeploy.** It only writes the env var; production keeps
  serving the old URL until the next build. Push to `master` (Git integration) rather than using
  `vercel --prod`, which is broken here by the Root Directory setting.
- **The service now depends on two absolute paths** (`$ServiceRoot`, `$EwaterRepoRoot`). Moving
  either the repo or the service directory breaks it silently until the next run's log is read.
- **`$DemCogFile` must match `DEM_COG_PATH`** in `web/src/lib/demLayer.ts`, otherwise the readiness
  poll passes against the wrong file or fails against a nonexistent one.
- The leftover Windows service `Cloudflared` was still present at the end of this session; deleting
  it requires an elevated shell.
