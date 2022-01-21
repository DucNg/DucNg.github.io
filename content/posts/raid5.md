---
title: "Creating an encrypted RAID5 on Linux using mdadm, LUKS and LVM"
date: 2022-01-20T18:31:00+01:00
---

Having a bunch of unused 1 To HDD and my NAS getting full I decided to take extra steps ensuring my data were safe.

## Stuff I already had

* Custom NAS running Arch Linux (just an old PC) with 2*1 To HDD
* 2 unused 1 To HDD

## The idea

Having 4 1 To HDD I could potentialy create a RAID 5 with 3 To of usable space ensuring my data were a little more safe.

The problem was that I needed to backup all the stuff from the 4 HDD before doing anything. So I brough a brand new 3 To HDD just to do that.

The new drive will serve 2 purposes: backup everything to allow creation of the RAID5 and having a cold backup in case of RAID failure (or unwanted rm -rf).

## First step: backup

First step was to backup everything in the same place: the new 3 To HDD. It's a long a tedious task considering I had a lot of duplicates datas in different places. Using rsync helps a bit.

To backup the existing NAS I used the -a option in rsync that keeps files ownerships and rights and also copy symlinks. Using this command I could backup the entire Linux system and save time not having to reinstall everything.

From an ArchLinux live ISO on the NAS:


```sh
# Partition the new 3 To HDD
cfdisk /dev/sdX
# Format the new 3 To HDD to ext4
mkfs.ext4 /dev/sdX1
# Mount the new disk
mkdir /mnt/backup
mount /dev/sdX1 /mnt/backup
# Mount the 2 existing disks
mkdir /mnt/old_system
mount /dev/sdY /mnt/old_system
mount /dev/sdZ /mnt/old_system/<mount_point>
# Backup everything
rsync --progress -a /mnt/old_system /mnt/backup
```

Backuping the 2 other drives was mostly a similar process except I ensured using the -u to avoid copying existing data.

## Second step: Creating the RAID

Connect the 5 drives to the NAS. Considering my old 1 To drives are /dev/sda, /dev/sdb, /dev/sdc, /dev/sdd and the the 3 To HDD is /dev/sde

Create a single partition on each drives leaving about 100 Mo of free space at the end in case every drive don't have the exact same size.
The goal is to have partitions of the exact same size even if there is some empty space.

Set every drives to GPT and partition type to Linux RAID

```sh
cfdisk -z /dev/sda
cfdisk -z /dev/sdb
cfdisk -z /dev/sdc
cfdisk -z /dev/sdd
```

Then we can processed creating the RAID

```sh
mdadm --create --verbose --level 5 --raid-devices=4 /dev/md/<RAID NAME> /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1
```

We can now encrypt the whole RAID.

### Encrypting the RAID using LUKS

```sh
cryptsetup luksFormat /dev/md/<RAID NAME>
```

Open the LUKS volume

```sh
cryptsetup open /dev/md/<RAID NAME> cryptroot
```

We now have this 3 To block device: /dev/mapper/cryptroot

If we want to create partitions the usual way won't work since we only have access to a single encrypted block device.
What we can use however is logical volume management, namely LVM.

### Partition the RAID using LVM

```sh
pvcreate /dev/mapper/cryptroot
vgcreate volgroup /dev/mapper/cyptroot
lvcreate -L 20G volgroup -n root
lvcreate -L 2G volgroup -n swap
lvcreate -l +100%FREE volgroup -n home
```
Format every partitions

```sh
mkfs.ext4 -L root /dev/mapper/volgroup-root
mkfs.ext4 -L home /dev/mapper/volgroup-home
mkswap /dev/mapper/volgroup-swap
```

Mount everything

```sh
mount /dev/mapper/volgroup-root /mnt
mkdir /mnt/home
mount /dev/mapper/volgroup-home /mnt/home
swapon /dev/mapper/volgroup-swap
```

### Install Linux

For me it's as easy as:

```sh
mount /dev/sde /media
rsync --progress -a /dev/media /mnt
```

Let Linux magic happen...

Create fstab and mdadm.conf

```sh
genfstab -U /mnt > /mnt/etc/fstab
mdadm --detail --scan >> /etc/mdadm.conf
```

## Thrid step: get Linux to boot

At this point my Linux is basically ready to rock (we have a fully usable chroot system). Except for one thing: I cannot boot into it.

Getting it to boot is no easy task: there is no way my BIOS is able to boot into the RAID directly.
The only thing my BIOS is capable of booting is single simple MBR device.
Since all my SATA ports were already used for the RAID, my only option was to use an old USB drive and install grub on it. Plug it in: /dev/sdf

### Prepare the USB stick

```sh
cfdisk -z /dev/sdf # make sure it's MBR
mkfs.ext4 -L boot /dev/sdf1
rm -rf /mnt/boot/* # remove previous GRUB installation
mount /dev/sdf1 /mnt/boot
arch-chroot /mnt # at this point we can chroot onto the new system
grub-install --target=i386-pc /dev/sdf # install grub on the USB stick
```

### Add needed hooks to initramfs

Edit HOOKS in /etc/mkinitcpio.conf: insert mdadm_udev, encrypt, lvm2 between block and filesystems:

```sh
HOOKS=(base udev ... block mdadm_udev encrypt lvm2 filesystems)
```

Build initramfs onto the USB stick:

```
mkinitcpio -P
```

### Configure GRUB

We want boot to happen in a very specific way since it needs to: assemble the RAID, open the LUKS volume and detect my LVM volume group. In this specific order.

GRUB helps a lot getting everything to work properly. The only thing you need is to edit GRUB_CMDLINE_LINUX in /etc/default/grub

```sh
GRUB_CMDLINE_LINUX="cryptdevice=/dev/md/<RAID NAME>:cryptroot root=/dev/mapper/volgroup-root"
```
The root= parameter specifies the device of the actual (decrypted) root file system, the cryptdevice= parameter make the system prompt for the passphrase to unlock the device containing the encrypted root.

Create grub config file:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Thanks to GRUB everything should work now. Change BIOS options to boot on the USB stick and we're good to go.