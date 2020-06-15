# Borg s3 home backup

## Introduction

This script will backup a specified directory using borg. The backup will be both compressed and encrypted.
Since borg does not natively support s3, and the fuse/mount solution is slow as heck, we're using aws
sync instead at the end. This actually works pretty well.

This also prunes your backups to keep 7 dailys and 4 end-of-weeks.

The side effect however is you'll have two backups, one wherever borg has its backup repo, and another
on s3. This is good though, always a good idea to keep several backups even if one is on the same
computer.

## Compatibility

  * Linux for sure.
  * MacOS likely.

## Requirements

### Tools

  * [Borg backup.](https://www.borgbackup.org/)
  * awscli - you must configure a profile with aws access key and secret for use on the command line.
  * An s3 bucket the above profile has access to.

### Environment variables

These are borg-standard, as per [borg's documentation](https://borgbackup.readthedocs.io/en/stable/usage.html#environment-variables):

  * **BORG_REPO** (mandatory): location where all your backups will go into as they're made (NOT s3).
  Whereas borg supports ssh paths here as well as any mounted folder (eg s3 via fuse), I would recommend
  this to be a local folder on a real drive, if you can afford the space, for speed.
  * **BORG_PASSPHRASE** (optional, recommended): borg will encrypt your backups using this passphrase. You should.
  Make sure you keep this somewhere safe, other than your backup as you'll also need it to restore your files.

These are required by the script to function:

  * **BORG_S3_BACKUP_ORIGIN**: put in here the directory you want to back up.
  * **BORG_S3_BACKUP_BUCKET**: put in here the bucket name only.
  * **BORG_S3_BACKUP_AWS_PROFILE**: put in here the aws cli profile that has access to that bucket (eg `default`).

This is optional:

  * **BORG_S3_BACKUP_EXCLUDE**: put in here the path to a exclude patterns. Refer to borg-create(1) for the specification.

## How to use

  * Git clone this repo somewhere in your computer.
  * Install borg backup according to your platform. Possibly already on your distro's software repositories.
  * Install awscli - same as above.
  * AWS setup:
    * You obviously need an account in there.
    * You must have access keys and secrets for it.
    * Configure aws cli with these credentials, eg `aws configure`.
    * Make yourself a bucket in your aws account.
  * Set up the environment variables discussed above. I do recommend you also set `BORG_PASSPHRASE`.
  * Optionally, create an exclude file.
  * Create a borg repo: `borg init`
  * Run [borg-s3-backup.sh](borg-s3-backup.sh)!

### Systemd timer

Example timer unit that runs the backup every day at 19:00:

```
[Unit]
Description=Back up to S3 daily

[Timer]
OnCalendar=*-*-* 19:00
Persistent=true

[Install]
WantedBy=timers.target
```

Corresponding service unit, assuming you added your environment variables into `~/.config/borg-s3-backup/config`
and installed the backup script in `~/.local/bin`:

```
[Unit]
Description=Back up to S3

[Service]
Type=oneshot
EnvironmentFile=%h/.config/borg-s3-backup/config
ExecStart=/bin/bash %h/.local/bin/borg-s3-backup.sh

[Install]
WantedBy=default.target
```

### Note: borg backup locking

We lock the borg backup repository during aws s3 sync to ensure it doesn't change during uploads. Borg achieves locks using lock files within the repo,
therefore these files will also be backed up to s3 during sync. If you ever need to download your backup from s3 it will thus be locked and will need
unlocking with `borg break-lock`.

## Restoring backups

There's no script here to restore your backups, you'll have to use borg for that.
See [borg's documentation](https://borgbackup.readthedocs.io/en/stable/usage.html#borg-extract). Generally:

  * Make sure the environment variables above are all set.
  * Download from s3 all your backup files into the location at $BORG_REPO (if you don't have your local borg repo).
  * Run `borg break-lock` to unlock your backup repo. See [note above](#note-borg-backup-locking) for explanation on why your backup is locked.
  * `borg list` will show you available backups
  * CD into some folder then `borg extract ::backup-name`, should extract on that same folder.
  * Move extracted files where they're meant to be.

Example for a typical desktop computer - total restore of your home folder:
  * Make an administrative user whichever way you'd like, make sure they can `sudo` (for instance, on Arch
  they must be in the `wheel` group). You'll be using this user to restore your data. Do this even if your user
  is already sudo-able to avoid issues when running the commands below.
  * Log out of your desktop back to the log in screen.
  * `CTRL+ALT+F1` to switch to tty1.
  * Log in as said user.

Then:

```bash
# Move your current home folder out of the way
cd /home
mv myusername myusername-old

# Make yourself a new one belonging to you
sudo mkdir myusername
sudo chown myusername:myusername myusername

# Work out available backups and extract!
borg list
cd /
borg extract ::whichever-backup-you-need-maybe-latest
reboot
```
