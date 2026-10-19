# Immich-Backing-Up
This guide covers how to back up Immich on Proxmox, including the VM system disk, media and database dumps to an external HDD. The backup disk stays unplugged most of the time ("cold" backup).

## Table of contents

1. [Overview](#1-overview)
2. [Prepare the external backup disk](#2-prepare-the-external-backup-disk)
    - [2.1 Find the disk](#21-find-the-disk)
    - [2.2 Wipe old signatures](#22-wipe-old-signatures)
    - [2.3 Format as ext4](#23-format-as-ext4)
    - [2.4 Create the mount point and mount the disk](#24-create-the-mount-point-and-mount-the-disk)
    - [2.5 Adding disk for automount (`fstab`)](#25-adding-disk-for-automount-fstab)
    - [2.6 Optional: reduce the reserved space](#26-optional-reduce-the-reserved-space)
3. [Add the disk to Proxmox](#3-add-the-disk-to-proxmox)
    - [3.1 Create the storage](#31-create-the-storage)
    - [3.2 Block backups when the disk isn't mounted, and verify](#32-block-backups-when-the-disk-isnt-mounted-and-verify)
4. [Understand how Immich stores data](#4-understand-how-immich-stores-data)
5. [Exclude the media disk from the VM backup](#5-exclude-the-media-disk-from-the-vm-backup)
6. [Run the backup](#6-run-the-backup)
    - [6.1 Back up the system](#61-back-up-the-system)
    - [6.2 Mount the media disk on the host (read-only)](#62-mount-the-media-disk-on-the-host-read-only)
    - [6.3 Copy the media](#63-copy-the-media)
    - [6.4 Verify copying](#64-verify-copying)
    - [6.5 Unmount and finish](#65-unmount-and-finish)
7. [What ends up on the backup disk](#7-what-ends-up-on-the-backup-disk)
8. [Restore](#8-restore)

---

## 1. Overview

**How my Immich is set up**

My Immich is split into 2 parts:
1) **SSD:** the OS and all the system settings (Docker, configs, the database). I keep it here so the system runs fast.
2) **HDD:** all the media. It's a big, roomy drive, which is what I want for stuff that keeps growing.

Here I show how to back up both of these separate disks together onto one external drive. The backup target is an external HDD, connected to the Proxmox host by hand about once a month. Since my Immich is split across 2 drives, I back them up in 2 different ways:

| What | Where it is | How I back it up |
|---|---|---|
| The system (OS, Docker, configs, database) | SSD | `vzdump` Proxmox's built-in backup |
| The media (photos, videos, thumbnails, database dumps) | HDD | `rsync` |

Why 2 ways? The media HDD is passed through to the VM as a whole disk, so Proxmox's backup button can't handle it nicely. It's easier to copy it separately.

`rsync` only copies what's new, so the first backup takes a while, but the next ones are quick. It never deletes anything from the backup disk, so if I delete a photo in Immich, it stays in my backup.

## 2. Prepare the external backup disk

### 2.1 Find the disk

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINT
```

This command lists all disks with size, model, filesystem and mount point. Identify the new disk by its size and model. In my case it's `sdf`.

![List disks](https://github.com/MikeMilenk/Immich-Backing-Up/blob/6f15fc9b690d2920560502eb07b0d3647ae90f8d/images/2.1%20-%20Checking%20Disks.png)

### 2.2 Wipe old signatures

> **Careful:** `wipefs` and `mkfs` erase the disk. Always check the disk name with `lsblk` first, so you don't wipe the wrong one.

>Again, in my case the backup disk is disk **F** (`sdf`), so that's what you'll see in the commands below. If your disk has a different name, use yours instead.

I wiped the old filesystem and partition table, so I could set the disk up from scratch, just for this backup.

```bash
wipefs -a /dev/sdf
```

### 2.3 Format as ext4

My Immich runs on an **Ubuntu Linux server**, and `ext4` is the standard filesystem for Linux, so that's what I'm going to use for the backup disk.
I'll also create it on the **whole disk, without a partition table**. The disk is just for backups and only goes into my Linux server, so there's no point in splitting it up.

```bash
mkfs.ext4 /dev/sdf
```

![Formatting disk](https://github.com/MikeMilenk/Immich-Backing-Up/blob/4b8324a0b9d41e86d4f7547b0320ba9502657795/images/2.3%20-%20Formatting%20disk.png)

Save the filesystem UUID from the `mkfs` output, you'll need it in a minute for `fstab`.

> If you plug this drive into Windows or macOS, it will look empty or uninitialized and offer to format it. Don't accept that, or you'll lose your backup!

 ### 2.4 Create the mount point and mount the disk

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

### 2.5 Adding disk for automount (`fstab`)

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

Add one line at the end (no quotes and no brackets around the **UUID**):

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

### 2.6 Optional: reduce the reserved space

`ext4` format reserves 5% of the disk for root by default. That's useless on a backup-only disk, so I'm going to lower it to 1%.

```bash
tune2fs -m 1 /dev/sdX
```

![Decreasing reserve](https://github.com/MikeMilenk/Immich-Backing-Up/blob/106810e690d774225266c52cad6a282efc352a08/images/2.6%20-%20Decreasing%20reserve%20to%201%25.png)

---

## 3. Add the disk to Proxmox

### 3.1 Create the storage

Open Immich Web Interface and go to: **Datacenter → Storage → Add → Directory**

![Adding Backup Directory: step 1](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9811f9085e49baba40825ed41907fb925389d130/images/3.1.1%20-%20Adding%20Backup%20Directory.png)

- **ID:** `immich-backup`
- **Directory:** `/mnt/immich-backup`
- **Content:** `Backup` only

![Adding Backup Directory: step 2](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9811f9085e49baba40825ed41907fb925389d130/images/3.1.2%20-%20Adding%20Backup%20Directory.png)

Click **Add**, and the new storage should now show up in the list.

![New Added Storage](https://github.com/MikeMilenk/Immich-Backing-Up/blob/78ff6117f8fd9c135d2ef759b64966e1ac80115a/images/3.1.3%20-%20New%20Added%20Storage.png)

### 3.2 Block backups when the disk isn't mounted, and verify

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

---

## 4. Understand how Immich stores data

Inside the Immich VM, look at the Immich `.env` file.
The `.env` file holds the settings **Docker Compose uses** to start **Immich**: where the photos are stored, where the database lives, the Immich version, and the database password. The `.env `file is hidden because its name starts with a *dot*, so a plain `ls` won't show it. You can see it with `ls -a`.

![env file location](https://github.com/MikeMilenk/Immich-Backing-Up/blob/59b6565b66708e376c1cb7d5398f4982a857b9f4/images/4.1%20-%20env%20file%20location.png)

In my case, it's in the `immich-app` folder inside my home directory. I created that folder when I set up **Immich** (here's [my guide](https://github.com/MikeMilenk/Immich-deployment)).

Before backing up, I want to know what's stored where, so I open `.env` to find out where Immich keeps its data (`UPLOAD_LOCATION`).

```bash
cat ~/immich-app/.env
```

![env file content](https://github.com/MikeMilenk/Immich-Backing-Up/blob/fc00037a6da891e106d4b193d66915496b99d934/images/4.2%20-%20env%20file%20content.png)

| Variable | Meaning |
|---|---|
| `UPLOAD_LOCATION` | Where media, thumbnails, profile pictures and database dumps are stored |
| `DB_DATA_LOCATION` | Live Postgres data (do not copy it by hand, use the dumps) |

> **Never publish your `DB_PASSWORD`.

Check what is on the media disk:

- `Findmnt` shows which disk is mounted in that folder (in my case the media HDD, not the system disk)
```bash
findmnt /mnt/immich-storage
```
- `du` shows how big each folder is, so I know the backup disk has enough space.
```bash
du -sh /mnt/immich-storage/library/*
```
  The `upload` folder in *library* directory is where my original media are stored, and it's the biggest one by far: **631 GB**.

- `ls -la` checks that Immich makes daily database dumps. The dumps hold albums, users and settings, so without them the backup would be just photos.
```bash
ls -la /mnt/immich-storage/library/backups
```

![media disk content](https://github.com/MikeMilenk/Immich-Backing-Up/blob/fc00037a6da891e106d4b193d66915496b99d934/images/4.3%20-%20media%20disk%20content.png)

  The file name contains the **Immich** and **Postgres** versions. You need compatible versions to restore.

---

## 5. Exclude the media disk from the VM backup

By default, Proxmox's backup (`vzdump`) also includes passthrough disks, so it would try to copy my whole media HDD. I don't want that, because I back up the media separately with `rsync`.

**VM → Hardware → Hard Disk (the passthrough disk) → Edit → Advanced → uncheck "Backup" → OK**

![Disable media disk backup](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9b8ece26ccd3708f9c016b13a6219baf3daf6e19/images/5.1%20-%20Disable%20media%20disk%20backup.png)

Keep it unchecked permanently. In the disk line you should see `backup=0`.

![backup=0](https://github.com/MikeMilenk/Immich-Backing-Up/blob/9b8ece26ccd3708f9c016b13a6219baf3daf6e19/images/5.2%20-%20Backup%3D0.png)

---

## 6. Run the backup
> Repeat this procedure every month.

### 6.1 Back up the system

**Shutdown** the VM and wait until the VM is stopped. This is required because the media disk must not be mounted in the VM and on the host at the same time, which can corrupt the filesystem. It also guarantees a consistent copy.

Connect and mount the backup disk (`mount /mnt/immich-backup`), then:

**VM → Backup → Backup now**

- **Storage:** `immich-backup`
- **Mode:** `Stop` (the VM is already off, so this is the safest mode)
- **Compression:** `ZSTD`

![System Backup](https://github.com/MikeMilenk/Immich-Backing-Up/blob/08fab6a66744a6cc65dfb4bb0bb9cfba88e176b3/images/6.1%20-%20Backup%20System%20(SSD).png)

Wait for `TASK OK`. The result is a `.vma.zst` file in `dump/` if you need it.

### 6.3 Mount the media disk on the host (read-only)

First, you need to know which disk to mount. For that you need its model. Run `lsblk` with the model column:

```bash
lsblk -o NAME,SIZE,MODEL
```

Now let's get the permanent name of that disk. Find your media disk by its size and note the model. Device letters like `sdа` can change, but the name in `/dev/disk/by-id` always points to the same disk. Mine is 7.3T and the model is `WD8002FZWX`:

```bash
ls /dev/disk/by-id | grep <DISK-MODEL>
```
So in my case it would be:
```bash
ls /dev/disk/by-id | grep WD8002
```

![Connecting HDD to host Proxmox step 1](https://github.com/MikeMilenk/Immich-Backing-Up/blob/cf232ba41283b6698047816c288682ac4434df60/images/6.2.1%20-%20Connecting%20HDD%20to%20host%20Proxmox.png)

You'll see two entries. The one ending in `-part1` is the partition with the photos, and that's the one we mount. Enter your *<DISK-ID>* (no quotes and no brackets around the **DISK-ID**).

- Create an empty folder where we'll attach the media disk:
```bash
mkdir -p /mnt/immich-hdd
```

- Attach the media disk to that folder. `ro` means read-only, so nothing on the disk can be changed by accident.
```bash
mount -o ro /dev/disk/by-id/<DISK-ID>-part1 /mnt/immich-hdd
```

- Check that both disks are mounted and that the backup disk has enough free space.
```bash
df -h /mnt/immich-hdd /mnt/immich-backup
```

![Connecting HDD to host Proxmox step 2](https://github.com/MikeMilenk/Immich-Backing-Up/blob/cf232ba41283b6698047816c288682ac4434df60/images/6.2.2%20-%20Connecting%20HDD%20to%20host%20Proxmox.png)

> **Don't** add this disk to `fstab`: it belongs to the VM, and the host should only mount it by hand while the VM is off. Check that the backup disk has enough free space.

### 6.4 Copy the media

```bash
rsync -a --info=progress2 /mnt/immich-hdd/ /mnt/immich-backup/media/
```

| Part | Meaning |
|---|---|
| `rsync` | Copies only files that are new or changed |
| `-a` | Archive mode: keeps folder structure, timestamps and permissions |
| `--info=progress2` | Shows one overall progress line |
| `/mnt/immich-hdd/` | Source. The trailing `/` means "the contents of this folder" |
| `/mnt/immich-backup/media/` | Destination (created automatically) |

![Backing up Immich media](https://github.com/MikeMilenk/Immich-Backing-Up/blob/d5eefd8f1b9cb776c400fdeddc3b29ee3b47afe7/images/6.5.1%20-%20Backing%20up%20Immich%20media.png)

A few points:
- Keep the shell tab open until it finishes. If it is interrupted, run the same command again: it continues where it stopped.
- The percentage and remaining time are unreliable, because `rsync` discovers files while copying. Copying my ~750 GB of media took about 6h, even though `rsync` first estimated only about 10 minutes. To see the progress, check `df -h /mnt/immich-backup` in another terminal instead.

### 6.5 Verify copying

This checks that the backup is complete without copying anything:

```bash
rsync -an --itemize-changes /mnt/immich-hdd/ /mnt/immich-backup/media/
```
`-n` is a dry run. It only lists the files that differ between the media disk and the backup. If nothing is listed, the backup has everything.

```bash
echo $?
```
Empty output and exit code `0` mean the copy is complete. For a stricter check, add `-c` to compare checksums (it reads all data and takes hours).

![Finished backup](https://github.com/MikeMilenk/Immich-Backing-Up/blob/d5eefd8f1b9cb776c400fdeddc3b29ee3b47afe7/images/6.5.2%20-%20Finished%20backup.png)

The backup disk (`sdf`) shows about 10 GB more because it also holds the system backup from Proxmox (the `dump` folder).

**6.6 Unmount and finish**

Unmounting makes sure everything is written to the disk and nothing is still in use, so it's safe to unplug. It also hands the media disk back to the VM: it must not stay mounted on the host when the VM starts!

```bash
umount /mnt/immich-hdd
```
```bash
umount /mnt/immich-backup
```
```bash
df -h | grep immich
```

![Unmounting disks](https://github.com/MikeMilenk/Immich-Backing-Up/blob/8eadbffa7de5d74164f48afc229da8c046fe8c08/images/6.6%20-%20Unmounting%20disks.png)

When `df` prints nothing, both disks are unmounted. Unplug the backup disk and **start the VM**. If you get `target is busy`, run `cd /` and try again.

---

## 7. What ends up on the backup disk

```
/mnt/immich-backup
├── dump/          # Proxmox VM backup (.vma.zst), restorable only on Proxmox
├── media/         # exact copy of the media disk
│   └── library/
│       ├── upload/          # original photos and videos
│       ├── thumbs/
│       ├── encoded-video/
│       ├── profile/
│       └── backups/         # Immich daily database dumps
└── lost+found/    # created automatically by ext4, leave it alone
```

---

## 8. Restore

> **Note:** restore steps haven't been tested yet. Guide is in progress...
