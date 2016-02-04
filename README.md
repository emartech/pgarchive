# pgarchive

A PostgreSQL Archive Server providing fast and incremental Backup and Restore Services

## Release Status

This is a very early alpha software. You will need to change parameters and possibly code in a shell script to make this work for you.

It is used in production at emarsys.com to back up large (up to 1TB sizec, 100GB WAL per day) databases and you may find it useful or at least inspirational.

## Features and Design Goals

* pgarchive runs an archive container service using local storage
    * more than just a one-shot backup and restore tool
    * can backup local and remote databases
    * works for whole DB clusters (like pg_basebackup, not like pg_dump)
    * incremental data storage
    * can run containers for multiple database servers on a single host
    * a database server can be backed up to multiple pgarchive containers (e.g. at different sites)
    * can backup a hot standby server
    * CLI interface to create and manage containers, and assist in recovery
* avoinding complexity and reinventing the wheel
    * it's a single shell script without fancy dependencies
    * most work is accomplished using the built-in PostgreSQL applications, sometimes in novel ways
    * uses [btrfs] to store only the incremental difference between snapshots of a database cluster directory
    * uses cron for time based jobs
* user friendliness
    * it is intended to be usable by people who are not PostgreSQL experts for routine tasks
    * one command initialization and teardown of whole backup containers
    * supports restore with point in time recovery
    * includes maintenance tasks for time based expiry, compression, etc. via cron
    * keeps helpful logs
    * has a simple dashboard
    * custom bash completion
* little impact on upstream database, continuous operation
    * after initial setup containers pull down data using only streaming replication
    * no load spikes or delays due to full dumps, pg_basebackups or rsyncs
    * does not require instrumentation of upstream DB except for a replication slot, which it creates and removes itself
    * the upstream DB does not need to (and should not!) run on btrfs
* fast (local) restore
    * as fast as activating a btrfs snapshot and doing postgres recovery to consistency
    * allow point in time recovery forward from any snapshot
    * full disaster recovery is currently not supported, but easy to do and add in the future (rsync or [pg_basebackup] a local copy somewhere else)
* resilience
    * to outages of the upstream DB
    * network interruptions
    * continues where it stopped if interrupted
    * it is easy to monitor


## Caveats

* opinionated
    * uses standard tools in a unique way
    * presumes some upstream configuration parameters, like CSV logging, and an available streaming replication connection and slot
    * at the moment contains many hard coded configuration parameters in the script instead of in a configuration file
* requires PostgreSQL 9.4
    * due to its architecture and use of replication slots earlier versions are not easily supported
    * changes for 9.5 should be trivial and will be done soon
    * does not support cross-version backup&restore, containers must match upstream databases exactly
* requires btrfs (for containers), despite well known problems with PostgreSQL
    * locally restored and started database clusters run on btrfs, and will be slow
    * btrfs fragmentation issues cause even more performance loss over time, needs frequent defragmentation
    * deemed acceptable for an archive system, as the throughput is good enough and snapshots are a huge benefit
* use at your own risk!


## About

For more information please read the [manual](MANUAL.md).

Created by Jürgen Strobel <juergen.strobel@emarsys.com> while working at [emarsys.com].


[archive_timeout]: http://www.postgresql.org/docs/9.4/static/runtime-config-wal.html#GUC-ARCHIVE-TIMEOUT
[btrfs mount options]: https://btrfs.wiki.kernel.org/index.php/Mount_options
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[emarsys.com]: http://emarsys.com/
[pg_basebackup]: http://www.postgresql.org/docs/9.4/static/app-pgbasebackup.html
[pg_receivexlog]: http://www.postgresql.org/docs/9.4/static/app-pgreceivexlog.html
[recovery.conf]: http://www.postgresql.org/docs/9.4/static/recovery-config.html
[replication slots]: http://www.postgresql.org/docs/9.4/static/warm-standby.html#STREAMING-REPLICATION-SLOTS
