---
date: '2025-04-13'
title: Using GPT fdisk to recover corrupted partition table
description: I was not able to boot into Linux due to a corrupted partition table, which I tried to fix by using GPT fdisk.
---

I used Windows for a few hours on my laptop, which dual boots Arch Linux and Windows 11, and rebooted into Linux. However, after a while of unusual waiting, it failed to do so. It would have been nice to read the error first, but having previously experienced a similar problem, which once occurred when I first installed Windows 11 on top of the existing Linux installation <s>thanks to anti-cheat software</s>, I felt it has something to do with a damaged partition table.

# Preparing Fedora

On Windows, I used [Fedora Media Writer](https://fedoraproject.org/workstation/download#fedora-media-writer "Fedora Workstation | The Fedora Project") to write Fedora 41 into a blank microSD card and boot into it.

![Fedora Media Writer on the page named "Write Options". Three selections, which are "Version", "Hardware Architecture", and "USB Drive", are set with values "41", "Intel/AMD 64bit", and "SMI USB DISK (63.9 GB)", respectively. A checkbox with a label "Delete download after writing" is unchecked. Bottom row of the window has two buttons labelled "Previous" and "Write".](https://images.lyuk98.com/c785dd37-f170-4f89-aa4c-ddd2d7b786c2.avif "Writing Fedora 41 into a microSD card")

The device rebooted, and Fedora was up. I connected to the internet and installed GPT fdisk.

```
liveuser@localhost-live:~$ sudo dnf install gdisk
```

# GPT fdisk to the rescue

With `gdisk` ready, it was time to see what went wrong.

```
liveuser@localhost-live:~$ sudo gdisk /dev/nvme0n1
GPT fdisk (gdisk) version 1.0.10

Caution! After loading partitions, the CRC doesn't check out!
Warning! Main partition table CRC mismatch! Loaded backup partition table
instead of main partition table!

Warning! One or more CRCs don't match. You should repair the disk!
Main header: OK
Backup header: OK
Main partition table: ERROR
Backup partition table: OK

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: damaged

****************************************************************************
Caution: Found protective or hybrid MBR and corrupt GPT. Using GPT, but disk
verification and recovery are STRONGLY recommended.
****************************************************************************

Command (? for help): 
```

The output confirmed my suspicion: the partition table was damaged. Before doing anything further, I made sure that I know what I am doing.

```
Command (? for help): ?
b	back up GPT data to a file
c	change a partition's name
d	delete a partition
i	show detailed information on a partition
l	list known partition types
n	add a new partition
o	create a new empty GUID partition table (GPT)
p	print the partition table
q	quit without saving changes
r	recovery and transformation options (experts only)
s	sort partitions
t	change a partition's type code
v	verify disk
w	write table to disk and exit
x	extra functionality (experts only)
?	print this menu

Command (? for help): 
```

Recovery from this problem was what I was looking for, so I chose `r`.

```
Command (? for help): r

Recovery/transformation command (? for help): 
```

[The ArchWiki's guide](https://web.archive.org/web/20250412173417/https%3A%2F%2Fwiki.archlinux.org%2Ftitle%2FGPT_fdisk#Recover_GPT_header "GPT fdisk - ArchWiki") on recovery exists, but it only pertained to the recovery of headers. What I actually needed was the recovery of the partition table, though, so I first looked at available options.

```
Recovery/transformation command (? for help): ?
b	use backup GPT header (rebuilding main)
c	load backup partition table from disk (rebuilding main)
d	use main GPT header (rebuilding backup)
e	load main partition table from disk (rebuilding backup)
f	load MBR and build fresh GPT from it
g	convert GPT into MBR and exit
h	make hybrid MBR
i	show detailed information on a partition
l	load partition data from a backup file
m	return to main menu
o	print protective MBR data
p	print the partition table
q	quit without saving changes
t	transform BSD disklabel partition
v	verify disk
w	write table to disk and exit
x	extra functionality (experts only)
?	print this menu

Recovery/transformation command (? for help): 
```

Upon reading, it was obvious that `c` is the operation I needed to do.

```
Recovery/transformation command (? for help): c
Warning! This will probably do weird things if you've converted an MBR to
GPT form and haven't yet saved the GPT! Proceed? (Y/N): y

Recovery/transformation command (? for help): 
```

With the changes made, it was time to actually write them into the storage.

```
Recovery/transformation command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/nvme0n1.
The operation has completed successfully.
liveuser@localhost-live:~$ 
```

I then rebooted the system. Linux successfully booted up without any further action.

# Final thoughts

Now that this problem happened twice, I started wondering what messed up the partition table. Something could have happened during the installation of updates to Windows, but I was not able to find anything that supports this claim.

Nevertheless, I decided to back up the partition table. I installed the command-line utility and successfully created the backup.

```
[lyuk98@framework ~]$ sudo pacman -S gptfdisk
[lyuk98@framework ~]$ sudo sgdisk --backup=/tmp/nvme0n1 /dev/nvme0n1
The operation has completed successfully.
```

With other problems in hand, I did not bother to take a deeper look into this matter. Perhaps I will, if/when this problem happens again one day.
