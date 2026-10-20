# Badger Frame device record

Recorded 20 October 2026 for the private fork deployment.

| Item | Confirmed value |
| --- | --- |
| TV | Samsung The Frame `QN75LS03DDFXZA`, platform `24_PONTUSM_FTV` |
| Firmware | `T-PTMDAKUC-1400.1`, build `TIZEN-TRUNK2024-PontusM-HOTFIX-RELEASE_20260120.1` |
| Runtime | Tizen 8, 32-bit ARM, .NET 6 |
| CleanFrame package | `BadgrFrm01` |
| CleanFrame application | `BadgrFrm01.CleanFrame` |
| Jellyfin application | `AprZAARz4r.Jellyfin` |
| TizenTube application | `xvvl3S1TT1.TizenTubeStandalone` |
| Samsung Home escape | `com.samsung.tv.csfs` |
| Art Mode target | `org.tizen.art-app` |

The signed web widget installed and launched successfully. The owner confirmed that the CleanFrame screen rendered on the TV. Native Home assignment is maintained separately through the guarded helper in `scripts/native-home/`; its recovery baseline and output remain under ignored `.local/native-home/`.

The native Home workflow was tested as a full query, apply, exact restore, and reapply cycle. The immediate readback assigned both the native Home operation and `db/appfw/home_appid` to `BadgrFrm01.CleanFrame`, enabled its autostart flag, and retained the original `org.tizen.tv-viewer` startup application. A later readback found that the firmware had asynchronously rewritten `db/appfw/startup_appid` to `BadgrFrm01.CleanFrame`. The helper retries restoration for this reason. See [Badger Frame recovery](badger-tv-recovery.md) for the exact original values and recovery sequence.

Changing the Home target does not stop Samsung's original Home process. On this firmware, `com.samsung.tv.csfs` is marked for preload and remains resident with background launch enabled. The Tizen web runtime and other platform services also remain active. TizenTube additionally keeps Samsung's Cobalt/YouTube runtime available. CleanFrame changes which application handles Home; it does not replace or remove the underlying firmware services.

On 20 October 2026, selected Home children, preview/quick-panel applications and Prime Video were disabled through the package manager's per-application disable flag. Each Samsung child was changed separately, followed by a reboot and explicit launch test. The three Prime Video entries were changed together because they belong to one package. The database passed `PRAGMA integrity_check`; Samsung Home and CleanFrame continued to launch after every checkpoint. No package files were removed.

The disabled application IDs are recorded in [Badger Frame recovery](badger-tv-recovery.md), together with the targeted rollback transaction. AirPlay, Netflix and Apple TV remain enabled. `com.samsung.tv.coba.multiscreen` also remains enabled; it was not treated as the tested `com.samsung.tv.multiscreen` child.
