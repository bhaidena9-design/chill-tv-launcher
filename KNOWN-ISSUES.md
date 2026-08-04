# Known issues & fixes

Real bugs hit on the audited TV (Realme/Changhong DVB91RM, Android TV 11), with
the diagnosis that found them and the fix that closed them. Everything here was
reproduced and verified on hardware — no speculative advice.

---

## 1. TV appears to restart on its own — `com.mediatek.tv.service` crash loop

**Status:** fixed 2026-08-04 · **Cause:** this repo's own debloat list

### Symptom

Every so often the TV blanks and comes back at the launcher, as if it rebooted.
No pattern the user could pin down.

### Root cause

`debloat.sh` used to remove `com.google.android.leanbacklauncher.partnercustomizer`
as an ads package. It *is* an ads package — but it also owns the ContentProvider
authority **`tvlauncher.widget`**:

```
[tvlauncher.widget]:
  Provider{... com.google.android.leanbacklauncher.partnercustomizer/.ConfigurationProvider}
```

MediaTek's `com.mediatek.tv.service` is flagged **`PERSISTENT`** in its manifest,
and it registers a network callback that calls `ContentResolver.notifyChange()`
on that authority every time the network comes up:

```
FATAL EXCEPTION: ConnectivityThread
Process: com.mediatek.tv.service
java.lang.SecurityException: Failed to find provider tvlauncher.widget for user 0;
    expected to find a valid ContentProvider for this authority
    at ...MyNetworkCallback.notifyChange(MyStartUp.kt:40)
    at ...MyNetworkCallback.onAvailable(MyStartUp.kt:25)
```

With the provider gone the exception is fatal, the process dies — and because
it's persistent, Android immediately respawns it, where it crashes again. The
observed rate was **one crash every ~6 seconds, indefinitely**, with a fresh PID
each time.

Knock-on effects:

- `system_server` spamming `WTF ActivityManager: Restarting service that is not needed`.
- The system dropbox saturated at **300/300 entries**, all from this loop —
  evicting every other diagnostic record, so real crash evidence was being lost.
- Constant process-spawn and dex-load churn on a 1 GB device.

A persistent-process crash loop is the standard cause of a **runtime restart**
(zygote/system_server restart): the screen blanks, the boot animation plays, and
you land back at the launcher. It looks exactly like a reboot but never reaches
the bootloader, which is why `ro.boot.bootreason` still reads
`shutdown,userrequested` from the last real power cycle.

> Honest scope note: the crash loop is proven. The runtime-restart mechanism is
> the leading explanation for the visible restarts, but no kernel panic or
> watchdog was ever recorded in `persist.sys.boot.reason.history` — the restart
> itself was not captured in the act.

### Fix

```bash
adb shell cmd package install-existing com.google.android.leanbacklauncher.partnercustomizer
```

`debloat.sh` no longer removes this package. Keeping it costs nothing visible:
its only job is injecting partner/sponsored rows into the **stock** launcher,
which stays disabled.

### Verifying

```bash
# provider must resolve
adb shell dumpsys package providers | grep -A2 '\[tvlauncher.widget\]'

# PID must stay stable — before the fix it never survived past ~6s
adb shell ps -o PID,ETIME,NAME -p $(adb shell pidof com.mediatek.tv.service)
```

Result after the fix: same PID across a 90-second watch, **0 crashes** in the
event log over that window (~15 expected at the old rate).

---

## 2. Clock stops updating automatically

**Status:** improved; a firmware limit remains

### Symptom

The TV's time is occasionally wrong and stays wrong for a long stretch, then
silently fixes itself.

### Root cause

This panel has **no battery-backed RTC**. When it can't reach NTP it boots to
the epoch fallback — visible in the boot history as timestamps around
`946684809` (2000-01-01 00:00:09 UTC).

The firmware's polling policy is the real problem:

```
$ adb shell dumpsys network_time_update_service
PollingIntervalMs:        +1d0h0m0s0ms   # once a day
PollingIntervalShorterMs: +1m0s0ms       # retry gap
TryAgainTimesMax:         3              # then give up
```

If NTP fails 3 times at boot, one minute apart, the service **gives up for a
full 24 hours**. Issue #1 made this materially worse: the crash loop fired on
network-available at boot — precisely the window those three attempts use.

`auto_time` and `auto_time_zone` are already `1` and the timezone is correct.
Chasing those settings is a dead end.

### Fix (partial)

```bash
adb shell settings put global ntp_server time.google.com
adb shell settings put global ntp_timeout 10000
```

Google's anycast NTP answers in ~4 ms from this TV; the default was unset, so it
used the firmware's pool. Better odds of landing inside those three attempts.

**What can't be fixed without root:** the polling interval, retry gap and retry
count come from framework resources, not from `settings`. Changing them needs
root or a framework overlay.

Also worth knowing: a VPN app that auto-connects at boot can block NTP's UDP 123
and reintroduce this. NordVPN is installed on the audited TV but was not
connected during diagnosis, so it was not the cause here.

---

## 3. App cache creeping up on a small `/data`

**Status:** maintenance, not a bug

Streaming apps quietly accumulate hundreds of MB. On the audited TV, 68% of a
4.2 GB `/data` was used, ~408 MB of it cache — including 32 MB held by the
**disabled** stock launcher.

```bash
adb shell pm trim-caches 4500000000     # target free bytes; exceeds the partition, so trims everything
adb shell df -h /data/user/0
```

Freed ~390 MB (68% → 58% used). Safe: it only touches `cache/` directories,
never app data. Note it also clears Chill's ~0.2 MB icon cache, so the next cold
start re-renders art once and is back to instant afterwards.

Per-app breakdown without root:

```bash
adb shell dumpsys diskstats     # "Package Names" and "Cache Sizes" arrays line up by index
```

---

## Debugging notes for this platform

Things that cost time to figure out, worth knowing before you chase a ghost:

- **Load average sits at 90–110 permanently, and that's normal.** Dozens of
  MediaTek/MStar kernel threads (`DMX_Filt_Task`, `GPIO poll Task`,
  `T_sensor_Check`) park in uninterruptible `D` state by design, and Linux counts
  D-state tasks into the load average. Judge CPU by `top`'s idle % instead — the
  TV was ~50% idle while showing a load of 104.
- **No root, no `su`.** `/data/tombstones`, `/data/anr` and `/data/data/*/cache`
  are unreadable by the shell user. Use `dumpsys dropbox --print <tag>` and
  `dumpsys diskstats` instead.
- **Real reboot history lives in `persist.sys.boot.reason.history`**, and it's
  multi-line — a plain `getprop | grep` truncates it. Read it directly:
  ```bash
  adb shell getprop persist.sys.boot.reason.history
  ```
- **`exec-out screencap` emits garbage on this TV.** Use
  `adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png`.
- **Check nobody's watching before sending key events:**
  ```bash
  adb shell dumpsys activity activities | grep mResumedActivity
  ```

## Reporting something new

Include the output of `adb shell getprop persist.sys.boot.reason.history`,
`adb shell dumpsys dropbox | tail -20`, and the package list from
`adb shell pm list packages -u -f` — the last one matters because most issues in
this repo trace back to something the debloat list removed.
