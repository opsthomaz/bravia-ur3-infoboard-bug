# UR3 firmware notes (XH80 / X80H family)

Collected while investigating the InfoBoard bug. The UR3 chassis (2020,
MediaTek MT5893 "Uroboros") shipped with Android 9 and was upgraded to
Android 10; firmware uses the `PKG6.xxxx` / `v6.xxxx` numbering shared by the
2018–2020 Android chassis generations (UR1/UR2/UR3/VH1/VH2).

## Known public releases (EU)

| Marketing version | Date | Notes |
|---|---|---|
| v6.5076 / v6.5077 | 2021–2022 | early updates |
| v6.6130 | 2022/2023 | intermediate |
| v6.7220 | 2023-09-20 | Android 9 → 10 upgrade (new Settings UI, Living Decor, removed Photo Sharing Plus due to CVE) |
| v6.6188 | ~2024 | security patch (NA) |
| v6.6180 | 2025-02-11 | security patch (EU) |
| v6.6212 | 2026-02-11 | security patch |
| **v6.6230** | **2026-04-29** | **latest public release** |

## Mapping the on-device build to a marketing version

The TV reports only the internal build string:

```
QTG3.200305.006.S97 662301   (built 2026-04-08, security patch 2026-02-01)
```

Sony's download pages only show `v6.xxxx` numbers, and for months no public
page listed anything newer than v6.6212 (2026-02-11), making `S97` look like a
mystery build. The release of **v6.6230 on 2026-04-29** — three weeks after the
`S97` build date of 2026-04-08 — closes the gap: `S97 662301` **is** v6.6230.

## Other UR3 facts

- **Downgrade is impossible**: Sony states it officially, and AVB + rollback
  fuses enforce it (A/B units can only fall back to the previous slot before
  the first post-update reboot).
- All UR3 market variants report the same identity: user agent
  `BRAVIA 4K UR3 Build/QTG3.200305.006.Sxx`.
- The InfoBoard crash is present at least on `S97 662301` (v6.6230) and the
  builds before it; given the stack trace it has likely been there since the
  Android 10 upgrade (v6.7220, 2023).
