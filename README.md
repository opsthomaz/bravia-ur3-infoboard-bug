# Sony BRAVIA UR3: SystemUI crashes on every boot (InfoBoard Timer bug)

A reproducible firmware bug in Sony's Android TV SystemUI customization that makes
**every cold boot ~5–7 seconds slower** on 2020 BRAVIA UR3-chassis TVs (XH80/X80H
series and siblings). Documented here because, as of November 2026, no public
report of the root cause exists anywhere — only scattered "slow boot" complaints
in Sony's community forums.

> **TL;DR:** `com.sony.dtv.systemui.infoboard.InfoBoard` reuses a `java.util.Timer`
> after calling `cancel()` on it, which the Timer contract forbids. The next
> `scheduleAtFixedRate()` throws `IllegalStateException`, SystemUI dies with a
> FATAL EXCEPTION on every boot, Android restarts it, and the boot takes 5–7
> seconds longer. The fix is ~5 lines.

## Affected device / firmware

| | |
|---|---|
| Model tested | KD-55XH8096 (BRAVIA 4K UR3, EU) |
| Chassis | Uroboros / MediaTek MT5893 |
| Android | 10 (security patch 2026-02-01) |
| Firmware | v6.6230 — build `QTG3.200305.006.S97 662301` (latest public, 2026-04-29) |
| Likely affected | all 2020 UR3 TVs (XH80xx / X80xxH / X81xH) on current firmware |

The bug was first identified in **May 2026** and verified still present in
**November 2026** on the latest public firmware — the system dropbox logs a
`system_app_crash` for `com.android.systemui` on every single boot.

## The crash

```
FATAL EXCEPTION: main
Process: com.android.systemui
java.lang.IllegalStateException: Task already scheduled or cancelled
    at java.util.Timer.sched(Timer.java:408)
    at java.util.Timer.scheduleAtFixedRate(Timer.java:377)
    at com.sony.dtv.systemui.infoboard.InfoBoard.startUpdateClockTimer(InfoBoard.java:359)
    at com.sony.dtv.systemui.infoboard.InfoBoard.startClock(InfoBoard.java:210)
    at com.sony.dtv.systemui.infoboard.InfoBoard.onStart(InfoBoard.java:124)
    at com.sony.dtv.systemui.infoboard.InfoBoard.onConfigurationChanged(InfoBoard.java:146)
    at com.sony.dtv.systemui.SonySystemUIManager.lambda$onConfigurationChanged$4$SonySystemUIManager(SonySystemUIManager.java:329)
    at com.sony.dtv.systemui.SonySystemUIManager.startSonySystemUI(SonySystemUIManager.java:259)
```

100% reproducible — both via `adb reboot` and the physical power button.

## Root cause

`InfoBoard` keeps a single `java.util.Timer` to refresh its clock display.
During boot, `SonySystemUIManager` fires a rapid sequence of
`onConfigurationChanged` callbacks (display config, locale and density settling).
Each one runs `onStart → startClock`, and the intervening `stopClock` calls
`Timer.cancel()`.

Per the documented contract of `java.util.Timer`, **once `cancel()` is invoked
the instance is dead** — any later `schedule()`/`sched()` throws
`IllegalStateException`. The next `startUpdateClockTimer()` therefore crashes
SystemUI.

## Suggested fix (~5 lines)

```java
// current pattern — buggy
private final Timer clockTimer = new Timer();   // single instance
void stopClock() { clockTimer.cancel(); }
void startUpdateClockTimer() {
    clockTimer.scheduleAtFixedRate(...);         // throws after cancel()
}

// fixed
private Timer clockTimer;
void stopClock() {
    if (clockTimer != null) { clockTimer.cancel(); clockTimer = null; }
}
void startUpdateClockTimer() {
    if (clockTimer != null) clockTimer.cancel();
    clockTimer = new Timer();
    clockTimer.scheduleAtFixedRate(...);
}
```

Alternative: debounce the duplicate `onConfigurationChanged` callbacks during boot.

## Impact

- ~5–7 s added to every cold boot (SystemUI crash + auto-restart cycle)
- Brief black status bar during recovery
- No persistent damage — SystemUI recovers cleanly
- Matches the widely-reported but undiagnosed "slow startup / loading screens"
  complaints across Sony's community forums

## How to verify on your own TV

With ADB over the network (Developer options → enable ADB debugging):

```sh
adb connect <tv-ip>:5555
# crash history (one entry per boot):
adb shell dumpsys dropbox | grep -i systemui
# live capture right after a reboot:
adb reboot && sleep 60 && adb connect <tv-ip>:5555 && \
  adb shell logcat -d -b main,events | grep -A15 "FATAL EXCEPTION"
```

## Timeline of reports to Sony

| Date | Channel | Result |
|---|---|---|
| 2026-05-19 | Email to sonylistens@sony.com (full report + fix) | no reply |
| 2026-11-11 | Same channel, follow-up noting persistence on v6.6230 | no reply yet |
| 2026-11-11 | Formal case via Sony CH support web form | case opened, pending |

## Repository contents

- [`bug-report.md`](bug-report.md) — the full formal report as submitted to Sony
- [`evidence/stacktrace.txt`](evidence/stacktrace.txt) — the fatal exception
- [`evidence/persistence-2026-11.md`](evidence/persistence-2026-11.md) — dropbox crash records on the latest firmware
- [`firmware-notes.md`](firmware-notes.md) — UR3 firmware history and the `S97` ↔ `v6.6230` version mapping

---

*Root-cause analysis was done by the owner using AI-assisted log debugging as a tool.*
