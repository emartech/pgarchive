
# pgarchive Manual

A PostgreSQL Archive Server providing fast and incremental Backup and Restore Services


## Architecture

### Overview

`pgarchive` uses the concept of a container which stores data and meta data, and manages backup, restore and maintenance services for a single PostgreSQL cluster.

There may be several independent containers per host, and a database cluster may be backed up to containers on multiple hosts. The basic components and data flow for a single container are sketched in the following diagram. Some details like automatic expiry and btrfs filesystem maintenance are omitted for clarity.

```
   +--------------------------+
   |  Upstream PostgreSQL DB  | <-------------------------------------------+
   |      may be remote       |                                             |
   |   not part of container  | -------------+                              |
   |  has a replication slot  |              |                              |
   +--------------------------+              | pg_basebackup                |
        |                                    | on container init only       |
        | streaming replication              |                              |
        V                                    V                              |
+-------------------+                  +--------------------------+         |
| pg_receivexlog    | restore_command  | warm standby PG process  |         |
| /wal_archive      | ---------------> | /standby                 |         |
+-------------------+                  +--------------------------+         |
    ^       |                                   |                           |
    |       |                                   | btrfs snapshot            |
    v       | restore_command                   | <cron>                    |
  gzip      |          |                        V                           |
 <cron>     |  <user creates clone>    +--------------------------+         |
            |          |               | snapshot archive         |         |
            |     btrfs snapshot       | /snapshots               |         |
            |  +---------------------- | /snapshots/<timestamp>   |         |
            |  |                       | /snapshots/...           |         |
            |  |                       +--------------------------+         |
            |  |                                                            |
            V  V                                                            |
        +------------------------------------+        user does full or     |
        | user created clone                 |-+    partial DR on his own   |
        | (name, base snapshot, target time) | | ---------------------------+
        | /clones/<name>                     | |   rsync, streaming repl.,
        +------------------------------------+ |    custom dump&restore, ...
          +------------------------------------+

```

In the filesystem a container is a single directory passed via $PGARCHIVE in the environment to all `pgarchive` invocations. Pathes in the diagram above are assumed to be rooted at this directory.

There are two demon subsystems for every container. One is a [pg_receivexlog] process pulling down WAL data from the upstream DB to $PGARCHIVE/wal_archive using the streaming replication protocol. There's also an associated optional cron job which compresses WAL segments.

The second demon subsystem is a PostgreSQL "warm" standby instance in $PGARCHIVE/standby which is initially created with [pg_basebackup] from the upstream DB. Afterwards it polls the local wal_archive and applies completed WAL segments to its cluster directory, so it mirrors the upstream DB with some delay.

The snapshot subsystem is purely cron-based, it uses btrfs to create snapshots of the standby cluster directory. This is fast and space efficient, although there's some redundancy with the wal_archive. Creating more snapshots will decrease time to recovery for some storage increase. Automatic time based thinning and expiry is included in a cron job.

Finally snapshots may be "cloned" to separate directories by users. They are provisioned with a suitable [recovery.conf] automatically, and can be run instantly without manual modification. Once started they will perform recovery according to parameters in [recovery.conf], and present a usable PostgreSQL server instance some time afterwards. "Cloning" is just doing another btrfs snapshot under the hood and so is very fast, but the time needed for WAL playback will vary according to the recovery target (time) and available CPU and IO resources. Faster restore times can be traded for storage space required by more frequent snapshots. Clones are created and deleted by users only.

There is no full disaster recovery support yet, but see the Operations section for suggestions on how to do this manually. Support for scripted disaster recovery is planned for the future.


### Filesystem Layout

Each container directory must be on a btrfs filesystem, and is created as a btrfs subvolume. Each backup directory works in isolation from other ones (apart from global resources like TCP ports) and has the following files and subdirectories, some of which are btrfs subvolumes too:

<dl>
<dt>pgarchive.conf</di>
  <dd>Container settings, in shell format and sourced by pgarchive before executing commands.</dd>
<dt>logs</dt>
  <dd>Directory of container logs. Also contains standby's pg_log and pg_xlog directories -- although the latter aren't proper logs -- to avoid snapshotting them.</dd>
<dt>wal_archive</dt>
  <dd>WAL segements retrieved by streaming replication from the upstream DB.</dd>
<dt>standby</dt>
  <dd>A btrfs subvolume containing a PostgreSQL data dir for a warm-standby instance. Snapshots are created from this directory. It uses an embedded socket dir and turns off TCP so as not to interfere with any other PostgreSQL instances on localhost.</dd>
<dt>snapshots</dt>
  <dd>Each subdirectory here is a btrfs read-only snapshot of the standby directory, with a ISO 8601 timestamp as the name (always UTC). Note snapshots lack pg_xlog and pg_log directories because these are symlinks.</dd>
<dt>clones</dt>
  <dd>Each subdirectory is a write-able btrfs subvolume created by snapshotting a snapshot yet again. The names of clones is user-determined.</dd>
</dl>

### Running Services

Each running container consists of 2 process groups, one a pg_receivexlog process maintaining the wal_archive and a minimal PostgreSQL instance, like this:

```
$ ps xf
 4737 ?        S      0:01 pg_receivexlog -D /srv/backup/<NAME>/wal_archive \
                                     --slot=wal_archive_<NAME> -v -w -d  user=replication
 5542 ?        S      0:00 /usr/pgsql-9.4/bin/postgres -D /srv/backup/<NAME>/standby
 5544 ?        Ss     0:01  \_ postgres: logger process
 5545 ?        Ss     0:10  \_ postgres: startup process   recovering 00000005000000480000005C
 5549 ?        Ss     0:00  \_ postgres: checkpointer process
 5550 ?        Ss     0:00  \_ postgres: writer process
```

These 2 process groups may be started and stopped independently of each other (and even the upstream database) using the `pgarchive wal_archive` and `standby` command groups. They will catch up after extended breaks, and retry connecting to their upstream indefinitely should it be unavailable. Their logs should show any problems.

For each container 3 cron jobs are added to the crontab of the user creating the container. All are disabled (commented out) by default, but you should verify and enable all 3 usually. There are some parameters in pgarchive.conf which further control the behavior of these.

```
# *** pgarchive container /srv/backup/<NAME> ***
#*/7 * * * *        PGARCHIVE='/srv/backup/<NAME>' '/var/lib/pgsql/bin/pgarchive' cron compress-wal-archive
#01 */4 * * *       PGARCHIVE='/srv/backup/<NAME>' '/var/lib/pgsql/bin/pgarchive' cron expire-and-create-snapshot
#05 01 * * sun      PGARCHIVE='/srv/backup/<NAME>' '/var/lib/pgsql/bin/pgarchive' cron defrag-btrfs
```

## Installation

1. You must have PostgreSQL 9.4 or 9.5 (or both) installed, including client and server applications, matching the version(s) used by the database(s) to be backed up.
1. You must have [btrfs] installed, from btrfs-progs.
1. Copy `pgarchive` from this repository to `/usr/local/bin` or somewhere else in your path. Make sure it has `+x` permissions.

If you want fancy bash completion to work you should execute the following, or better yet put it in a .bashrc or equivalent file:

```
eval "$(pgarchive bash-completion)"
```

All invocations of pgarchive require $PGARCHIVE to be the path to a backup container, which must be located on a btrfs filesystem. You might want to add something like this to your .bashrc too:

```
export PGARCHIVE=/path/to/container
```

For high-load DBs at [emarsys.com] the following [btrfs mount options] work best. You do need `user_subvol_rm_allowed` for snapshot expiry to work, the rest merely improve performance for us.

```
user_subvol_rm_allowed,noatime,nobarrier,enospc_debug,nodatacow,metadata_ratio=6
```

Now you need to use `pgarchive container init`, possibly with more configuration, to create container(s). Re-starting containers on server re-boots is left up to you. (TODO: add sysV and systemd service scripts to repository)

### Security Considerations

There are no internal security mechanisms at all. Note especially:

* This is expected to be run on a trusted network. You could secure the upstream DB connections using SSL, but this is up to you.
* By default backup containers can log into their upstream PostgreSQL instance using a super user, although you could reduce this to replication only after container initialization and replication slots have been created.
* Access to containers and their data is purely host based. Data is not encrypted. Guard your pgarchive servers like the real thing.

### Monitoring Hints

Backup services need monitoring. How exactly this is done is out of scope of pgarchive and determined by local policy, but here is what we found useful at [emarsys.com], for each container:

* The wal_archive directory should have frequent activity (mtime). If your upstream DB has long idle periods setting [archive_timeout] will put a limit on this (like 1 hour). Typically the wal_archive subsystem is not running if this alert is raised, or otherwise cannot write to the wal_archive directory.

* The snapshots directory should have frequent activity too, depending on the cron job for snapshot creation. Note snapshot creation only succeeds if the standby process is active, which means running and having a recent restartpoint, so this typically flags standby process problems.

* You absolutely need to monitor the size of your upstream DB's $PGDATA/pg_xlog directory. But you already do this, don't you? If offline pgarchive containers cannot stream replication data, old WAL segments cannot be released and will fill up pg_xlog [replication slots]. You may eventually need to decide to drop the replication slot, which will break synchronization with your container.

Of course you should monitor free disk space for your containers, and other common system parameters.It's probably less useful to check for the presence of [pg_receivexlog] and PostgreSQL standby processes, because it's hard to associate them with multiple containers easily, and this may miss certain problems if subsystems become stuck without dying fully.


## Basic Operation

`pgarchive` is primarily a command line utility, but it also manages two external daemon processes for each container. It is intended to be used by the postgres user, and it contains a built in `help` command which outputs a summary of all (public) commands. See the Command Reference below.

`pgarchive` is defensively written (for a shell script) and will terminate itself on the first unexpected error using `set -e`. You have to be prepared to see and handle error messages of common shell utilities like ls, grep, gzip, and of PostgreSQL applications like pg_ctl, pg_receivexlog, pg_controldata, pg_archivecleanup, and more. `pgarchive` does its best to catch and handle many expected errors, and sometimes even gives advice about how to correct them, but many other errors just bubble through.

### Configuration

All invocations of pgarchive (except some internal ones) require PGARCHIVE to be set in your enviroment. For most commands it needs to point to an existing container directory, except for the `container init` command, which creates this directory. For convenience it is practical to set PGARCHIVE in `.bashrc` or a similar file, as described under Installation.

For per-site customization optionally a global config file is read and sourced as a shell script just before commands are executed. The location of this config file may be provided by $PGARCHIVE_CONF, and `/etc/pgarchive.conf` is tried if $PGARCHIVE_CONF is undefined. Note that containers always provide a container config file which is read after this.

By default operation does not require a global config file. Moreover, if you do provide a global config file, be careful about how modifying it may affect existing containers.

`pgarchive` internally sets LC_ALL and LANG to `en_US.UTF-8`. The reason for this is twofold: the log files of containers should not depend on the locale of the admin starting them, and `pgarchive` may depend on output of commands in this locale. You may change this using the (global) configuration file, but be warned other locales are completely untested.

### Create And Manage Containers

Containers may be created, purged, started, stopped and inspected in various ways using the `pgarchive container <CMD>` command group.

#### `container init`

New containers are created by the `init` command, which takes special extra parameters via environment. It detects the upstream PostgreSQL version automatically and expects the matching PostgreSQL tools ("Server Applications") to be found in its PATH, and refuses to work without them. It saves this PATH to a container config file for later use. So the PATH from `container init` will be remembered for a container, even when called later by cron, or when containers with different PATHes are created (e.g. containing PostgreSQL tools of a different version).

It is required that upstream's postgresql.conf enables streaming replication and connection slots. It is also assumed to log to $PGDATA/pg_log using CSV format using the default weekly rotation, although violating this should break only non-essential parts of pgarchive.

`pgarchive` is quite chatty during `init`, for example:

```
$ UPSTREAM="port=5435" pgarchive container init
2016-04-06T12:35:15Z Detected upstream is PostgreSQL version 9.5.
2016-04-06T12:35:15Z Verified PostgreSQL server applications' versions.
2016-04-06T12:35:15Z .
2016-04-06T12:35:15Z Setting up backup location /mnt/backup/t2.
Create subvolume '/mnt/backup/t2'
Create subvolume '/mnt/backup/t2/wal_archive'
Create subvolume '/mnt/backup/t2/standby'
2016-04-06T12:35:22Z .
2016-04-06T12:35:22Z Creating warm standby.
transaction log start point: 0/4B000028 on timeline 1
pg_basebackup: starting background WAL receiver
29881/29881 kB (100%), 1/1 tablespace
transaction log end point: 0/4B0000F8
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: base backup completed
2016-04-06T12:35:29Z .
2016-04-06T12:35:29Z Creating WAL archive at /mnt/backup/t2/wal_archive.
2016-04-06T12:35:29Z Creating replication slot pgarchive_t2 in upstream DB cluster.
2016-04-06T12:35:29Z starting wal_archive process pgarchive_t2
2016-04-06T12:35:30Z .
2016-04-06T12:35:30Z Creating first snapshot.
2016-04-06T12:35:30Z Creating snapshot 2016-04-06T12:35:22Z.
Create a readonly snapshot of '/mnt/backup/t2/standby' in '/mnt/backup/t2/snapshots/2016-04-06T12:35:22Z'
2016-04-06T12:35:41Z .
server starting
< 2016-04-06 14:35:43.569 CEST >LOG:  redirecting log output to logging collector process
< 2016-04-06 14:35:43.569 CEST >HINT:  Future log output will appear in directory "pg_log".
2016-04-06T12:35:50Z .
2016-04-06T12:35:50Z Adding commented out cron jobs, you need to check and enable them using 'crontab -e'.
```

To complete the container creation you need to enable the cron jobs using `crontab -e` (see Cron Jobs).

#### `container status` and `dashboard`

After `ìnit` the container is already up and running, doing its work of streaming replication data from the upstream DB to a local archive, and feeding completed WAL-segments to a standby process. This can be verified with the status command:

```
$ pgarchive container status
2016-04-04T14:40:10Z wal_archive is running (pid=25821)
pg_ctl: server is running (PID: 25780)
/usr/pgsql-9.5/bin/postgres "-D" "/srv/backup/t1/standby"
```

As promised there's also a (very simple) dashboard command to get a summary of what's going on:

```
$ pgarchive container dashboard
2016-04-06T14:24:46Z wal_archive is running (pid=20786)
pg_ctl: server is running (PID: 20796)
/usr/pgsql-9.5/bin/postgres "-D" "/mnt/backup/t1/standby"

Filesystem      Size  Used Avail Use% Mounted on
-               6.6T  5.3T  1.3T  81% /mnt/backup/t1

Processes:
  PID TTY      STAT   TIME COMMAND
20796 pts/2    S      0:00 /usr/pgsql-9.5/bin/postgres -D /mnt/backup/t1/standby
20804 ?        Ss     0:00  \_ postgres: logger process
20805 ?        Ss     0:00  \_ postgres: startup process   recovering 00000001000000000000004C
20812 ?        Ss     0:00  \_ postgres: checkpointer process
20814 ?        Ss     0:00  \_ postgres: writer process
20786 pts/2    S      0:00 pg_receivexlog -D /mnt/backup/t1/wal_archive --slot=pgarchive_t1 --verbose --no-password -d port=5435 user=replication

Replication status:
-[ RECORD 1 ]----+------------------------------   -[ RECORD 1 ]+-------------
pid              | 20787                           slot_name    | pgarchive_t1
usesysid         | 16385                           plugin       |
usename          | replication                     slot_type    | physical
application_name | pgarchive_t1                    datoid       |
client_addr      |                                 database     |
client_hostname  |                                 active       | t
client_port      | -1                              active_pid   | 20787
backend_start    | 2016-04-06 16:24:42.162962+02   xmin         |
backend_xmin     |                                 catalog_xmin |
state            | streaming                       restart_lsn  | 0/4C000000
sent_location    | 0/4C0012C0
write_location   | 0/4C000000
flush_location   | 0/4C000000
replay_location  |
sync_priority    | 0
sync_state       | async

==> /mnt/backup/t1/log/cron_snapshot.log <==
2016-04-05T12:21:08Z Expiring snapshots before 2016-03-08T12:21:08Z.
2016-04-05T12:21:08Z No WAL segments to expire.
2016-04-05T12:21:08Z Thinning snapshots to 1/day before 2016-03-22T12:21:08Z.
2016-04-05T12:21:09Z Creating snapshot 2016-04-05T12:19:52Z.
Create a readonly snapshot of '/mnt/backup/t1/standby' in '/mnt/backup/t1/snapshots/2016-04-05T12:19:52Z'

==> /mnt/backup/t1/log/wal_archive.log <==
pg_receivexlog: disconnected; waiting 5 seconds to try again
pg_receivexlog: starting log streaming at 0/4A000000 (timeline 1)
pg_receivexlog: finished segment at 0/4B000000 (timeline 1)
pg_receivexlog: finished segment at 0/4C000000 (timeline 1)
pg_receivexlog: starting log streaming at 0/4C000000 (timeline 1)

==> /mnt/backup/t1/log/standby/postgresql-Wed.csv <==
2016-04-06 16:24:43.675 CEST,,,20805,,57051c2b.5145,2,,2016-04-06 16:24:43 CEST,,0,LOG,00000,"entering standby mode",,,,,,,,,""
2016-04-06 16:24:43.824 CEST,,,20805,,57051c2b.5145,3,,2016-04-06 16:24:43 CEST,,0,LOG,00000,"restored log file ""00000001000000000000004B"" from archive",,,,,,,,,""
2016-04-06 16:24:44.459 CEST,,,20805,,57051c2b.5145,4,,2016-04-06 16:24:43 CEST,,0,LOG,00000,"redo starts at 0/4B000028",,,,,,,,,""
2016-04-06 16:24:44.459 CEST,,,20805,,57051c2b.5145,5,,2016-04-06 16:24:43 CEST,,0,LOG,00000,"consistent recovery state reached at 0/4C000000",,,,,,,,,""
2016-04-06 16:24:44.829 CEST,,,20805,,57051c2b.5145,6,,2016-04-06 16:24:43 CEST,,0,LOG,00000,"unexpected pageaddr 0/44000000 in log segment 00000001000000000000004C, offset 0",,,,,,,,,""
```

#### `container start` and `stop`

The `start` and `stop` commands start and stop both of the wal_archive and the standby demon subsystems.

However to restart containers on server reboots you have to create your own upstart or systemd service depending on local policies. Make sure pgarchive is executed running as the correct user (postgres), and with PGARCHIVE set to the correct container directory. You may run an arbitrary amount of containers on a single host, resources permitting.

#### `container retire` and `resync`

A container may be retired using `container retire`. This breaks the synchronization with the upstream database by removing its replication slot and stopping wal streaming. It also completely deletes the standby directory. A retired container creates no new snapshots, but it still can be used to restore snapshots to clones. Also the expiry cron job may be left in place. This functionality is intended to gracefully decomission containers, e.g. after a major version upgrade of the upstream database, or to disable them for longer than you would like to hold the replication slot.

It is possible to resync a retired container using `container resync`. This will re-create the standby using pg_basebackup. Resyncing is conceptually not very different from `container init`, but allows to keep existing config and old and new history in one place. A resynced container will most likely contain a gap in its wal_archive, and the new standby and snapshot directories will not be de-duplicated with old ones. A future version may do this more efficiently using rsync and the low level backup protocol.

### The wal_archive Subsystem

This subsystem is a wrapper around pg_receivexlog streaming data from the upstream database to the wal_archive subdiretctory. There is also a cron job for data compression using gzip (See Cron Jobs).

It may be started and stopped individually, but usually there is no need to since it is automatically started and stopped by the respective `container` commands. Status and log file viewing commands may be useful in case of problems.

The `wal_archive du` command gives an overview of disk space used per day.

### The standby Subsystem

This subsystem is wraps a minimal PostgreSQL standby instance, which is fed completed WAL segments from the wal_archive via `restore_command`.

It may be started and stopped individually, but usually there is no need to since it is automatically started and stopped by the respective `container` commands. Status and log file viewing commands may be useful in case of problems.

### Manage Snapshots

#### `snapshot list`

Snapshots are automatically named after their latest contained checkpoint time, formatted in a fixed way as ISO timestamp in UTC. Available snapshots can be listed using the `snapshot list` command. After `container init` there is already a first initial snapshot:

```
$ pgarchive snapshot list
2016-04-04T16:31:30Z
```

#### `snapshot create` and `delete`

Snapshots may be created and deleted explicitly with the `snapshot create` and `snapshot delete` commands. Due to the naming policy some time needs to pass (and there needs to be some activity in the upstream DB) to create a new snapshot:

```
$ pgarchive snapshot create
2016-04-04T16:35:10Z The snapshot 2016-04-04T16:01:01Z already exists.
...
$ pgarchive snapshot create
2016-04-04T16:45:44Z Creating snapshot 2016-04-04T16:45:31Z.
Create a readonly snapshot of '/mnt/backup/t2/standby' in '/mnt/backup/t2/snapshots/2016-04-04T16:45:31Z'
```

#### `snapshot thin` and `expire`

There is support for time based thinning and expiry with the `snapshot thin` and `snapshot expire` commands. Thinning reduces the number of snapshots per day to one for snapshots before the cutoff time, while expiry removes all snapshots before the cutoff time and also removes all related WAL segments from the wal_archive. The container's `pgarchive.conf` file holds the properties `expire_date` and `thin_daily_date` which control cutoff times. To debug this it may be useful to pass `--show` to the commands, and to override the config file values by setting `THIN_DAILY_DATE` and `EXPIRE_DATE` in the environment.

Usually you do not need to manually create, delete or even expire snapshots, as a cron job takes care of all of this. See Cron Jobs below for details.

### Restoring Data

Restoring data is done by cloning a snapshot (inspired by zfs' terminology). Clones are named explicitly by the user creating them.

#### `clone create`, `duplicate` and `delete`

The `create` command takes the new clone name, a snapshot timestamp, and optionally a timestamp for PITR. It creates the new clone in the clones directory, and sets up postgresql.conf and recovery.conf, but does not yet start recovery to give the admin a chance to modify these files manually. New clones are assigned a new unused port somewhere above 54000.

Cloning is exceptionally fast, because contrary to most other backup solutions `pgarchive` can create a shallow copy-on-write copy of an existing snapshot, and only needs to run minimal recovery on startup. Frequent snapshots are cheap due to btrfs' cow, so we don't have to rely on doing large stretches of PITR.

The `duplicate` command creates a (copy-on-write) copy of an existing clone.

The `delete` command completely removes a (stopped) clone again. Old clones which are not deleted may lead to increased disk usage, or at least prevent freeing space when related snapshots are expired.

#### `clone list`

The `list` command lists created clones. With parameter `--status` it also lists which clones are running, and what port they are using.

#### `clone start` and `stop`

These commands start and stop clones. The first time a clone is started it will do recovery according to `recovery.conf`, and if successful this file will be renamed to `recovery.done`.

#### `clone log`

This will open today's postgres log file of a named clone in $PAGER, or `tail -f` it to stdout if `-f`is given as parameter. This assumes CSV log format, log storage in $PGDATA/pg_log, and the default daily log rotation with name of weekday in the filename as `postgresql-<DAY>.csv`.

#### `clone psql`

This convenience command opens a `psql` shell to a named clone, without having to provide command line parameters for its port.

#### Disaster Recovery

The clone directory has to be on btrfs, and it is not recommended to run clones as production replacements directly due to btrfs' bad performance characteristics, especially its widely volatile IO latency. Clones are useful to investigate the origin of an issue, or to extract partial data. Depending on your demands it may be possible to run your application on a clone temporily with reduced performance, or with essential services only.

For full disaster recovery a clone needs to be copied back to a production server. The two easiest options probably are using `rsync` or `pg_basebackup` for this task, but at the moment `pgarchive` provides no further support for this. A more sophisticated approach might involve running essential services on the clone while setting up streaming replication back to a production machine, and doing a switch-over eventually.

### Cron Jobs

As reported by `container init` three cron jobs are created for each container, but initially disabled. You need to verify and enable all three usually, because without them no further snapshots will be created, nor any WAL data expired:

```
# *** pgarchive container /srv/backup/t1 ***
#*/5 * * * *            PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive' cron compress-wal-archive
#01 */4 * * *           PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive' cron expire-and-create-snapshot
#05 01 * * sun          PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive' cron maintenance
```

You may adapt runtimes to you requirements. The first job rarely needs changing, unless you want to disable compression alltogether.

The second job's runtime affects disk usage by how often snapshots are created, but keep in mind that frequent snapshots also reduce PITR time. There are more options regarding expiry in the `pgarchive.conf` config file. Expiry removes both snapshots and segments from the WAL archive.

Since `btrfs` is ill prepared to deal with the large number of random updates to table and index files done by the standby process, you will need to enable the third job too or performance will degrade over time. But it may cause quite some extra IO for considerable time, so it's best to do this during off-times, and not too often. You may experiment and monitor system performance to tune this. A tell-tale sign for btrfs fragmentation issues are climbing CPU utilization and increasing IO latency spikes. If you are already doing btrfs maintenance in a different way, and it is aggressive enough, you should leave this one disabled.

For the last job to work you will need a sudo rule allowing postgres to run defragmentation as root without using a password. Add the following to your sudoers file:

```
# allow postgres
postgres        ALL = (root) NOPASSWD: /sbin/btrfs filesystem defrag *
```

If you have overridden the fs_* functions the third job calls fs_maintenance() and appends its output to log/cron_maintenance.log. It uses flock (1) to guard against concurrent calls.

### Upgrading the Upstream DB

At the moment (other than patch level) upgrades of the upstream DB cannot be followed, and this restriction is likely to remain. Supporting different versions inside a single container just adds too much complexity. You *cannot* just update `upstream_version` and `path` in the pgarchive.conf file, this will fail badly and may damage your container.

However it is easy to `retire` a container and create a new one. Make sure to have the correct (new) PostgreSQL tools in your PATH (pgarchive tries to check and complains about wrong versions). There is no data deduplication between containers so this may increase space usage, plan and monitor for space accordingly.

Note that restoring snapshots in your old retired container still works as long as you keep your old PostgreSQL version's bindir around. You may also continue to run its `expire-and-create-snapshot` cron job,  it just won't create new snapshots. When there's no need for the container any more  simply `purge` it. You may even rename your old container, but make sure to adapt the cron jobs referencing it if you do so.

### Upgrading `pgarchive` itself

First a promise: upgrading `pgarchive` will never require creating new containers, except for new major version numbers. In the future there may be automatic upgrade support. But for now this procedure is necessary:

1. Stop all containers on a single host.
1. Replace the old `pgarchive` version with the new one.
1. Manually modify containers as instructed by the section below.
1. Re-start all containers.
1. Verify.

#### 0.3.x to 0.4.0

The name of the cron job `defrag-btrfs` changed to `maintenance`, however the old name is still accepted so no changes are necessary. It will be removed completely with version 1.0.0.

#### 0.2.x to 0.3.0

The pgarchive.conf files needs 3 adaptions:

1. Add a line giving the upstream DB's version like this:

   ```
   upstream_version='9.4'
   ```

1. Add a line assigning the value of PATH to `path`:

   ```
   # path to all used applications, must contain PostgreSQL server applications of the same
   # version as the upstream database, btrfs, and commonly used shell utilities
   path='/usr/pgsql-9.4/bin:/usr/local/bin:/usr/bin:/bin:/sbin'
   ```

1. Remove the line starting with `next_clone_port=` from pgarchive.conf, it is not needed any more.

#### before 0.2.x

Sorry, you are on your own. No one else is using this.


## Command Reference (`pgarchive help`)

```
PGARCHIVE=<dir> pgarchive <cmd> <arg...>

Commands and sub-commands:

help                Show this help.

container init      Create a new container and upstream replication slot, both of which
                    must not exist yet. This requires additional environment variables to be set,
                    see below. The new backup container is fully started and operational at the end,
                    except you need to verify and enable the cron jobs (crontab -e).
container purge [--force]
                    Completely remove the full backup location and the replication slot.
                    Tries to read PGARCHIVE's config, and falls back to the same
                    extra environment parameters init uses. Make sure nothing at all is needed
                    any more, and everything is stopped.
container start     Start backup container, which starts both wal_archive and standby processes.
container stop      Stop backup container.
container status    Show status of wal_archive and standby subsystems, exit 1 if any is not running.
container dashboard Show detailed status and tails of all log files.
container retire [--force]
                    Disconnect from upstream DB, but keep data for restores.
                    Releases the replication slot, and completely removes the standby subvolume.
                    With --force errors are ignored, useful e.g. if the slot is gone already.
container resync    Re-connect a retired container by creating a connection slot and doing a fresh
                    pg_basebackup. The old standby configuration is preserved.
container initcron  Add default cron entries to the calling user's crontab, all jobs disabled.
container checkversions
                    Check versions of upstream DB and local PostgreSQL tools.
container upstream  Open a psql shell to upstream database, additional arguments are passed through.

wal_archive start   Start the wal_archive process. This process connects to the upstream DB
                    using the PostgreSQL's replication protocol, with the configured
                    replication slot, and stores retrieved WAL segments in the wal_archive
                    directory. Implemented by using pg_receivexlog.
wal_archive stop    Stop the wal_archive process.
wal_archive status  Check if the wal_archive process is running.
wal_archive list    List wal_archive directory and size.
wal_archive du      Show disk usage per day and total (by segment mtime).
wal_archive log [-f]
                    Write wal_archive.log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.

standby (start|stop|status|reload|restart) [arg] ...
                    The standby process is a warm standby PostgreSQL instance which pulls
                    completed segments from the wal_archive directory. This is a simple frontend
                    to control it using pg_ctl, and extra arguments are passed through.
standby log [-f]    Write today's standby log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.
                    Assumes the standard weekly log rotation and CSV logging in standby/pg_log.

snapshot create [--force]
                    Create a new btrfs snapshot of the standby directory under snapshots/.
                    With --force do this even when the standby process is not running,
                    and or if there's no recent restartpoint.
snapshot delete <pattern> ...
                    Directly delete one or more snapshots, may use a simple globbing pattern.
snapshot expire [--show]
                    Delete snapshots older than $expire_date, which should be defined in the
                    config file. Passing $EXPIRE_DATE in the environment may be used to
                    override the config file. This also removes WAL segments which are not
                    needed any more from the wal_archive.
                    The format of the date must be understood by date (1) --date=STRING,
                    and may be a relative term like "1 week ago".
                    With --show only show the actions to be taken, but don't delete anything.
snapshot thin [--show]
                    Delete snapshots older than $thin_daily_date which are not the first snapshot
                    of a day. This allows to thin snapshot density for older data. Like expire
                    this supports --show and overriding the config file with $THIN_DAILY_DATE.
snapshot list [<pattern>]
                    List snapshots, optionally restricted to a simple globbing pattern.

clone create <name> <snapshot> [<target-time>]
                    Create a new named (PostgreSQL) clone instance from a snapshot.
                    A new port is assigned and appended to postgresql.conf.
                    The clone will have a recovery.conf suitable to do recovery to
                    either the next consistent point in time, or an arbitrary target time
                    (which should be after snapshot creation time). The clone is not
                    started automatically, so you may edit *.conf files before
                    initiating recovery. Recovery success depends on the wal_archive
                    still containing all required WAL segments.
clone duplicate <source> <target>
                    Duplicates an existing clone instance. Note that if source is running
                    target will have to do crash recovery once started. A new port is
                    assigned and appended to postgresql.conf automatically.
clone delete <name> Delete a (stopped) clone.
clone (start|stop|status|reload|restart) <name> [arg] ...
                    This is a simple frontend for pg_ctl to start and stop named clones.
                    The stop command also accepts --all instead of <name>.
clone list [--status] [<pattern>]
                    List clones, optionally restricted to a simple globbing pattern.
                    With --status the assigned port and pg_ctl status is given for each.
clone log <name> [-f]
                    Write the named clone's log to stdout or tail -f it.
                    If stdout is a terminal $PAGER is used to show it.
                    Assumes the standard weekly log rotation and csv logging.
clone psql <name> [arg] ...
                    Open a psql shell connected to a running named clone instance.
                    Additional arguments are passed through.

cron compress-wal-archive
                    Compress completed segments in wal_archive, which have already been
                    processed by the standby process. Logs to log/cron_wal_archive.log.
cron expire-and-create-snapshot
                    Expire old snapshots and wal segments, thin snapshots, and create a new one.
                    The log is written to log/cron_snapshot.log.
cron maintenance    Defragment the standby directory, since PostgreSQL tends to cause a critical
                    amount of btrfs fragmentation fast. Requires password-less sudo privileges to
                    call "/sbin/btrfs filesystem defrag ...". Logs to log/cron_maintenance.log.

bash-completion     Output a bash completion snippet for this utility.
                    This may be saved to global or personal profiles, or eval'ed directly:
                        eval "$(pgarchive bash-completion)"

Environment:

PGARCHIVE           Required base path to a backup container, which must be on btrfs.
PGARCHIVE_CONF      Optional path to a global config file.

        The following environment parameters are only used by the 'container init' and 'purge'
        commands, and written to the $PGARCHIVE/pgarchive.conf container config file on 'init':

PATH                The PATH used internally. This must contain PostgreSQL's server applications
                    (pg_ctl, pg_basebackup, ...) of the same version as the DB cluster a container
                    is created for.
SLOT                Create and use this replication slot at upstream. Defaults to
                    "pgarchive_<basename of $PGARCHIVE>".
UPSTREAM            A connection info string used when accessing the upstream DB in
                    command mode, for example when creating a SLOT or checking status. See
                    http://www.postgresql.org/docs/9.4/static/libpq-connect.html#LIBPQ-CONNSTRING.
                    The default value is "" (local connection, default port, user, DB etc.)
                    This is persisted to the config file and may be edited there later.
                    Note that common libpq environment settings (PG*) which are active in the
                    caller's shell are explicitly cleared by pgarchive. This must work without
                    a password prompt, and the DB user must have sufficient privileges to
                    create and drop a connection slot for the init, purge, retire, resync commands.
UPSTREAM_REPL       The connection info string the WAL archive process (pg_receivexlog) uses to
                    stream new WAL segments. Defaults to UPSTREAM. All of UPSTREAM's caveats apply.

Configuration files:

$PGARCHIVE_CONF     If defined this is sourced as shell script after all internal variables and
                    functions have been defined, but before commands are executed. This could be
                    used to override internal variables and functions, at your own risk.
/etc/pgarchive.conf This is only read if $PGARCHIVE_CONF is not defined, in the same fashion
                    as $PGARCHIVE_CONF.
$PGARCHIVE/pgarchive.conf
                    This per-container config file is always read after the global one.
                    It contains container specific settings, like DB connection settings,
                    the PATH to use, and some settings regarding expiry and cron jobs.
```

[archive_timeout]: http://www.postgresql.org/docs/9.4/static/runtime-config-wal.html#GUC-ARCHIVE-TIMEOUT
[btrfs mount options]: https://btrfs.wiki.kernel.org/index.php/Mount_options
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[emarsys.com]: http://emarsys.com/
[pg_basebackup]: http://www.postgresql.org/docs/9.4/static/app-pgbasebackup.html
[pg_receivexlog]: http://www.postgresql.org/docs/9.4/static/app-pgreceivexlog.html
[recovery.conf]: http://www.postgresql.org/docs/9.4/static/recovery-config.html
[replication slots]: http://www.postgresql.org/docs/9.4/static/warm-standby.html#STREAMING-REPLICATION-SLOTS

