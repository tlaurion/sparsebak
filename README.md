_<h1 align="center">Wyng</h1>_
<p align="center">
Fast incremental backups for logical volumes.
</p>

### Introduction

Wyng is able to deliver faster incremental backups for logical
volumes and disk images. It accesses *copy-on-write* metadata (instead of comparing all data
for each backup) to instantly find changes since the last backup.
Combined with its efficient archive format, Wyng can also very quickly reclaim space
from older backup sessions.

Having nearly instantaneous access to volume changes and a nimble archival format
enables backing up even terabyte-sized volumes multiple times per hour with little
impact on system resources.

Wyng pushes data to archives in a stream-like fashion, which avoids temporary data
caches and re-processing data. And Wyng's ingenious snapshot rotation avoids common
_aging snapshot_ space consumption pitfalls.

Wyng also doesn't require the source admin system to ever mount processed volumes or
to handle them as anything other than blocks, so it safely handles
untrusted data in guest filesystems to bolster container-based security.


### Status

Public release with a range of features including:

 - Incremental backups of Linux logical volumes from Btrfs, XFS and Thin-provisioned LVM

 - Supported destinations: Local filesystem, Virtual machine or SSH host

 - Fast pruning of old backup sessions

 - Basic archive management such as add/delete volume and auto-pruning

 - Automatic management of local snapshots

 - Data deduplication

 - Marking and selecting archived snapshots with user-defined tags

Beta release v0.8 major enhancements:

 - Btrfs and XFS reflink support

 - Authenticated encryption with auth caching
 
 - Full data & metadata integrity checking
 
 - Fast differential receive based on available snapshots

 - Overall faster operation

 - Change autoprune settings with --apdays

 - Configure defaults in /etc/wyng/wyng.ini

 - Mountpoints no longer required at destination

 - Simple selection of archives and local paths: Choose any local or dest each time you run Wyng

 - Multiple volumes can now be specified for most Wyng commands

 Wyng is released under a GPL license and comes with no warranties expressed or implied.


v0.8beta Requirements & Setup
---

Before starting:

* Python 3.8 or greater is required for basic operation.

* For encryption and top performance, the _python3-pycryptodome_ and _python3-zstd_ packages
should be installed, respectively.

* Volumes to be backed-up must reside locally in one of the following snapshot-capable
storage types:  LVM thin-provisioned pool, Btrfs subvolume, or XFS/reflink capable filesystem.

* For backing up from LVM, _thin-provisioning-tools & lvm2_ must be present on the source system.

* The destination system where the Wyng archive is stored (if different from source) should
also have python3, plus a basic Unix command set and filesystem (i.e. a typical Linux or BSD
system). Otherwise, FUSE may be used to access remote storage using sftp or s3 protocols
without concern for python or Unix commands.

* See the 'Testing' section below for tips and caveats about using the alpha and beta versions.


## Getting Started

Wyng is distributed as a single Python executable with no complex
supporting modules or other program files; it can be placed in '/usr/local/bin'
or another place of your choosing.


Archives can be created with `wyng arch-init`:

```
wyng arch-init --dest=ssh://me@exmaple.com:/home/me/mylaptop.backup

...or...

wyng arch-init --dest=file:/mnt/drive1/mylaptop.backup
```

The examples above create a 'mylaptop.backup' directory on the destination.
The `--dest` argument includes the destination type, remote system (where applicable)
and directory path.

Next you can start making backups with `wyng send`:

```
wyng send --dest=file:/mnt/drive1/mylaptop.backup --local=volgrp1/pool1 root-volume home-volume
```

This command sends two volumes 'root-volume' and 'home-volume' from the LVM thin pool 'volgrp1/pool1' to the destination archive.

## Operation

Run Wyng using the following commands and arguments in the form of:

**wyng \[--options] command \[volume_names] \[--options]**


### Command summary

| _Command_ | _Description_  |
|---------|---|
| **list** _[volume_name]_    | List volumes or volume sessions.
| **send** _[volume_name]_    | Perform a backup of enabled volumes.
| **receive** _volume_name [*]_   | Restore volume(s) from the archive.
| **verify** _volume_name [*]_    | Verify volumes' data integrity.
| **prune** _[volume_name] [*]_   | Remove older backup sessions to recover archive space.
| **delete** _volume_name_    | Remove entire volume from config and archive.
| **rename** _vol_name_ _new_name_  | Renames a volume in the archive.
| **arch-init**               | Initialize archive configuration.
| **arch-deduplicate**        | Deduplicate existing data in archive.
| **version**                 | Print the Wyng version and exit.


### Advanced commands

| _Command_ | _Description_  |
|---------|---|
| **monitor**                     | Collect volume change metadata & rotate snapshots.
| **diff** _volume_name [*]_      | Compare local volume with archived volume.
| **add** _volume_name [*]_       | Adds a volume name without session data to the archive.
| **arch-check** _[volume_name] [*]_    | Thorough check of archive data & metadata


#### send

Performs a backup by storing volume data to a new session in the archive.  If the volume
already exists in the archive, incremental mode is automatically used.

```

wyng send my_big_volume --local=vg/pool --dest=file:/mnt/drive1/mylaptop.backup


```

A `send` operation may refuse to backup a volume if there is not enough space on the
destination. One way to avoid this situation is to specify `--autoprune=on` which
will cause Wyng to remove older backup sessions from the archive when space is needed.

Volume names for non-LVM storage may include subdirectories, making them relative paths in
the same manner as file paths in `tar`.
For example, `wyng --local=/mnt/pool1 send appvms/personal.img` will send the volume located
at '/mnt/pool1/appvms/personal.img'.

#### receive

Retrieves a volume instance (using the latest session ID
if `--session` isn't specified) from the archive and saves it to either the `--local`
storage or the path specified with `--save-to`.
If `--session` is used, only one date-time is accepted. The volume name is required.

```

wyng receive vm-work-private --local=vg/pool --dest=file:/mnt/drive1/mylaptop.backup


```

...restores a volume called 'vm-work-private' to 'myfile.img' in
the LVM thin pool 'vg/pool'.  Note that `--dest` always refers to the archive location, so
the volume is being restored _from_ '/mnt/drive1/mylaptop.backup'.

Its possible to receive to any valid file path or block device using the `--save-to` option,
which can be used in place of `--local`.
For any save path, Wyng will try to discard old data before receiving unless `--sparse`,
`--sparse-write` or `--use-snapshot` options are used.


#### verify

The `verify` command is similar to `receive` without saving the data. For both
`receive` and `verify` modes, an error will be reported with a non-zero exit
code if the received data does not pass integrity checks.


#### prune

Quickly reclaims space on a backup drive by removing
any prior backup session you specify; it does this
without re-writing data blocks or compromising volume integrity.

To use, supply a single exact date-time in _YYYYMMDD-HHMMSS_ format to remove a
specific session, or two date-times representing a range:

```
wyng prune --all --session=20180605-000000,20180701-140000 --dest=file:/mnt/drive1/mylaptop.backup
```

...removes backup sessions from midnight on June 5 through 2pm on July 1 for all
volumes. Alternately, `--all-before` may be used with a single `--session` date-time
to prune all sessions prior to that time.

The `--keep` option can accept a single date-time or a tag in the form `^tagID`.
Matching sessions will be excluded from pruning and autopruning.


#### delete

Removes a volume's Wyng-managed snapshots, config and metadata from the source system and
all of its *data* from the destination archive (everything deleted except the source
volume). Use with caution!

An alternate form of `delete` will remove all Wyng archive-related metadata (incl. snapshots) from the
local system without affecting the archive on the destination:

```

wyng delete --clean

```

Alternately, using `delete --clean --all` will remove all known Wyng metadata from the local system,
including any snapshots from the `--local` path.

#### rename
```

wyng rename oldname newname

```

Renames a volume _'oldname'_ in the archive to _'newname'_. Note: This will rename only the
archive volume, _not_ your source volume.


#### arch-deduplicate

De-duplicates the entire archive by removing repeating patterns. This can save space
on the destination's drive while keeping the archived volumes intact.

De-duplication can also be performed incrementally by using `--dedup` with `send`.


```

wyng arch-deduplicate

```


#### arch-init

Initialize a new archive on a mounted drive...
```

wyng arch-init --dest=file:/mnt/backups/archive1

```

Initialize a new archive with stronger compression on a remote system...
```

wyng arch-init --dest=ssh://user@example.com --compression=zstd:7

```

Optional parameters for `arch-init` are _encrypt, compression, hashtype_ and _chunk-factor_.
These cannot be changed for an archive after it is initialized.


#### arch-check

Intensive check of archive integrity, reading each session's _deltas_ completely starting with
the newest and working back to the oldest. This differs from `verify` which first builds a complete
index and checks a session as a complete volume (thus reading delta information from past sessions
in addition to the specified session).

Using `--session=newest` provides a 'verify the last session' function (useful after an incremental
backup). Otherwise, supplying a date-time will make `arch-check` start the check from that point and
then continue working toward the oldest session. Session ranges are not yet supported.

Depending on how `arch-check` is used, the verification process can be shorter _or much longer_
than using `verify` as the latter is always the size of a volume snapshot. The longest, most
complete form `arch-check` is to supply no parameters, which checks all sessions in all volumes.



#### monitor

Frees disk space that is cumulatively occupied by aging snapshots, thereby addressing a
common resource usage issue with snapshot-based backups.
After harvesting their change metadata, the older snapshots are replaced with
new ones occupying zero space.  Running `monitor` isn't necessary,
but it only takes a few seconds and is good to run on a frequent, regular basis
if you have some volumes that are very active. Volume names may also be
specified if its desired to monitor only certain volumes.

This rule in /etc/cron.d runs `monitor` every 20 minutes:

```
*/20 * * * * root su -l -c '/usr/local/bin/wyng monitor --all'
```


#### diff

Compare a local volume snapshot with the archive and report any differences.
This is useful for diagnostics and can also be useful after a verification
error has occurred. The `--remap` option will record any differences into the
volume's current change map, resulting in those blocks being scanned on
the next `send`.


#### add

Adds new, empty volume name(s) to the archive.  On subsequent `send -a`, Wyng will backup
the volume data if it present.


---

### Parameters / Options summary

| _Option_                      | _Description_
|-------------------------------|--------------
--dest=_URL_           | Location of backup archive.
--local=_vg/pool_  _...or..._    | Storage pool containing local volumes.
--local=_/absolute/path_    | 
--authmin=_N_          | Remember authentication for N minutes (default: 2)
--all, -a              | Select all volumes (most cmds); Or clean all (delete).
--volex=_volname_      | Exclude volumes (send, monitor, list, prune).
--dedup, -d            | Use deduplication for send (see notes).
--session=_date-time[,date-time]_ | Select a session or session range by date-time or tag (receive, verify, prune).
--all-before           | Select all sessions before the specified _--session date-time_ (prune).
--autoprune=off        | Automatic pruning by calendar date.
--apdays=_A:B:C:D_     | Number of days to keep or to thin-out older sessions
--keep=_date-time_     | Specify date-time or tag of sessions to keep (prune).
--tag=tagname[,desc]   | Use session tags (send, list).
--sparse               | Receive volume data sparsely (implies --sparse-write)
--sparse-write         | Overwrite local data only where it differs (receive)
--use-snapshot         | Use snapshots when available for faster `receive`.
--unattended, -u       | Don't prompt for interactive input.
--clean                | Perform garbage collection (arch-check) or metadata removal (delete).
--verbose              | Increase details.
--quiet                | Shhh...


### Advanced Options

| _Option_                      | _Description_
|-------------------------------|--------------
--save-to=_path_       | Save volume to _path_ (receive).
--local_from=_json file_ | Specify local:[volumes] sets instead of --local.
--import-other-from    | Import volume data from a non-snapshot capable path during `send`
--encrypt=_cipher_     | Set encryption mode or _'off'_ (default: _'xchacha20-t3'_)
--compression          | (arch-init) Set compression type:level.
--hashtype             | (arch-init) Set data hash algorithm: _hmac-sha256_ or _blake2b_.
--chunk-factor         | (arch-init) Set archive chunk size.
--tar-bypass           | Use direct access for file:/ archives (send)
--passcmd=_'command'_  | Read passphrase from output of a wallet/auth app
--upgrade-format       | Upgrade older Wyng archive to current format. (arch-check)
--remap                | Remap volume to current archive during `send` or `diff`.
--json                 | Output volume: session info in json format (list).
--force                | Not used with most commands.
--meta-dir=_path_      | Use a different metadata dir than the default.
--debug                | Debug mode




### Options Detail

`--dest=URL`

This option tells Wyng where to access the archive and has the same meaning for all read or write
commands. It accepts one of the following forms:

| _URL Form_ | _Destination Type_
|----------|-----------------
|__file:__/path                           | Local filesystem
|__ssh:__//user@example.com[:port][/path]      | SSH server
|__qubes:__//vm-name[/path]                     | Qubes virtual machine
|__qubes-ssh:__//vm-name:me@example.com[:port][/path]  | SSH server via a Qubes VM


`--local`

The location of local storage where logical volumes, disk images, etc. reside.  This serves as
the _source_ for `send` commands, and as the place where `receive` restores/saves volumes.

This parameter takes one of two forms: Either the source volume group and pool as 'vgname/poolname'
or a directory path on a reflink-capable filesystem such as Btrfs or XFS (for Btrfs the path should
end at a subvolume).  Required for commands `send`, `monitor` and `diff` (and `receive` when
not using `--saveto`).


`--session=<date-time>[,<date-time>]` OR
`--session=^<tag>[,^<tag>]`

Session allows you to specify a single date-time or tag spec for the`receive`, `verify`, `diff`,
and `arch-check` commands. Using a tag selects the last session having that tag. When specifying
a tag, it must be prefixed by a `^` carat.

For `prune`, specifying
a tag will have different effects: a single spec using a tag will remove only each individual session
with that tag, whereas a tag in a dual (range) spec will define an inclusive range anchored at the first
instance of the tag (when the tag is the first spec) or the last instance (when the tag is the
second range spec). Also, date-times and tags may be used together in a range spec.


`--volex=<volume1> [--volex=<volume2> *]`

Exclude one or more volumes from processing. May be used with commands that operate on multiple
volumes in a single invocation, such as `send`.  volex is useful in cases where a volume is
in the archive, but frequent automatic backups aren't needed.  Or when certain volumes should
be excluded from prune, monitor, etc.

**Please note:** volex syntax had to be changed from the v0.3 option syntax which used a comma to
specify multiple volumes.


`--sparse-write`

Used with `receive`, the sparse-write mode tells Wyng not to create a brand-new local volume and
results in the data being sparsely written into the existing volume instead. This is useful if
the existing
local volume is a clone/snapshot of another volume and you wish to save local disk space. It is also
best used when the backup/archive storage is local (i.e. fast USB drive or similar) and you don't
want the added CPU usage of full `--sparse` mode.


`--sparse`

The sparse mode can be used with the `receive` command to intelligently retrieve and overwrite
an existing
local volume so that only the differences between local and archived volumes will be fetched
from the archive and written to the local volume. This results in reduced network
usage at the expense of some extra CPU usage on the local machine, and also uses
less local disk space when snapshots are a factor.  The best situation for sparse mode is when
you want to restore/revert a large volume with a containing a limited number of changes
over a low-bandwidth connection.


`--use-snapshot` _(experimental)_

A faster-than-sparse option that uses a snapshot as the baseline for the
`receive`, if one is available.  Use with `--sparse` if you want Wyng to fall back to
sparse mode when snapshots are not already present.


`--tar-bypass` _(experimental)_

Use direct access for file:/ archives during `send`.  This can reduce sending times by
up to 20%.


`--dedup`, `-d`

When used with the `send` command, data chunks from the new backup will be sent only if
they don't already exist somewhere in the archive. Otherwise, a link will be used saving
disk space and possibly time and bandwith.

The trade-off for deduplicating is longer startup time for Wyng, in addition to using more
memory and CPU resources during backups. Using `--dedup` works best if you are backing-up
multiple volumes that have a lot of the same content and/or you are backing-up over a slow
Internet link.


`--autoprune=(off | on | min | full)`

Autoprune may be used with either the `prune` or `send` commands and will cause Wyng to
automatically remove older backup sessions according to date criteria. When used with `send`
specifically, the autopruning process will be triggered in advance of sending new sessions
when using _full_ mode, or in _on_ mode only or if the destination filesytem is
low on free space.  (See _--apdays_ to specify additional autoprune parameters.)

Selectable modes are:

__off__ is the current default.

__on__ removes more sessions than _min_ as space is needed, while trying to retain any/all older sessions
whenever available storage space allows.

__full__ removes all sessions that are due to expire according to above criteria.

`--apdays=A:B:C:D`

Adjust autoprune with the following four parameters:

* A: The oldest day before which _all_ sessions are removed.  Default is 0 (disabled).
* B: Thinning days; the number of days before which _some_ sessions will be removed
according to the ratio _D/C_.  Default is 62 days.
* C: Number of _days_ for the D/C ratio.  Default is 1.
* D: Number of _sessions_ for the D/C ratio.  Default is 2.

An example:  `--apdays=365:31:1:2` will cause autoprune to remove all sessions that are older
than 365 days, and sessions older than 31 days will be thinned-out while preserving
(roughly on average) two sessions per day.

`--tag=<tagname[,description]>`

With `send`, attach a tag name of your choosing to the new backup session/snapshot; this may be
repeated on the command line to add multiple tags. Specifying an empty '' tag will cause Wyng
to ask for one or more tags to be manually input; this also causes `list` to display tag
information when listing sessions.

`--authmin=<minutes>`  
`--passcmd=<command>`

These two options help automate Wyng authentication, and may be used together or separately.

`--authmin` takes a numeric value from -1 to 60 for the
number of minutes to remember the current authentication for subsequent Wyng invocations.
The default authmin time is 2 minutes.  Specifying a -1 will cancel a prior authentication
and 0 will skip storing the authentication.

The `--passcmd` option takes a string representing a shell command that outputs a passphrase, which
Wyng then reads instead of issuing an input prompt for the passphrase.  If a prior auth from
`--authmin` is active, this option is ignored and the command will not be executed.


`--import-other-from=volname:|:path`

Enables `send`ing a volume from a path that is not a supported snapshot storage type.  This may
be any regular file or a block device which is seek-able.

When it is specified this option causes slow delta comparisons to be used for the specified volume(s)
instead of the default fast snapshot-based delta comparisons.  It is not recommended for regular
use with large volumes.

The special delimeter used to separate the _volname_ (archive volume name) and the _path_ is ':|:'
which means this option cannot be used to `send` directly to volume names in the archive which
contain that character sequence.


`--local_from=_json file_`

Specify both local storage and volume names for `send` or `receive` as sets, instead
of using --local and volume names on the command line.  The json file must take the form
of `{local-a: [[volname1, alias1], [volnameN, aliasN], ...], ...]}`.  This allows multiple
local storage sources to be sent/received in a single session.  However, the volume names (or aliases)
must all be unique across different sources as they are stored in the same archive.  Aliases
currently define which local volume name into which an archive volume will be received; they
are ignored when sending.


`--compression`

Accepts the forms `type` or `type:level`. The three types available are `zstd` (zstandard),
plus `zlib` and `bz2` (bzip2). Note that Wyng will only default
to `zstd` when the 'python3-zstd' package is installed; otherwise it will fall back to the less
capable `zlib`. (default=zstd:3)


`--hashtype`

Accepts a value of either _'blake2b'_ or _'hmac-sha256'_ (default).  The digest size is 256 bits.


`--chunk-factor`

Sets the pre-compression data chunk size used within the destination archive.
Accepted range is an integer exponent from '1' to '6', resulting in a chunk size of 64kB for
factor '1', 128kB for factor '2', 256kB for factor '3' and so on. To maintain a good
space efficiency and performance balance, a factor of '2' or greater is suggested for archives
that will store volumes larger than about 100GB. (default=2)


`--encrypt`

Selects the encryption cipher/mode.  The available modes are:

- `xchacha20-dgr` — Using HMAC-SHA256(rnd||hash) function.  This is the default.
- `xchacha20-msr` — Using HMAC-SHA256(rnd||msg) function.
- `xchacha20-ct` — Counter based; fast with certain safety trade-offs (see issue [158](https://github.com/tasket/wyng-backup/issues/158)).
- `off` — Turns off Wyng's authentication and encryption.



### Configuration files

Wyng will look in _'/etc/wyng/wyng.ini'_ for option defaults.  For options that are flags with
no value like `--dedup`, use a _1_ or _0_ to indicate _enable_ or _disable_ (yes or no).
For options allowing multiple entries per command line, in the .ini use multiple lines with the
2nd item onward indented by at least one space.

An example _wyng.ini_ file:

```
[var-global-default]
dedup = 1
authmin = 10
autoprune = full
dest = ssh://user@192.168.0.8/home/user/wyng.backup
local = /mnt/btrfs01/vms
volex = misc/caches.img
  misc/deprecated_apps.img
  windows10_recovery.vmdk
```


### Verifying Code

* Wyng code can be cryptographically verified using either `gpg` directly or via `git`:

```sh
# Import Key
~$ cd wyng-backup
~/wyng-backup$ gpg --import pubkey
gpg: key 1DC4D106F07F1886: public key "Christopher Laprise <tasket@posteo.net>" imported
gpg: Total number processed: 1
gpg:               imported: 1

# GPG Method
~/wyng-backup$ gpg --verify src/wyng.gpg src/wyng

# Git Method
~/wyng-backup$ git verify-commit HEAD

# Output:
gpg: Signature made Sat 26 Aug 2023 04:20:46 PM EDT
gpg:                using RSA key 0573D1F63412AF043C47B8C8448568C8B281C952
gpg: Good signature from "Christopher Laprise <tasket@posteo.net>" [unknown]
gpg:                 aka "Christopher Laprise <tasket@protonmail.com>" [unknown]
```


### Protecting and Verifying Archive Authenticity

With encryption enabled, Wyng provides a kind of built-in verification of archive authenticity;
this is because it uses an AEAD cipher mode.  However, custom verification
(BYOV) is also possible with Wyng and even works on non-encrypted archives.  All you need
to do is sign the 'archive.ini' file from the top archive directory after executing any Wyng
command that changes the archive (i.e. _arch-init, add, send, prune, delete, rename_).

Subsequently, the steps to verify total archive authenticity would be to simply run
`wyng arch-check --dest <URL>` (using Wyng's built-in authenticated encryption), or else using custom
authentication based on GPG, for instance:
```
gpg --verify archive.ini.sig laptop1.backup/archive.ini && wyng arch-check --dest <URL>
```
Note that custom signature files should _not_ be stored within the archive directory.

(Although volumes can be verified piecemeal with the `wyng verify` command, it is not suited
to verifying everything within an archive.)

#### Security side note

Authentication schemes in general can only verify the authenticity for an
object at any point in time; they aren't well suited to telling us if that object
(i.e. a backup archive) is the most recent update, and so they are vulnerable to rollback
attacks that replace your current archive with an older version (in Wyng this is related to
replay attacks, but not downgrade attacks).  Wyng guards against
such attacks by checking that the time encoded in your locally cached archive.ini isn't newer
than the one on the destination/remote; Wyng also displays the last archive modification time
whenever you access it.


### Tips & Caveats

* LVM users: Wyng has an internal snapshot manager which creates snapshots of volumes
in addition to any snapshots you may already have on your local storage system.
This can pose a serious challenge to _lvmthin_ (aka thin-provisioned LVM) as the default space
allocated for metadata is often too small for rigorous & repeated snapshot rotation
cycles.  It is recommended to _at least double_ the existing or default tmeta space
on each thin pool used with `wyng send` or `wyng monitor`; see the man page
section _[Manually manage free metadata space of a thin pool LV](https://www.linux.org/docs/man7/lvmthin.html)_ for guidance on using
the `lvextend --poolmetadatasize` command.

* To reduce the size of incremental backups it may be helpful to remove cache
files, if they exist in your source volume(s). Typically, the greatest cache space
consumption comes from web browsers, so
volumes holding paths like /home/user/.cache can impacted by this, depending
on the amount and type of browser use associated with the volume. Three possible
approaches are to clear caches on browser exit, delete /home/user/.cache dirs on
system/container shutdown (this reasonably assumes cached data is expendable),
or to mount .cache on a separate volume that is not configured for backup.

* If you've changed your local path without first running `wyng delete --clean` to
remove snapshots, there may be unwanted snapshots remaining under your old volume group
or local directory.  LVM snapshots can be found with the patterns `*.tick` and `*.tock` with
the tag "wyng";  Btrfs/XFS snapshots can be found with `sn*.wyng?`.
Deleting them can prevent unnecessary consumption of disk space.


### Troubleshooting notes

* Since v0.4alpha3, Wyng may appear at first to not recognize older alpha archives.
This is because Wyng no longer adds '/wyng.backup040/default' to the `--dest` path. To access the
archives simply add those two dirs to the end of your `--dest` URLs.  Alternately, you can rename
those subdirs to a single dir of your choosing.

* Archives from older v0.3 versions of Wyng must be upgraded before they can be used with
later versions.  Run `wyng arch-check --upgrade-format` to perform the upgrade, which will
convert an archive in-place after creating a backup of the metadata as 'wyng_metadata_bak.tbz'
in your current directory in case something goes wrong during the procedure.  A full manual
backup of the archive is also recommended before running this procedure (see tip to efficiently
'backup the backup' under the Testing section).  Also note that 'upgraded' archives continue
to use the old hashing and compression settings and will remain unencrypted;  you may want to
consider setting aside the old archive and create a new encrypted archive for your backups
going forward.

* A major change in v0.8 is that for `send` and `monitor` Wyng will no longer assume you want to
act on all known volumes if you don't specify any volumes.  You must now use `-a` or `--all`, which
now work for other commands as well.  This change also enables adding new volumes while doing a
complete backup, for instance: `wyng -a send my-new-volume` – updates every volume already
in the archive plus backup 'my-new-volume' as well.

* Backup sessions shown in `list` output may be seemingly (but not actually) out of
order if the system's local time shifts
substantially between backups, such as when moving between time zones (including DST).
If this results in undesired selections with `--session` ranges, its possible
to nail down the precisely desired range by observing the output of
`list volumename` and using exact date-times from the listing.

* Wyng locally stores information about backups in two ways:  Snapshots alongside your local
source volumes, and metadata under _/var/lib/wyng_.  It is safe to _delete_
Wyng snapshots without risking the integrity of backups (although `send` will become slower).
However, as with all CoW snapshot based backup tools, you should never attempt to directly mount,
alter or otherwise utilize a Wyng snapshot
as this could (very likely) result in future backup sessions being corrupt (this is why Wyng
snapshots are stored as read-only).  If you think you have somehow altered a Wyng snapshot, you
should consider it corrupt and immediately delete it before the next `send`.
If you're in a pinch and need to use the data in a Wyng snapshot, you should first make your own
copy or snapshot of the Wyng snapshot using `cp --reflink` or `lvcreate -s` and use that instead.

* Metadata cached under _/var/lib/wyng_ may also be manually deleted.  However, the _archive.\*_
root files in each 'a_*' directory are part of Wyng's defense against rollback attacks, so if you
feel the need to manually reclaim space used in this dir then consider leaving the _archive.\*_
files in place.


### Testing

* Wyng v0.4alpha3 and later no longer create or require the `wyng.backup040/default`
directory structure.  This means whatever you specify
in `--dest` is all there is to the archive path.  It also means accessing an alpha1 or
alpha2 archive will require you to either include those dirs explicitly in your --dest path
or rename '../wyng.backup040/default' to something else you prefer to use.

* Testing goals are basically stability, usability, security and efficiency. Compatibility
is also a valued topic, where source systems are generally expected to be a fairly recent
Linux distro or Qubes OS. Destination systems can vary a lot, they just need to have Python and
Unix commands or support a compatible FUSE protocol such as sshfs(sftp) or s3.

* If you wish to run Wyng operations that you want to roll back later,
its possible to "backup the backup" in a relatively quick manner using a hardlink copy:
```
sudo cp -rl /dest/path/wyng.backup /dest/path/wyng.backup-02

...or...

rsync -a --hard-links --delete source dest
```

The `rsync` command is also suitable for efficiently updating an archive copy, since it can
delete files that are no longer present in the origin archive (`cp` is not suitable for
this purpose).



### Donations

<a href="https://liberapay.com/tasket/donate"><img alt="Donate using Liberapay" src="media/lp_donate.svg" height=54></a>

<a href="https://www.buymeacoffee.com/tasket"><img src="media/buymeacoffee_57.png" height=57></a> <a href="https://www.buymeacoffee.com/tasket">Buy me a coffee!</a>

<a href="https://www.patreon.com/tasket"><img alt="Donate with Patreon" src="media/become_a_patron_button.png" height=50></a>

If you like this project, monetary contributions are welcome and can
be made through [Liberapay](https://liberapay.com/tasket/donate) or [Buymeacoffee](https://www.buymeacoffee.com/tasket) 
or [Patreon](https://www.patreon.com/tasket).
