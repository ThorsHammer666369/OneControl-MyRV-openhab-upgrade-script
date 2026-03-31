# Original Upgrade Script Reference

This folder keeps the original `myrv_openhab_21_to_2512_upgrade.sh.txt` exactly as it was used earlier in development.

Do not edit that script in place.

It is kept for historical reference only, not as the current production upgrade path.

The supported script is the one in the parent folder:

- `../myrv_openhab_21_to_2512_upgrade.sh.txt`

## Why This Original Script Is Kept

- It preserves the first working draft and the original upgrade approach.
- It documents the path that led to the final script.
- It provides a stable before/after reference when comparing behavior against the corrected script.

## Confirmed Problems Found In The Original Script

- It used an openHAB JFrog key URL that later failed in live testing. The final script uses the current working public key endpoint.
- It did not protect the upgrade from the Monit watchdog reboot rule, so openHAB CPU churn during package install or startup could reboot the Pi mid-upgrade.
- It did not require an operator confirmation that a full raw SD-card image had been created before the upgrade.
- It cleared `/var/lib/openhab2/cache` and `/var/lib/openhab2/tmp` in place instead of moving those directories aside into timestamped restoreable paths.
- It recopied the custom `idsmyrv` and custom `openhabcloud` JARs, but it did not quarantine conflicting add-on JARs first and did not disable `/etc/openhab2/services/openhabcloud.cfg`, which made bundle collisions more likely.
- It did not normalize `/etc/openhab2/services/addons.cfg`, so the post-upgrade UI and add-on package selection was not preserved correctly.
- It restarted `myrvcg.service`, but the live image uses `myrv.service`, so the intended vendor restart path did not actually match the live system.
- It treated the upgrade as complete too early. It did not wait for openHAB REST readiness, did not wait through post-upgrade provisioning, and did not wait for `idsmyrv` Things or Items to repopulate before declaring success.
- It did not include the later final openHAB restart that testing showed can help the upgraded stack settle before `myrv.service` is restarted.

## Extra Behavior Removed From The Final Script

- Large exploratory backups that were not necessary for the production upgrade path, such as `/usr/share/openhab2`, `/opt`, `/etc/udev/rules.d`, and `/root/homekit`.
- The generated filesystem inventory map of `/`.
- The background Python 3 keyword scanner. This project targets a legacy environment where Python 3 cannot be assumed, and the scanner was exploratory rather than upgrade-critical.
- The old HomeKit service creation and management path. The final upgrade script was narrowed back to the intended scope: upgrade openHAB while preserving the stock OneControl / LCI system.
- The old restart logic that mixed `openhab2`, `myrvcg`, and optional HomeKit handling without a verified readiness check.

## Final Status Of This Original Script

- Keep it unchanged.
- Do not run it as the preferred upgrade method.
- Use it only as a historical reference when comparing the original draft against the corrected final script in the parent folder.
