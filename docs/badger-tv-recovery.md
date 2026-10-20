# Badger Frame recovery runbook

Recorded 20 October 2026 for the private `QN75LS03DDFXZA` running
`T-PTMDAKUC-1400.1`. These values and commands are specific to this TV. The TV
is reachable only from `optimus`. Keep both current LAN addresses in private
local configuration rather than in this repository.

## Recovery assets

- CleanFrame app: `BadgrFrm01.CleanFrame`
- Samsung Home app: `com.samsung.tv.csfs`
- Jellyfin app: `AprZAARz4r.Jellyfin`
- TizenTube app: `xvvl3S1TT1.TizenTubeStandalone`
- Local native-Home baseline: `.local/native-home/original.json`
- Copy on `optimus`: `/tmp/cleanframe-native/.local/native-home/original.json`
- TV package-database backup:
  `/opt/usr/home/owner/share/tmp/cleanframe-recovery/pkgmgr_parser.before-home-children.db`
- Backup SHA-256:
  `6590e4475d7ff2ae908d4b7c3d8acaa7ebb1efe2ba2e7d32d42edec92e6f69d5`

Keep `com.samsung.tv.csfs` installed and enabled. It is the recovery Home and
still launches successfully. Do not disable `com.samsung.tv.coba.multiscreen`;
only the separately tested `com.samsung.tv.multiscreen` app is disabled.

## Connect from optimus

From the workstation:

```sh
ssh -F /dev/null \
  -o IdentityAgent=/run/user/1000/ssh-agent \
  -o IdentitiesOnly=no \
  -o BatchMode=yes \
  badger@"$OPTIMUS_ADDRESS"
```

Set `OPTIMUS_ADDRESS` from private local configuration before connecting. On
`optimus`, the private CleanFrame copy already contains the TV address:

```sh
cd /tmp/cleanframe-native
TV_IP="$(python3 -c 'import json; print(json.load(open(".local/device.json"))["host"])')"
/home/badger/.local/bin/sdb connect "$TV_IP"
/home/badger/.local/bin/sdb devices
```

The device list must show `$TV_IP:26101` in `device` state. If it does not,
follow **SDB is unavailable** below before changing anything.

## Fast recovery when the screen or Home button is wrong

First try launching CleanFrame directly:

```sh
/home/badger/.local/bin/sdb -s "$TV_IP:26101" \
  shell 0 execute BadgrFrm01.CleanFrame
```

If CleanFrame does not render, launch Samsung Home:

```sh
/home/badger/.local/bin/sdb -s "$TV_IP:26101" \
  shell 0 execute com.samsung.tv.csfs
```

If the SDK launcher syntax changes, use the already-reviewed root helper on
`optimus` while it is present:

```sh
python3 /tmp/frame_root.py 'app_launcher -s com.samsung.tv.csfs'
```

Samsung Home remains installed for this purpose. The service changes below do
not delete application files.

## Restore the original native Home assignment

The exact pre-change baseline is private and is already stored in
`/tmp/cleanframe-native/.local/native-home/original.json` on `optimus`. It
contains:

| Value | Original setting |
| --- | --- |
| Native Home operation | `com.samsung.tv.csfs` |
| `db/appfw/home_appid` | `com.samsung.tv.csfs` |
| `db/appfw/startup_appid` | `org.tizen.tv-viewer` |
| `db/appfw/booting_appid_backup` | `BadgrFrm01.CleanFrame` |
| `db/first_screen/type` | `com.samsung.tv.csfs` |
| `db/menu/smart_hub/ontvsetting/auto_start` | `0` |

Open a temporary callback port and run the guarded restore on `optimus`:

```sh
sudo -n firewall-cmd --zone=home \
  --add-rich-rule="rule family=\"ipv4\" source address=\"$TV_IP/32\" port port=\"43761\" protocol=\"tcp\" accept" \
  --timeout=180
cd /tmp/cleanframe-native
CLEANFRAME_SDB=/home/badger/.local/bin/sdb \
  CLEANFRAME_CALLBACK_PORT=43761 \
  python3 scripts/native-home/run.py restore
```

Then inspect instead of assuming the transport completed:

```sh
CLEANFRAME_SDB=/home/badger/.local/bin/sdb \
  CLEANFRAME_CALLBACK_PORT=43761 \
  python3 scripts/native-home/run.py inspect
```

The helper retries restoration because this firmware can rewrite a setting
after the Home operation changes. After the final apply in this deployment,
the firmware later changed `db/appfw/startup_appid` to
`BadgrFrm01.CleanFrame`; this delayed rewrite is expected drift and is why
recovery must use `restore` followed by `inspect`.

To reapply CleanFrame as Home after recovery, replace `restore` with `apply`,
then run `inspect` and physically test Home from another app.

## Re-enable the disabled Samsung children and Prime Video

The following app rows were set to `app_disable='true'`, one change and reboot
at a time. Every launch was rejected after reboot, while `com.samsung.tv.csfs`
and CleanFrame continued to launch:

- `com.samsung.tv.home.media`
- `com.samsung.tv.multiscreen`
- `com.samsung.tv.home.sidebar`
- `com.samsung.tv.preview-downloader`
- `com.samsung.tv.preview-updater`
- `com.samsung.tv.quickpanel`
- `com.samsung.tv.stpreview-quickpanel`
- `com.samsung.tv.ST-LifePlus`
- `org.tizen.primevideo`
- `org.tizen.primevideo-dummy`
- `org.tizen.primevideo_companionapp`

AirPlay, Netflix and Apple TV were left enabled. Apple TV uses package
`com.samsung.tv.aria-video` on this firmware.

To reverse only these flags, use root on the TV and run this transaction:

```sql
BEGIN IMMEDIATE;
UPDATE package_app_info
SET app_disable = 'false'
WHERE app_id IN (
  'com.samsung.tv.home.media',
  'com.samsung.tv.multiscreen',
  'com.samsung.tv.home.sidebar',
  'com.samsung.tv.preview-downloader',
  'com.samsung.tv.preview-updater',
  'com.samsung.tv.quickpanel',
  'com.samsung.tv.stpreview-quickpanel',
  'com.samsung.tv.ST-LifePlus',
  'org.tizen.primevideo',
  'org.tizen.primevideo-dummy',
  'org.tizen.primevideo_companionapp'
);
SELECT changes();
COMMIT;
PRAGMA integrity_check;
```

The database is `/opt/dbspace/.pkgmgr_parser.db`. Confirm that the update
matches exactly 11 rows, that `PRAGMA integrity_check` returns `ok`, then
`sync` and reboot. The app framework caches these values; a live database
write does not reliably change launch behaviour until reboot.

Prefer this targeted transaction over replacing the whole package database.
The full backup predates the service changes and is a last-resort artifact; it
can also discard later legitimate package registrations. Before relying on
it, verify its hash and run an SQLite integrity check on the copy.

## Root-helper callback is stuck

A reboot issued inside `/tmp/frame_root.py` can leave its Python process on
`optimus` waiting with port 43759 open. Identify the exact process before
stopping it:

```sh
ps -ef | grep '/tmp/frame_root.py' | grep -v grep
ss -ltnp | grep -E '43759|43760'
```

Terminate only the stale PID shown by both checks:

```sh
kill <PID>
```

Do not kill unrelated Python services. Reconnect SDB, then retry the query.

## SDB is unavailable

1. Confirm the TV responds on the trusted LAN and that `.local/device.json`
   contains its current address.
2. On the TV, open **Apps → App Settings**, enter `12345` with the remote's
   number pad, and confirm Developer Mode is on with the current LAN address
   of `optimus` as its host PC.
3. If Samsung requests a restart, avoid interrupting an update. A short Power
   press may enter Art Mode. Power the TV fully off, wait ten seconds, restore
   power, and retry SDB from `optimus`.
4. If CleanFrame is broken but Samsung Home is visible, use Samsung Home while
   repairing or reinstalling the signed widget with the existing package and
   author identity.
5. Use a factory reset only after the saved native-Home restore and targeted
   service rollback cannot be run. A reset removes accounts, apps and settings
   and may require developer access and certificates to be configured again.

## Verification after any recovery

Check all of these before declaring recovery complete:

1. Samsung Home launches through `com.samsung.tv.csfs`.
2. CleanFrame launches and the physical Home button reaches the intended app.
3. Jellyfin and TizenTube launch from CleanFrame.
4. Art Mode still opens through `org.tizen.art-app`.
5. AirPlay, Netflix and Apple TV remain available.
6. The package database returns `ok` from `PRAGMA integrity_check`.
7. Reboot once more and repeat the Home and CleanFrame checks.
