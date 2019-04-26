---
layout: post
title: "Backup ZFS on Linux to FreeNAS using Syncoid"
date: "2019-04-26 11:19:10 -0500"
---

## Introduction
I :heart: ZFS.  It has saved me from disaster more times than I can count.  I
also :heart: FreeNAS and FreeBSD.  My FreeNAS server sits in my home office,
serving up media and file sharing for my entire home.  I recently decided to
automate my ZFS backups to my FreeNAS server.  These are my notes from that
process:

## Requirements

My laptop and desktop are running Ubuntu Linux, with the root filesystem on
ZFS.  This allows me to simply backup the root pool recursively and I will have
everything I need.  However, I also have a few filesystems in my `zroot` pool
that I do not wish to back up.  For example:

- `zroot/swap`: my swap partition
- `zroot/var/docker`: hundreds of filesystems under here due to many docker
  containers for work.  If disaster strikes I'd rather just rebuild these or
  re-download from docker hub.
- `zroot/var/tmp`: temporary files don't need to be backed up
- `zroot/var/cache`: same for cache files

I also have a number of Virtual Machines running on `zvol` storage.  I need to
be able to back these up as well.

For automatic snapshots, I use
[zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot) to keep a
set of snapshots on my machines automatically with varying frequences (e.g. 1
days worth of hourly snapshots, 1 month of weekly snapshots etc).  My
requements where to replicate my `zroot` pool and all of its snapshots to a ZFS
pool on FreeNAS, keeping all of the snapshots, plus keeping older snapshots on
FreeNAS.

## Encryption

My FreeNAS drives are all `geli` encrypted.  I do not care if the actual
backups themselves are encrypted.  My main concern is that if a disk fails and
I need to get rid of it, or if I wish to sell it to upgrade the disk, no
information will be compromised, and `geli` device level encryption solves this.

## Options

ZFS comes with built in tools to replicate itself via `zfs send` and `zfs
recv`, either incrementally or in full to another system.  These tools are very
low level however, and some plumbing around these tools is ideal.  I evaluated
several tools to replicate snapshots including:

- [znapzend](https://www.znapzend.org/)
- [zfsbackup-go](https://github.com/someone1/zfsbackup-go)
- [zrep](www.bolthole.com/solaris/zrep)
- [zetaback](https://github.com/omniti-labs/zetaback)
- [syncoid](https://github.com/jimsalterjrs/sanoid)

In the end, the only one that met **all** of my requirements was
[syncoid](https://github.com/jimsalterjrs/sanoid).  The others either simply
did not work with FreeNAS and/or Ubuntu, or had limitations such as not being
able to use an existing snapshot system, or, could not handle `zvols`, etc.

## Setup

### On the FreeNAS server

If necessary, setup password-less `root` login from your machines via SSH keys.
I already had this set up.

The only other requirement is to create a zfs filesystem to store the backups.
For example:

```shell
zfs create pool/zfs/somehostname
```

You could do this through the FreeNAS UI, but I prefer the CLI.

Note also that your default shell on FreeNAS needs to be `/bin/sh` because
`syncoid` does not work with the default `csh`.  So possibly you also need:

```shell
chsh -s /bin/sh
```

### On the machine to back up

First, on my machines to back up, I excluded the filesystems I did not want to
back up. This wasn't really clear from the `syncoid` documentation, but the way
to do this is to set `syncoid:sync=false` on the filesystems that you want to
exclude.  If a filesystem (or it's parent) has this property, the filesystem
will not be backed up.

```shell
zfs set syncoid:sync=false zroot/swap
zfs set syncoid:sync=false zroot/var/docker
zfs set syncoid:sync=false zroot/var/tmp
```

Install `syncoid` somewhere like `/usr/local/bin`, and run it.  I use this
incantation:

```shell
/usr/local/bin/syncoid \
    --quiet \
    --recursive \
    --compress=none \
    zroot root@[my-freenas-ip]:backup/zfs/[my hostname]
```

The first time this runs, it will do a full backup.  Subsequent runs will only
do an incremental backup from the last run.  In addition, if the backup is
interrupted, the next run will resume where it left off.

Thats it.  Everything just works.

Ideally, you don't want to run backups manually though.  You could hook this up
to `cron` for example.   I hooked it up to `systemd` using the `OnCalendar`
feature, and a `OnFailure` setting that emits a `dbus` notification to all
logged in users if the backup fails, but that is a topic for another blog post.

