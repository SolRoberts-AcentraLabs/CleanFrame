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
