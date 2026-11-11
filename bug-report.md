# Formal bug report (as submitted to Sony)

Sent to Sony on 2026-05-19 and again on 2026-11-11 (with the persistence note),
plus a formal support case opened via the Sony CH web form on 2026-11-11.
Serial number and personal contact details redacted here.

---

Subject: **[Bug Report] BRAVIA UR3 SystemUI crashes on every boot — InfoBoard / firmware QTG3.200305.006.S97 (KD-55XH8096)**

Hello Sony Support,

I'm writing to formally report a firmware bug I've identified in my Sony BRAVIA
TV. I have not factory-reset the unit because this is a deterministic crash
inside Sony's own SystemUI customization code path — the issue is evident from
the stack trace and cannot be resolved by user-level configuration.

## Device

```
Model:               BRAVIA 4K UR3 (sales name KD-55XH8096, sold in Switzerland)
Build fingerprint:   Sony/BRAVIA_UR3_EU/BRAVIA_UR3:10/QTG3.200305.006.S97/662301:user/release-keys
Android version:     10 (security patch 2026-02-01)
Chassis:             Uroboros / MediaTek MT5893
Bootloader:          locked, AVB green
```

## Symptom

On every cold boot, `com.android.systemui` crashes with a FATAL EXCEPTION early
in the boot sequence. Android auto-restarts SystemUI, but the recovery cycle
adds approximately 5–7 seconds to total boot time before the home screen is
fully responsive.

100% reproducible — both via `adb reboot` and the physical power button.

First identified in May 2026; still present as of November 2026 on the latest
public firmware for this chassis (v6.6230, build QTG3.200305.006.S97 662301) —
the system dropbox records a `system_app_crash` entry for
`com.android.systemui` on every boot.

## Stack trace

Captured via `adb logcat -b events,main` immediately after reboot:

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

## Root cause

InfoBoard maintains a `java.util.Timer` to refresh a clock display. In
`startUpdateClockTimer` (line ~359), `Timer.scheduleAtFixedRate()` is called on
a Timer instance that was previously cancelled by `stopClock` (which calls
`Timer.cancel()`).

Per the documented contract of `java.util.Timer`, once `cancel()` is invoked,
the same Timer instance becomes unusable — any subsequent `sched()`/`schedule()`
call throws `IllegalStateException`.

The trigger is a rapid sequence of `onConfigurationChanged` callbacks fired by
`SonySystemUIManager` during the boot sequence, as display configuration, locale
and density settle. Each callback invokes
`InfoBoard.onConfigurationChanged → onStart → startClock`, and the intervening
`stopClock → cancel` leaves the next iteration of `startUpdateClockTimer`
unable to schedule on the same Timer instance.

## Suggested fix

In `InfoBoard.java`, allocate a fresh Timer instance whenever the clock is
(re)started:

```java
private Timer clockTimer;
void stopClock() {
    if (clockTimer != null) {
        clockTimer.cancel();
        clockTimer = null;
    }
}
void startUpdateClockTimer() {
    if (clockTimer != null) clockTimer.cancel();
    clockTimer = new Timer();
    clockTimer.scheduleAtFixedRate(...);
}
```

An alternative is to debounce or short-circuit duplicate
`onConfigurationChanged` callbacks during boot.

## Impact

- ~5–7 seconds added to every cold boot
- Brief moment of black status bar during the recovery
- No persistent functional impact — SystemUI recovers cleanly
- Likely affects all UR3 chassis (KD-XHxx series 2020 in EU) with current firmware

## Why report it

Sony still ships security patches for the UR3 chassis (patch level 2026-02-01
on a 2020 device is excellent). This bug is minor in real-world impact but the
fix is trivial, the root cause is clear from the stack trace alone, and shipping
the fix in a security patch rollup would polish the boot experience for all UR3
owners.
