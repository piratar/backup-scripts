# Privacy preserving backup scripts

The Icelandic pirate party runs two privacy sensitive web-apps; wasa2il
(policies and elections) and icepirate (the membership roster). Both of
these apps contain sensitive data in their databases which we have an
obligation to protect and prevent from leakage.

In particular, icepirate contains PII about members of the Icelandic
Pirate Party and its affiliates, while wasa2il contains
user-identifiable ballots while elections are in process and have not
been finalized.

As all this data is very important, we need good automated backups.

However, we don't want the backups to unduly weaken peoples' privacy or
reduce the anonymity of elections. These scripts therefore implement a
backup policy where the local admin only has access to the latest 24
hours worth of backups, but a "write only" historic copy is encrypted
and stored off-site.


## Backup goals

1. Protect against administrator error
2. Protect against developer error
3. Protect against hardware failures
4. Protect user privacy and election privacy


## Backup strategy

1. Create MySQL/PostgreSQL dumps and tar archives in `~/backups.insecure/latest`.
2. Encrypt these dumps with the public PGP key of chosen trusted
   parties and store in `~/backups/dayN` and `~/backups/wkM`.
3. Synchronize the `~/backups/` folder to remote storage using rsync and
   send an email to admins indicating success or failure.
4. Run backups once per day (via cron), or on-demand before upgrades

### Details & rationale

The first step satisfies goals #1 and #2, in that an admin performing an
upgrade or other system operation can trigger a backup which they can
then use to roll-back if the upgrade causes problems.

The second step satisfies goal #4, protecting user and election privacy
by only keeping historic backups that have been encrypted. Ideally the
PGP keys in use should not be accessible to the administrators or
developers of the system.

The folders created in step 2 represent days of the week and week-of-the
year respectively, guaranteeing that only a finite number of backups
will ever be created (52 + 7), while providing a higher resolution of
backups for recent states.

Step three protects us against hardware failure of member.piratar.is,
satisfying goal #3.

Step four makes sure all this doesn't get forgotten.


## Configuration

An example `~/ppbackup.cfg` file might look like this:

    # This is the configuration for ~/bin/backup-scripts/ppbackup
    #
    BACKUP_NAME=myapp
    BACKUP_REMOTE=backupuser@example.com:backup-folder
    BACKUP_FILES="myapp logs tools"
    BACKUP_BINARIES="virtualenv"
    BACKUP_MYSQL_DB="myappdb"
    # or: BACKUP_PSQL_DB="myappdb"
    BACKUP_GPG_RECIPIENT="trustedperson@example.com"
    BACKUP_ADMIN_EMAILS="admin-one@example.com,admin-two@example.com"

    # This optional setting means that only one day of backups should
    # be retained. Appropriate for large backups that do not require
    # backups far into the past.
    #ONLY_ONE_DAY=1

    # Optional limit to how large backups are allowed to become. The
    # latest backup is not included in this limit, to guarantee at
    # least one backup will always be made.
    #BACKUP_LIMIT_MB=1024

Currently most variables have to be set for the script to run except for
the SQL database variables, they are optional as most people will only
be using one or the other (or neither). Note that the `BACKUP_BINARIES`
is optional and will only be preserved in daily backups, not in the more
long-lived weeklies.

This configuration relies on the following also being configured:

   1. `~/.my.cnf` or `~/.pgpass` must have credentials for `myappdb`
   2. The user's GnuPG key-chain must have a trusted key public key
      for `trustedperson@example.com`.
   3. The user must have a public SSH key which is in `~/.ssh/authorized_keys`
      on the `backupuser@example.com` account.
   4. The 'mailx' program must be installed for administrators to be
      notified about success or failure.

That's all, folks!
