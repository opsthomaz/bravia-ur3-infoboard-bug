# Persistence check — November 2026

Six months after the bug was first identified (May 2026), the TV is on the
latest public firmware for the UR3 chassis and the crash still happens on
every boot.

## Firmware at time of check (2026-11-11)

```
$ adb shell getprop ro.build.fingerprint
Sony/BRAVIA_UR3_EU/BRAVIA_UR3:10/QTG3.200305.006.S97/662301:user/release-keys

$ adb shell getprop ro.build.version.security_patch
2026-02-01

$ adb shell getprop ro.build.date
Wed Apr  8 14:37:14 JST 2026
```

This build corresponds to the last public release **v6.6230** (published
2026-04-29) — see [firmware-notes.md](../firmware-notes.md).

## Dropbox crash records

The system dropbox shows a `system_app_crash` for `com.android.systemui` on
each of the two most recent boots:

```
$ adb shell dumpsys dropbox | grep -i systemui
2026-11-09 03:30:32 system_app_crash (text, 1804 bytes)
    Process: com.android.systemui/PID: 3347/UID: 10143/Flags: 0x30c83e0d/P ...
2026-11-10 16:39:50 system_app_crash (text, 1804 bytes)
    Process: com.android.systemui/PID: 3342/UID: 10143/Flags: 0x30c83e0d/P ...
```

```
$ adb shell dumpsys dropbox --print | grep -A6 '2026-11-10 16:39:50'
2026-11-10 16:39:50 system_app_crash (text, 1804 bytes)
Process: com.android.systemui
PID: 3342
UID: 10143
Flags: 0x30c83e0d
Package: com.android.systemui v29 (3.0.55)
Foreground: No
```

One crash entry per boot, every boot. Conclusion: the bug shipped unfixed in
every firmware release through v6.6230.
