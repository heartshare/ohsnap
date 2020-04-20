# OHSNAP

A tool to (recursively) create ZFS snapshots and run a script for each such snapshot (e.g. to create a backup). After all snapshots were handled, they are deleted again.

This tool is especially useful to create backups with [borg](https://borgbackup.readthedocs.io/) or other tools.

## Example invocation

```shell
/mnt/tank/local/ohsnap/ohsnap -v --recursive --mount /var/run/ohsnap -- tank /mnt/tank/local/borg/backup_dataset
```

See `ohsnap --help` for details.

For this to work, you need a backup script to handle each snapshot. This script can look as follows. Of course, you can adapt it according to your specific needs.

```shell
BORG="/mnt/tank/local/borg_wrapper"

if [[ "$OHSNAP_DATASET" =~ ^tank/(\..*|backup|local)(/|$) ]]; then
  # Don't backup those datasets
  exit 0
fi

# Replace slashes in dataset name with a double-underscore
# to form the borg archive name
archive="${OHSNAP_DATASET////::}"

"$BORG" create --verbose "::${archive}--${OHSNAP_TIMESTAMP}" .
"$BORG" prune --list --prefix "${archive}--" \
  --keep-daily=8 \
  --keep-weekly=5 \
  --keep-monthly=13
```

Finally, I found it useul to use a wrapper around borg to set the local environment on each invocation. That way. you can easily mount backups and adjust things as required. I have added the wrapper script in `/mnt/tank/local/borg_wrapper`

```shell
#!/bin/sh

export BORG_CACHE_DIR="/mnt/tank/local/borg/.cache/borg"
export BORG_CONFIG_DIR="/mnt/tank/local/borg/.config/borg"

export BORG_RSH="ssh -i /mnt/tank/local/borg/.ssh/id_ed25519 -p 23 -o ChallengeResponseAuthentication=no -o PasswordAuthentication=no -o BatchMode=yes"
export BORG_REPO="u123456@u123456.your-storagebox.de:borg"

export BORG_PASSPHRASE="SET_YOUR_PASSPHRASE_HERE"
export BORG_KEY_FILE="/mnt/tank/local/borg/.config/borg/keys/u123456_your_storagebox_de__borg"

/mnt/tank/local/borg/borg-freebsd64 "$@"
````

### Restore

In FreeBSD (and FreeNAS), you need to mount the fuse kernel module first. Then, you can mount a borg archive using our wrapper

```
kldload fuse
/mnt/tank/local/borg_wrapper mount ::tank::work--2020-03-31T08:17:50 /var/run/mnt/
```
