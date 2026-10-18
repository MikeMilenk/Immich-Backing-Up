# Immich-Backing-Up
This guide covers how to back up Immich on Proxmox, including the VM system disk, media and database dumps to an external HDD. The backup disk stays unplugged most of the time ("cold" backup).

## Table of contents

1. [Overview](#1-overview)
2. [Prepare the external backup disk](#2-prepare-the-external-backup-disk)
3. [Add the disk to Proxmox](#3-add-the-disk-to-proxmox)
4. [Understand how Immich stores data](#4-understand-how-immich-stores-data)
5. [Exclude the media disk from the VM backup](#5-exclude-the-media-disk-from-the-vm-backup)
6. [Run the backup](#6-run-the-backup)
7. [What ends up on the backup disk](#7-what-ends-up-on-the-backup-disk)
8. [Restore](#8-restore)
9. [Notes and limitations](#9-notes-and-limitations)

---

## 1. Overview

**How my Immich is set up**

My Immich is split into two parts:
1) **SSD:** the OS and all the system settings (Docker, configs, the database). I keep it here so the system runs fast.
2) **HDD:** all the media. It's a big, roomy drive, which is what I want for stuff that keeps growing.

Here I show how to back up both of these separate disks together onto one external drive. The backup target is an external HDD, connected to the Proxmox host by hand about once a month. Since my Immich is split across 2 drives, I back them up in 2 different ways:

| What | Where it is | How I back it up |
|---|---|---|
| The system (OS, Docker, configs, database) | SSD | Proxmox's built-in backup (`vzdump`) |
| The media (photos, videos, thumbnails, database dumps) | HDD | `rsync` |

Why 2 ways? The media HDD is passed through to the VM as a whole disk, so Proxmox's backup button can't handle it nicely. It's easier to copy it separately.

`rsync` only copies what's new, so the first backup takes a while, but the next ones are quick. It never deletes anything from the backup disk, so if I delete a photo in Immich, it stays in my backup.

## 2. Prepare the external backup disk

**2.1 Find the disk**

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINT
```

This command lists all disks with size, model, filesystem and mount point. Identify the new disk by its size and model. In my case it's `sdf`.

![List disks](https://github.com/MikeMilenk/Immich-Backing-Up/blob/6f15fc9b690d2920560502eb07b0d3647ae90f8d/images/2.1%20-%20Checking%20Disks.png)

**2.2 Wipe old signatures**

> **Careful:** `wipefs` and `mkfs` erase the disk. Always check the disk name with `lsblk` first, so you don't wipe the wrong one.

>Again, in my case the backup disk is disk **F** (`sdf`), so that's what you'll see in the commands below. If your disk has a different name, use yours instead.

I wiped the old filesystem and partition table, so I could set the disk up from scratch, just for this backup.

```bash
wipefs -a /dev/sdf
```

**2.3 Format as ext4**

```bash
mkfs.ext4 /dev/sdX
```

My Immich runs on an **Ubuntu Linux server**, and `ext4` is the standard filesystem for Linux, so that's what I used for the backup disk.
I also created it on the **whole disk, without a partition table**. The disk is just for backups and only goes into my Linux server, so there's no point in splitting it up.

![Formatting disk](https://github.com/MikeMilenk/Immich-Backing-Up/blob/6f15fc9b690d2920560502eb07b0d3647ae90f8d/images/2.2%20-%20Formatting%20disk.png)

Save the filesystem UUID from the `mkfs` output, you'll need it in a minute for `fstab`.

> If you plug this drive into Windows or macOS, it will look empty or uninitialized and offer to format it. Don't accept that, or you'll lose your backup!
