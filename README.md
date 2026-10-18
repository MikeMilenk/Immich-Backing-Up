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

My Immich runs on an **Ubuntu Linux server**, and `ext4` is the standard filesystem for Linux, so that's what I'm going to use for the backup disk.
I'll also create it on the **whole disk, without a partition table**. The disk is just for backups and only goes into my Linux server, so there's no point in splitting it up.

```bash
mkfs.ext4 /dev/sdf
```

![Formatting disk](https://github.com/MikeMilenk/Immich-Backing-Up/blob/4b8324a0b9d41e86d4f7547b0320ba9502657795/images/2.3%20-%20Formatting%20disk.png)

Save the filesystem UUID from the `mkfs` output, you'll need it in a minute for `fstab`.

> If you plug this drive into Windows or macOS, it will look empty or uninitialized and offer to format it. Don't accept that, or you'll lose your backup!

**2.4 Create the mount point and mount the disk**

A **mount point** is an empty folder on the system disk. The backup disk gets "attached" to it when mounted. I named my mount point **immich-backup**.

```bash
mkdir -p /mnt/immich-backup
```

Now let's attach given `sdf` disk to that folder:

```bash
mount /dev/sdf /mnt/immich-backup
```

It never hurts to verify that it worked:

```bash
df -h /mnt/immich-backup
```

![Creating backup directory](https://github.com/MikeMilenk/Immich-Backing-Up/blob/4b8324a0b9d41e86d4f7547b0320ba9502657795/images/2.4%20-%20Creating%20backup%20directory.png)

The `Filesystem` column should show the mounted disk (`/dev/sdf`), and the size should match your disk.

**2.5 Adding disk for automount (`fstab`)**

This is where we need the **UUID** from the `mkfs` output. If you forgot it or didn't save it, you can find it with this command:

```bash
blkid /dev/sdX
```
...where "X" is the letter of your disk, in my case `sdf`.
![Find UUID](https://github.com/MikeMilenk/Immich-Backing-Up/blob/da13ba05d262d8a69a7ec41121e3b5837cf16ade/images/2.5.1%20-%20UUID.png)

Now open the `fstab` file.
> **fstab (File Systems TABle)** is a file where Linux keeps the list of disks and the folders they should be mounted to when the server boots.

```bash
nano /etc/fstab
```

Add one line at the end (no quotes and no brackets around the UUID):

```
UUID=<YOUR-UUID>  /mnt/immich-backup  ext4  defaults,nofail  0  2
```

![Adding disk for automount](https://github.com/MikeMilenk/Immich-Backing-Up/blob/726a780f92146e7937301bc11abb7bd65e8551f8/images/2.5.2%20-%20Adding%20disk%20for%20automount.png)

In `fstab` I use the **UUID** instead of `/dev/sdf`. If the letter changes after a reboot, the server could mount the wrong disk into the backup folder. The **UUID** always points to my backup disk.

What each part means:
- `defaults`: standard mount options.
- `nofail`: the server still boots even if the disk is unplugged.
- Optional: you can use `defaults,noauto,nofail` instead. That way a cold disk is never mounted at boot, but you can still mount it by hand with `mount /mnt/immich-backup`.

Let's make sure the line is correct. First unmount the disk:

```bash
umount /mnt/immich-backup
```

Then reload the config and mount everything from `fstab`:

```bash
systemctl daemon-reload
mount -a
```

If both commands print nothing, that's good. No output means no errors.

![Verifying fstab edits](https://github.com/MikeMilenk/Immich-Backing-Up/blob/726a780f92146e7937301bc11abb7bd65e8551f8/images/2.5.3%20-%20Verifying%20fstab%20edits.png)

Check that the disk is mounted:

```bash
df -h /mnt/immich-backup
```

It should show your disk and its size. The unmount at the start is important: if the disk is already mounted, `mount -a` skips it and tests nothing.

**2.8 Optional: reduce the reserved space**

`ext4` format reserves 5% of the disk for root by default. That's useless on a backup-only disk, so I'm going to lower it to 1%.

```bash
tune2fs -m 1 /dev/sdX
```

![Decreasing reserve](https://github.com/MikeMilenk/Immich-Backing-Up/blob/106810e690d774225266c52cad6a282efc352a08/images/2.6%20-%20Decreasing%20reserve%20to%201%25.png)

---

## 3. Add the disk to Proxmox

**3.1 Create the storage**

Open Immich Web Interface and go to: **Datacenter → Storage → Add → Directory**

![Adding Backup Directory: step 1](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9811f9085e49baba40825ed41907fb925389d130/images/3.1.1%20-%20Adding%20Backup%20Directory.png)

- **ID:** `immich-backup`
- **Directory:** `/mnt/immich-backup`
- **Content:** `Backup` only

![Adding Backup Directory: step 2](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9811f9085e49baba40825ed41907fb925389d130/images/3.1.2%20-%20Adding%20Backup%20Directory.png)

Click **Add**, and the new storage should now show up in the list.

![New Added Storage](https://github.com/MikeMilenk/Immich-Backing-Up/blob/78ff6117f8fd9c135d2ef759b64966e1ac80115a/images/3.1.3%20-%20New%20Added%20Storage.png)

**3.2 Block backups when the disk isn't mounted, and verify**

If the backup disk is unplugged, `/mnt/immich-backup` is just an empty folder on the system disk. Without extra restriction, Proxmox would silently write backups there and could fill up the system disk. Next command prevents that: if the backup disk isn't mounted, the backup job fails instead.

```bash
pvesm set immich-backup --is_mountpoint yes
```

Now let's check that everything is in place:

```bash
cat /etc/pve/storage.cfg
```
```bash
pvesm status
```

![Block backups when disk is not mounted](https://github.com/MikeMilenk/Immich-Backing-Up/blob/d4b58c6443683e14cd7a41732ece316ac8d5f30e/images/3.2%20-%20Block%20unnecessary%20backups.png)

`immich-backup` should show as `active`, and its block in `storage.cfg` should contain `is_mountpoint yes`.
