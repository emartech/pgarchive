
# pgarchive Manual

An Archive service providing fast and incremental Backup and Restore for PostgreSQL 9.4+


## Architecture

## Overview

`pgarchive` uses the concept of having a container to both store data and manage services for a PostgreSQL cluster. There may be several independant containers per host, and a database cluster may be backed up to serveral containers. The basic components and data flow for a single container are sketched in the following diagram. Some details like automatic expiry and btrfs filesystem maintenance are omitted for clarity.

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
            |                                   |                           |
            |                                   | btrfs snapshot            |
            | restore_command                   | <cron>                    |
            |          |                        V                           |
            |  <user requests clone>   +--------------------------+         |
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

There are two demon subsystems for every container. One is a [pg_receivexlog] process pulling down WAL data from the upstream DB to $PGARCHIVE/wal_archive using the streaming replication protocol. There's also an optional cron job which compresses old WAL segments.

The second demon subsystem is a PostgreSQL "warm" standby instance in $PGARCHIVE/standby which is initially created with [pg_basebackup] from the upstream DB. Afterwards it polls the local wal_archive and applies completed WAL segments to its cluster directory, so it mirrors the upstream DB with some lag.

The snapshot subsystem is purely cron-based, it uses btrfs to create snapshots of the standby cluster directory. This is fast and space efficient, although there's some redundancy with the wal_archive. Creating more snapshots will decrease time to recovery for some storage increase. Automatic time based thinning and expiry is included in a cron job.

Finally snapshots may be "cloned" to separate directories by users. They are provisioned with a suitable [recovery.conf] automatically, and can be run instantly without manual modification. Once started they will perform recovery according to parameters in [recovery.conf], and present a usable PostgreSQL server instance some time afterwards. "Cloning" is just doing another btrfs snapshot under the hood and so is very fast, but the time needed for WAL playback will vary according to the recovery target (time) and available CPU and IO resources. Faster restore times can be traded for storage space required by more frequent snapshots. Clones are created and deleted by users only.

Note that there is no full disaster recovery support yet. The user may rsync a clone back to a production server on his own, do fancy stuff with streaming replication, or whatever works best in his environment. A knowledgeable user could also copy a snapshot to some replacement server directly, connect remotely to the container's wal_archive, and thereby skip creating a local clone. It is not recommended to run a local clone as full production replacement (at least not for a longer than strictly necessary) because btrfs isn't up to this yet. Support for more scripted recovery options are planned for the future.


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

1. You must have PostgreSQL 9.4 or 9.5 installed, including its client and server applications.
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

`pgarchive` is a command line utility. It is intended to be used by the postgres user, and it contains in-built `help` command which outputs a summary of all (public) commands. See the Command Reference below.

All invocations of pgarchive (except internal ones) require PGARCHIVE to be set in your enviroment. For most commands it needs to point to an existing container directory, except for the `container init` command, for which it must not exist yet. For convenience it is practical to set PGARCHIVE in `.bashrc` or a similar file, as described under Installation.


`pgarchive` is a defensively written shell script which will terminate itself on the first unexpected error (using `set -e`). You have to be prepared to see and handle error messages of all underlying programs. `pgarchive` does its best to catch and handle errors wherever it is reasonable, and sometimes even gives advice how to correct them, but handling all edge cases on arbitrary user systems is clearly out of scope and would contradict the simplicity goal.

### Create And Manage Containers

Containers are created, purged, started, stopped and inspected in various ways using the `pgarchive container <CMD>` command group.

New containers are created by the `init` command, which takes special extra parameters via environment. Of special note, it detects the upstream PostgreSQL version automatically and expects the matching PostgreSQL tools ("Server Applications") to be found in its PATH, and will refuse to work without them. It will save this PATH to a container config file for later use. So the PATH from `container init` time will be remembered per container, even when called later by cron, or when containers with different PATHes are created (e.g. containing PostgreSQL tools of a different version).

`pgarchive` tends to be quite chatty during this, for example:

```
$ export PGARCHIVE=/srv/backup/t2
$ UPSTREAM="port=5435" pgarchive container init
2016-04-04T14:25:35Z Detected upstream is PostgreSQL version 9.5.
2016-04-04T14:25:35Z Verified PostgreSQL server applications' versions.
2016-04-04T14:25:35Z .
2016-04-04T14:25:35Z Setting up backup location /srv/backup/t2...
Create subvolume '/srv/backup/t2'
Create subvolume '/srv/backup/t2/wal_archive'
Create subvolume '/srv/backup/t2/standby'
2016-04-04T14:25:44Z .
2016-04-04T14:25:44Z Creating WAL archive at /srv/backup/t2/wal_archive...
 pg_create_physical_replication_slot
-------------------------------------
 (pgarchive_t2,)
(1 row)

2016-04-04T14:25:44Z starting wal_archive process pgarchive_t2
2016-04-04T14:25:47Z .
2016-04-04T14:25:47Z Creating warm standby...
transaction log start point: 0/17000028 on timeline 1
pg_basebackup: starting background WAL receiver
29788/29788 kB (100%), 1/1 tablespace
transaction log end point: 0/170000F8
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: base backup completed
2016-04-04T14:25:51Z .
2016-04-04T14:25:51Z Copying missing segments to wal_archive.
2016-04-04T14:25:51Z Creating snapshot 2016-04-04T14:25:47Z.
Create a readonly snapshot of '/srv/backup/t2/standby' in '/srv/backup/t2/snapshots/2016-04-04T14:25:47Z'
2016-04-04T14:25:56Z .
server starting
< 2016-04-04 16:25:59.105 CEST >LOG:  redirecting log output to logging collector process
< 2016-04-04 16:25:59.105 CEST >HINT:  Future log output will appear in directory "pg_log".
2016-04-04T14:26:10Z .
2016-04-04T14:26:10Z Adding commented out cron jobs, you need to check and enable them using 'crontab -e'.
```

After this the container is already up and running, doing its work of streaming replication data from the upstream DB to a local archive, and feeding completed WAL-segments to a standby process. This can be verified with the status command:

```
$ pgarchive container status
2016-04-04T14:40:10Z wal_archive is running (pid=25821)
pg_ctl: server is running (PID: 25780)
/usr/pgsql-9.5/bin/postgres "-D" "/srv/backup/t1/standby"
```

To complete the container creation you also need to enable the cron jobs (see Cron Jobs).

There are `start` and `stop` commands which do the expected. As promised there's also a (very simple) dashboard command to get a summary of what's going on:

```
$ pgarchive container dashboard
2016-04-04T16:27:51Z wal_archive is running (pid=25821)
pg_ctl: server is running (PID: 25780)
/usr/pgsql-9.5/bin/postgres "-D" "/mnt/backup/t2/standby"

Filesystem      Size  Used Avail Use% Mounted on
-               6.6T  5.2T  1.4T  80% /mnt/backup/t2

Replication status at upstream:
  pid  | usesysid |   usename   | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_location | write_location | flush_location | replay_location | sync_priority | sync_state
-------+----------+-------------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+---------------+----------------+----------------+-----------------+---------------+------------
 25823 |    16385 | replication | pgarchive_t2     |             |                 |          -1 | 2016-04-04 14:32:33.616647+02 |              | streaming | 0/1A0004C0    | 0/1A0004C0     | 0/1A000000     |                 |             0 | async
(1 row)

==> /mnt/backup/t2/log/cron_snapshot.log <==
2016-04-04T12:48:52Z *** expire-and-create-snapshot ***
2016-04-04T12:48:52Z Expiring snapshots before 2016-03-07T12:48:52Z.
2016-04-04T12:48:52Z No WAL segments to expire.
2016-04-04T12:48:52Z Thinning snapshots to 1/day before 2016-03-21T12:48:52Z.
2016-04-04T12:48:52Z The snapshot 2016-04-04T11:49:11Z already exists.

==> /mnt/backup/t2/log/wal_archive.log <==
pg_receivexlog: starting log streaming at 0/16000000 (timeline 1)
pg_receivexlog: finished segment at 0/17000000 (timeline 1)
pg_receivexlog: finished segment at 0/18000000 (timeline 1)
pg_receivexlog: finished segment at 0/19000000 (timeline 1)
pg_receivexlog: finished segment at 0/1A000000 (timeline 1)

==> /mnt/backup/t2/log/standby/postgresql-Mon.csv <==
2016-04-04 16:25:56.435 CEST,,,25783,,57025edc.64b7,8,,2016-04-04 14:32:28 CEST,,0,LOG,00000,"restored log file ""000000010000000000000017"" from archive",,,,,,,,,""
2016-04-04 16:26:05.765 CEST,,,25783,,57025edc.64b7,9,,2016-04-04 14:32:28 CEST,,0,LOG,00000,"unexpected pageaddr 0/14000000 in log segment 000000010000000000000018, offset 0",,,,,,,,,""
2016-04-04 18:01:07.197 CEST,,,25783,,57025edc.64b7,10,,2016-04-04 14:32:28 CEST,,0,LOG,00000,"restored log file ""000000010000000000000018"" from archive",,,,,,,,,""
2016-04-04 18:01:07.968 CEST,,,25783,,57025edc.64b7,11,,2016-04-04 14:32:28 CEST,,0,LOG,00000,"restored log file ""000000010000000000000019"" from archive",,,,,,,,,""
2016-04-04 18:01:11.967 CEST,,,25783,,57025edc.64b7,12,,2016-04-04 14:32:28 CEST,,0,LOG,00000,"unexpected pageaddr 0/15000000 in log segment 00000001000000000000001A, offset 0",,,,,,,,,""
```

If you want to have containers to start on server reboots you have to create your own upstart script or systemd service file. Make sure pgarchive is executed running as the correct user (most likely postgres), and with PGARCHIVE set to the correct container directory. You may run an arbitrary amount of containers on a single host, resources permitting.


### The wal_archive Subsystem

TODO

### The standby Subsystem

TODO

### Manage Snapshots

Snapshots are automatically named after their latest contained checkpoint time, formatted in a fixed way as ISO timestamp in UTC. Available snapshots can be listed. After `container init` there is already a first initial snapshot:

```
$ pgarchive snapshot list
2016-04-04T16:31:30Z
```

Snapshots can be created on demand. However note that some time needs to pass (and there needs to be some activity in the upstream DB) to create a new snapshot:

```
$ pgarchive snapshot create
2016-04-04T16:35:10Z The snapshot 2016-04-04T16:01:01Z already exists.
...
$ pgarchive snapshot create
2016-04-04T16:45:44Z Creating snapshot 2016-04-04T16:45:31Z.
Create a readonly snapshot of '/mnt/backup/t2/standby' in '/mnt/backup/t2/snapshots/2016-04-04T16:45:31Z'
```

Usually you do not need to manually create and delete snapshots, see Cron Jobs for a better solution.

TODO

### Restoring Data

Restoring data is done by cloning a snapshot locally (inspired by zfs' terminology).

TODO

### Cron Jobs

As reported by `container init` three cron jobs are created for each container, but initially disabled. You need to verify and enable all three usually, because without them no further snapshots will be created, nor any WAL data expired:

```
# *** pgarchive container /srv/backup/t1 ***
#*/5 * * * *            PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive-dev' cron compress-wal-archive
#01 */4 * * *           PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive-dev' cron expire-and-create-snapshot
#05 01 * * sun          PGARCHIVE='/srv/backup/t1' '/var/lib/pgsql/bin/pgarchive-dev' cron defrag-btrfs
```

You may adapt runtimes to you requirements. The first job rarely needs changing, unless you want to disable compression alltogether.

The second job's runtime affects disk usage by how often snapshots are created, but keep in mind that frequent snapshots also reduce PITR time. There are more options regarding expiry in the `pgarchive.conf` config file. Expiry effects snapshots and the WAL archive.

Since `btrfs` is ill prepared to deal with the large number of random updates to table and index files done by the standby process, you will need to enable the third job too or performance will degrade over time. But it may cause quite some extra IO for considerable time, so it's best to do this during off-times, and not too often. You may experiment and monitor system performance to tune this. A tell-tale sign for btrfs fragmentation issues are climbing CPU utilization and increasing IO latency spikes.


### After Upgrading the Upstream DB

At the moment upgrades of the upstream DB cannot be followed, and this restriction is likely to remain. Supporting different versions inside a single container just adds too much complexity. You *cannot* just update `upstream_version` and `path` in the pgarchive.conf file, this will fail badly and may damage your container.

However it is easy to retire a container and create a new one. Make sure to have the correct (new) PostgreSQL tools in your PATH (pgarchive tries to check and complains about wrong versions). There is no data deduplication between containers so this may increase space usage, plan and monitor for space accordingly.

Note that you may still restore snapshots in your old retired container since it has a saved version of the PATH it was created with, at least as long as you keep your old version's bindir around. You may also continue to run its `expire-and-create-snapshot` cron job until there's no need for the container any more, and then simply purge it (it won't create new snapshots). You may even rename your old container, but make sure to adapt the cron jobs referencing it if you do so.

### Upgrading `pgarchive` itself

First a promise: upgrading `pgarchive` will never require creating new containers, except for new major version numbers. In the future there may be automatic upgrade support. But for now this procudure is necessary:

1. Stop all containers on a single host.
1. Replace the old `pgarchive` version with the new one.
1. Manually modify containers as instructed by the section below.
1. Re-start all containers.
1. Verify.

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


## Command Reference (`pgarchive help)

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
container init-cron Add default cron entries to the calling user's crontab, all jobs disabled.
container checkversions
                    Check versions of upstream DB and local PostgreSQL tools.

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

cron compress-wal-archive <n>
                    Compress completed segments in wal_archive, which have already been
                    processed by the standby process. Run n processes in parallel (default 1).
                    The log is written to log/cron_wal_archive.log.
cron expire-and-create-snapshot
                    Expire old snapshots and wal segments, thin snapshots, and create a new one.
                    The log is written to log/cron_snapshot.log.
cron defrag-btrfs   Defragment the standby directory, since PostgreSQL tends to cause a critical
                    amount of fragmentation fast. Requires password-less sudo privileges to call
                    "/sbin/btrfs filesystem defrag ...". The log is written to log/cron_btrfs.log.

bash-completion     Output a bash completion snippet for this utility.
                    This may be saved to global or personal profiles, or eval'ed directly:
                        eval "$(pgarchive bash-completion)"

Required Environment for all commands:

PGARCHIVE           Base path to a backup container, which must be on btrfs.

Extra environment parameters are used by the 'container init' and 'purge' commands.
All of these settings here are persisted to the $PGARCHIVE/pgarchive.conf config file:

PATH                The PATH used internally. This must contain PostgreSQL's server applications
                    (pg_ctl, pg_basebackup, ...) of the same version as the DB a container
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
                    create a connection slot.
UPSTREAM_REPL       The connection info string the WAL archive process uses to fetch
                    new WAL segments. Defaults to UPSTREAM + " user=replication".
                    All of UPSTREAM's caveats apply.

Container configuration file:

$PGARCHIVE/pgarchive.conf
                    Contains container specific settings, primarily those above and some
                    extra parameters regarding expiry and other cron jobs.
                    This is sourced as a shell script just before commands are executed and could
                    be used to override most internal variables and functions, at your own risk.
```

[archive_timeout]: http://www.postgresql.org/docs/9.4/static/runtime-config-wal.html#GUC-ARCHIVE-TIMEOUT
[btrfs mount options]: https://btrfs.wiki.kernel.org/index.php/Mount_options
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[emarsys.com]: http://emarsys.com/
[pg_basebackup]: http://www.postgresql.org/docs/9.4/static/app-pgbasebackup.html
[pg_receivexlog]: http://www.postgresql.org/docs/9.4/static/app-pgreceivexlog.html
[recovery.conf]: http://www.postgresql.org/docs/9.4/static/recovery-config.html
[replication slots]: http://www.postgresql.org/docs/9.4/static/warm-standby.html#STREAMING-REPLICATION-SLOTS

