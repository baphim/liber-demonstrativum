---
tags:
  - runbook
  - "#nextcloud"
---

# Update Nextcloud from the command line

Updates a Nextcloud install that was many releases behind, using `occ` and the updater instead of the web updater. I started at 33.0.3, went to 33.0.8 with the updater, then did a major upgrade to 34.0.3 by hand when the updater could not download the release. See [Nextcloud](../services/nextcloud.md).

Run `occ` as the web server user, not as root. Running it as root changes file ownership and breaks the install.

## Step 1: Find out how the install was built
If the build was never written down, work it out from the system.
```
php -v
ls -la /var/www/
find / -maxdepth 4 -iname "occ" 2>/dev/null
```

## Step 2: Check the current state
```
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ status"
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ maintenance:mode"
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ update:check"
```
If SSH is as a non-root user that cannot `su -` to root, get a root shell through `pct enter <ctid>` from the Proxmox host.

## Step 3: Back up the container
From the Proxmox host. Stop mode gives a consistent copy.
```
vzdump <ctid> --storage <backup-storage> --mode stop --compress zstd
```

## Step 4: Run the updater
```
su -s /bin/bash www-data -c "php /var/www/nextcloud/updater/updater.phar"
```
Verify afterward with `occ status`, `occ maintenance:mode` and `occ app:list --shipped=false`. Run `update:check` again. Repeat steps 3 and 4 for each release until it reports nothing newer, or until a major version.

## Step 5: If the updater cannot download, do it by hand
My major upgrade failed twice at 0% on the download step, even though `curl` from the container reached the download server cleanly. Reading around, the likely cause is the updater's PHP stream, possibly related to the reverse proxy. I did not confirm that. The workaround is to fetch the release and verify it myself.
```
cd /tmp
curl -O https://download.nextcloud.com/server/releases/nextcloud-<version>.zip
curl -O https://download.nextcloud.com/server/releases/nextcloud-<version>.zip.asc
curl -o nextcloud.asc https://nextcloud.com/nextcloud.asc
gpg --import nextcloud.asc
gpg --verify nextcloud-<version>.zip.asc nextcloud-<version>.zip
```
The signature check should come back good.

## Step 6: Swap in the new release
```
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ maintenance:mode --on"
cd /var/www
mv nextcloud nextcloud-<old-version>-backup
unzip -q /tmp/nextcloud-<version>.zip -d /var/www/
cp -a /var/www/nextcloud-<old-version>-backup/config/. /var/www/nextcloud/config/
grep datadirectory /var/www/nextcloud/config/config.php
```
If the data directory is outside the web root, it needs no copying.

## Step 7: Check for third-party apps
Compare the app folders. Apps that were not in the new release are missing, and each one is a decision: copy it over or leave it.
```
ls /var/www/nextcloud-<old-version>-backup/apps/ | sort > /tmp/old-apps.txt
ls /var/www/nextcloud/apps/ | sort > /tmp/new-apps.txt
diff /tmp/old-apps.txt /tmp/new-apps.txt
```
I audited the five that were not bundled and chose not to copy them. They can be installed again later.

## Step 8: Finish the upgrade
```
chown -R www-data:www-data /var/www/nextcloud
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ upgrade"
su -s /bin/bash www-data -c "php /var/www/nextcloud/occ maintenance:mode --off"
```
Verify as in step 4. Once the new version has run cleanly, remove the old copy with `rm -rf /var/www/nextcloud-<old-version>-backup`. The container backup from step 3 remains as the rollback.

## Verification
- `occ status` shows the new version and maintenance mode off.
- `occ update:check` reports nothing pending.
- The web interface loads and files open.

## Lessons
- Write the build down when you do it. Reconstructing how an install was made, months later, took longer than the upgrade.
- I did not find the root cause of the web updater failing. I assumed the instance being far behind was the reason, which is a guess.

## Related Notes
- [Nextcloud](../services/nextcloud.md)
- [Caddy](../services/caddy.md)

## Tags
#runbook
