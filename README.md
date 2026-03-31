# MyRV openHAB 2.1 -> 2.5.12 Upgrade Script

## Critical Warning

Before you run the upgrade, make a full raw image of the Pi's SD card.

Do not skip this. This project treats the SD card as irreplaceable because it may contain box-specific OEM and cloud identifiers tied to the coach gateway. If those identifiers are lost and you do not have a full raw image of the card, there may be no simple way to recover them.

The safest rollback is not a file-level restore. The safest rollback is writing the full pre-upgrade SD-card image back to a replacement card.

This script and documentation are independent community work. They are not affiliated with, endorsed by, or supported by Lippert Components, LCI, OneControl, MyRV, openHAB, SmartThings, Samsung, or the RV manufacturer.

Use of the script is entirely at your own risk and without warranty. This README is operational guidance only and is not legal advice.

The script itself now stops and requires an explicit operator confirmation that:

- a full raw SD-card image has already been made and saved
- the operator accepts all risk before the upgrade proceeds

## What This Repository Contains

- `myrv_openhab_21_to_2512_upgrade.sh.txt`
  - Source file that generates `/root/myrv_openhab_21_to_2512_upgrade.sh` on the Pi.
- `README.md`
  - This procedure and safety guide.
- `old-original-reference/`
  - Historical copy of the earlier draft script plus notes describing the mistakes and extra scope that were removed from the final version.

## What the Script Does

The script upgrades a legacy LCI / MyRV image from `openhab2 2.1.0-1` to `openhab2 2.5.12-1` while preserving the custom MyRV-specific pieces this project depends on.

High-level behavior:

- Creates a timestamped restore point before making changes.
- Backs up key openHAB, add-on, log, and OEM mount data.
- Stages the custom `idsmyrv` binding JAR and the custom `org.openhab.io.openhabcloud-2.3.0-SNAPSHOT.jar`.
- Adds the openHAB stable package repository and upgrades `openhab2` and `openhab2-addons` to `2.5.12-1`.
- Quarantines conflicting or stale add-on JARs out of `/usr/share/openhab2/addons`.
- Restores the custom MyRV add-ons after the package upgrade.
- Normalizes `/etc/openhab2/services/addons.cfg` so the post-upgrade package selection stays compatible with this system and preserves the stock legacy UI set (`basic`, `paper`, `habmin`, `habpanel`, plus `restdocs`) while keeping `classic` out.
- Disables `/etc/openhab2/services/openhabcloud.cfg` if it exists, to avoid reactivating the stock cloud path when the custom cloud JAR is in use.
- Leaves the stock `myrv.service` and `rc.local` startup behavior alone.
- Restarts openHAB once for the package/config changes, waits through the post-upgrade provisioning window, then performs one final openHAB restart before restarting the stock `myrv.service`.
- Moves `/var/lib/openhab2/cache` and `/var/lib/openhab2/tmp` aside with timestamped names and recreates them empty.
- Temporarily changes the Monit openHAB CPU rule to `alert` at `65%` during the upgrade so package churn does not reboot the Pi.
- Restores the Monit rule to `reboot` at `65%` at the end of the script. used to be 50% but increased to 65% because openhab 2.5.12 uses more resources.
- Waits for openHAB REST after each restart, allows the upgraded runtime to settle, and then waits for `idsmyrv` activity to reappear before declaring success.

## What the Script Saves

The script creates a timestamped backup root here:

- `/root/rvsi-restorepoints/<timestamp>-openhab-21-to-2512-upgrade/`

It also updates this convenience link:

- `/root/rvsi-last-upgrade-backup`

Typical contents of the restore point:

- `openhab2-conf.tar.gz`
- `openhab2-userdata.tar.gz`
- `openhab2-addons.tar.gz`
- `openhab2-logs.tar.gz`
- `mnt-cfile.tar.gz`
- `custom-addons/`
- `addons-quarantine/`
- `dpkg-list.txt`
- `mounts.txt`
- `openhab2-status.txt`
- `myrv-status.txt`
- `systemd/myrv.service`
- `myrvcg/monit.cfg`
- `services/addons.cfg`
- `services/openhabcloud.cfg`

The script also moves these old runtime directories aside instead of deleting them in place:

- `/var/lib/openhab2/cache.pre-upgrade.<timestamp>`
- `/var/lib/openhab2/tmp.pre-upgrade.<timestamp>`

## What the Script Upgrades or Changes

Packages:

- `openhab2` -> `2.5.12-1`
- `openhab2-addons` -> `2.5.12-1`

Configuration and runtime changes:

- `/etc/apt/sources.list.d/openhab2.list`
- `/etc/openhab2/services/addons.cfg`
- `/etc/openhab2/services/openhabcloud.cfg` if present
- `/etc/myrvcg/monit.cfg`
- `/var/lib/openhab2/cache`
- `/var/lib/openhab2/tmp`
- `/usr/share/openhab2/addons`

## What the Script Does Not Protect You From

- It does not create the full SD-card image for you.
- It does not know how to reconstruct OEM or cloud identifiers if the raw SD-card contents are lost.
- It does not automatically install or start services that are not already part of the image, such as `mosquitto`, if they are absent.
- It does not replace the need for a real offline backup.

## Make a Full SD-Card Image First

### Windows: Win32 Disk Imager

Use Win32 Disk Imager's `Read` function to copy the entire SD card to a raw image file.

Suggested workflow:

1. Shut the Pi down cleanly.
2. Remove the SD card from the Pi.
3. Insert the SD card into a card reader on your Windows machine.
4. Start Win32 Disk Imager as Administrator.
5. In `Image File`, choose a new output path such as `D:\Backups\myrv-pre-upgrade-20260330.img`.
6. Select the correct removable device letter for the SD card.
7. Double-check the selected device.
8. Click `Read`.
9. Wait for `Read Successful`.
10. Keep that `.img` file somewhere safe before attempting the upgrade. Better yet get a different SD card and write the image on that and use that instead. Keep OEM safe.

To restore with Win32 Disk Imager:

1. Insert the target SD card.
2. Open the saved `.img` file.
3. Select the correct removable device.
4. Double-check the device again.
5. Click `Write`.

This is destructive to the target card. Writing the wrong device will overwrite it.

### Linux or macOS: `dd`


Linux example:

```bash
lsblk
sudo umount /dev/sdX1 /dev/sdX2 2>/dev/null || true
sudo dd if=/dev/sdX of=./myrv-pre-upgrade-YYYYMMDD.img bs=4M status=progress conv=fsync
sync
```

Restore example:

```bash
sudo dd if=./myrv-pre-upgrade-YYYYMMDD.img of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

Replace `/dev/sdX` with the whole SD-card device, not a partition such as `/dev/sdX1`.

macOS usually uses `/dev/diskN` or `/dev/rdiskN` instead of `/dev/sdX`. Unmount the card first, then use the whole-disk device.

## The Image File Will Be Large

A raw SD-card image is usually the full size of the card, not just the used space. That is normal.

If you want to make the image easier to store or faster to write back onto a fresh SD card, you can shrink or compress a copy after you have already saved the original full raw image.

For this project, removing the empty space can reduce the image enough to fit on an 8 GB card, which also makes writing it back to a fresh SD card faster.

One way to do that is with a shell command in the same family as `dd` that removes the unused space from a copy of the image.

Keep the original full raw image untouched, and do not make the shrunk copy your only backup.

After writing a shrunken image to a fresh card and booting it, use Raspberry Pi configuration to expand the filesystem back to the full size of the SD card.

## How to Run the Upgrade

On the Pi, from a root shell:

1. Paste the contents of `myrv_openhab_21_to_2512_upgrade.sh.txt`.
2. Run:

```bash
chmod +x /root/myrv_openhab_21_to_2512_upgrade.sh
/root/myrv_openhab_21_to_2512_upgrade.sh
```

## How to Restore

### Best Restore Method

If the upgrade goes badly, restore the full SD-card image you made before the upgrade.

That is the preferred recovery path because it restores:

- The full filesystem
- OEM state
- Cloud-related state
- Box-specific identifiers that may not be reconstructible

### Script-Level Restore Method

If you choose to restore only from the script backup instead of the full SD-card image, use the restore point under:

- `/root/rvsi-restorepoints/<timestamp>-openhab-21-to-2512-upgrade/`

At minimum, review and restore as needed:

- `openhab2-conf.tar.gz`
- `openhab2-userdata.tar.gz`
- `openhab2-addons.tar.gz`
- `myrvcg/monit.cfg`
- `systemd/myrv.service`
- `services/addons.cfg`
- `services/openhabcloud.cfg`
- `custom-addons/`
- `addons-quarantine/`

If you only need to roll back the Monit CPU watchdog:

- Restore `myrvcg/monit.cfg` from the restore point back to `/etc/myrvcg/monit.cfg`
- Validate with `monit -t`
- Restart Monit with `systemctl restart monit`

## Recommended Validation After the Upgrade

Check at least:

```bash
systemctl status openhab2 --no-pager
systemctl status myrv.service --no-pager
systemctl status monit --no-pager
curl -fsS http://127.0.0.1:8080/rest/
curl -fsS http://127.0.0.1:8080/rest/bindings
curl -fsS http://127.0.0.1:8080/rest/items | head
tail -n 200 /var/log/openhab2/openhab.log
```

For this project, also confirm that `idsmyrv` activity has returned through REST.

## References

- [Win32 Disk Imager project page](https://sourceforge.net/projects/win32diskimager/)
- [Win32 Disk Imager wiki home](https://sourceforge.net/p/win32diskimager/wiki/)
- [Raspberry Pi Magazine: back up your Raspberry Pi](https://magazine.raspberrypi.com/articles/back-up-raspberry-pi?class=c-link)
