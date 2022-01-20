---
title: "Creating an encrypted RAID5 on Linux using mdadm, LUKS and LVM"
date: 2022-01-20T18:31+01:00
draft: true
---

Having a bunch of unused 1 To HDD and my NAS getting full I decided to take extra steps ensuring my data were safe.

## Stuff I already had

* Custom NAS running Arch Linux (just an old PC) with 2*1 To HDD
* 2 unused 1 To HDD

## The idea

Having 4 1 To HDD I could potentialy create a RAID 5 with 3 To of usable space ensuring my data were a little more safe.

The problem was that I needed to backup all the stuff from the 4 HDD before doing anything. So I brough a brand new 3 To HDD just to do that.

The new drive will serve 2 purposes: backup everything to allow creation of the RAID5 and having a cold backup in case of RAID failure (unwanted rm -rf).

## Doing it

First step was to backup everything in the same place: the new 3 To HDD. It's a long a tedious task considering I had a lot of duplicates datas in different places. Using rsync helps a bit.