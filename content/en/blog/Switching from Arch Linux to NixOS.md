---
tags:
  - Arch Linux
  - Home Manager
  - Nix
  - NixOS
  - SOPS
date: '2025-05-16'
title: Switching from Arch Linux to NixOS
description: The switch to NixOS involved learning a whole new language. It may not be everyone's cup of tea, but I personally enjoyed the journey I took to get it running.
---

> I use Arch btw

Not any more.

During my last post about [deploying PeerTube](https://lyuk98.com/60610/self-hosting-peertube-with-tailscale "Self-hosting PeerTube with Tailscale"), I considered using NixOS for the server. I eventually decided against it due to a steep learning curve, but Nix's goal of declarative environment made me eager to try it out one day.

Arch Linux has been the daily driver on my laptop for well over two years. I had no complaints with it, receiving timely updates while still being a generally stable system. However, even though reproducibility is not exactly what I need in such a device, the ability to roll back with ease as well as declare my setup in a Git repository was appealing to me.

# The previous setup

I use a Framework Laptop 13 with a 12th-generation Intel Core processor. With a two-terabyte SSD, it was a home to two operating systems, Arch Linux and Windows <s>which I blame BattlEye for</s>.

```
[lyuk98@framework ~]$ sudo parted --list
Model: WDS200T1X0E-00AFY0 (nvme)
Disk /dev/nvme0n1: 2000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  1075MB  1074MB  fat32                                      boot, esp
 2      1075MB  1680GB  1679GB
 3      1680GB  1680GB  16.8MB               Microsoft reserved partition  msftres, no_automount
 4      1680GB  2000GB  319GB                Basic data partition          msftdata, no_automount
 5      2000GB  2000GB  675MB   ntfs                                       hidden, diag, no_automount


```

It uses full-disk encryption with LUKS, containing a single Btrfs partition for everything Linux. TPM is used to automatically decrypt the device on boot, and Secure Boot is enabled with my own keys registered at the firmware.

```
[lyuk98@framework ~]$ cat /etc/kernel/cmdline | tr ' ' '\n'
quiet
splash
vt.global_cursor_default=0
tpm_tis.interrupts=0
rd.luks.uuid=95582132-bd92-4355-8afc-f17636c73869
rd.luks.options=95582132-bd92-4355-8afc-f17636c73869=luks,discard,x-initrd.attach,tpm2-device=auto
root=UUID=391379dd-32a7-4100-8062-7492eb227b4b
rootfstype=btrfs
rootflags=defaults,rw,compress=zstd,discard=async,subvol=/@
psi=1
lockdown=integrity
```

The previous setup mounted a lot of subvolumes inside `/var`. It was a result of following [an old default layout for OpenSUSE](https://en.opensuse.org/SDB:BTRFS#Old_/var/*_subvolume_layout_(pre_Jan_2018) "SDB:BTRFS - openSUSE Wiki"), because I did not want to disable copy-on-write for the whole `/var` back then.

```
[lyuk98@framework ~]$ findmnt
TARGET                                        SOURCE                                                                         FSTYPE          OPTIONS
/                                             /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/@]                      btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=261,subvol=/@
├─/home                                       /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/home]                   btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=256,subvol=/home
├─/dev                                        devtmpfs                                                                       devtmpfs        rw,nosuid,size=4096k,nr_inodes=4045005,mode=755,inode64
│ ├─/dev/mqueue                               mqueue                                                                         mqueue          rw,nosuid,nodev,noexec,relatime
│ ├─/dev/hugepages                            hugetlbfs                                                                      hugetlbfs       rw,nosuid,nodev,relatime,pagesize=2M
│ ├─/dev/shm                                  tmpfs                                                                          tmpfs           rw,nosuid,nodev,inode64
│ └─/dev/pts                                  devpts                                                                         devpts          rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000
├─/sys                                        sysfs                                                                          sysfs           rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/tracing                       tracefs                                                                        tracefs         rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/debug                         debugfs                                                                        debugfs         rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/fuse/connections                  fusectl                                                                        fusectl         rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security                      securityfs                                                                     securityfs      rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup                            cgroup2                                                                        cgroup2         rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot
│ ├─/sys/fs/pstore                            pstore                                                                         pstore          rw,nosuid,nodev,noexec,relatime
│ ├─/sys/firmware/efi/efivars                 efivarfs                                                                       efivarfs        rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/bpf                               bpf                                                                            bpf             rw,nosuid,nodev,noexec,relatime,mode=700
│ └─/sys/kernel/config                        configfs                                                                       configfs        rw,nosuid,nodev,noexec,relatime
├─/proc                                       proc                                                                           proc            rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc                  systemd-1                                                                      autofs          rw,relatime,fd=39,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=25617
│   └─/proc/sys/fs/binfmt_misc                binfmt_misc                                                                    binfmt_misc     rw,nosuid,nodev,noexec,relatime
├─/run                                        tmpfs                                                                          tmpfs           rw,nosuid,nodev,size=6508296k,nr_inodes=819200,mode=755,inode64
│ ├─/run/user/1000                            tmpfs                                                                          tmpfs           rw,nosuid,nodev,relatime,size=3254144k,nr_inodes=813536,mode=700,uid=1000,gid=1000,inode64
│ │ ├─/run/user/1000/gvfs                     gvfsd-fuse                                                                     fuse.gvfsd-fuse rw,nosuid,nodev,relatime,user_id=1000,group_id=1000
│ │ └─/run/user/1000/doc                      portal                                                                         fuse.portal     rw,nosuid,nodev,relatime,user_id=1000,group_id=1000
│ ├─/run/credentials/systemd-journald.service tmpfs                                                                          tmpfs           ro,nosuid,nodev,noexec,relatime,nosymfollow,size=1024k,nr_inodes=1024,mode=700,inode64,noswap
│ └─/run/credentials/systemd-resolved.service tmpfs                                                                          tmpfs           ro,nosuid,nodev,noexec,relatime,nosymfollow,size=1024k,nr_inodes=1024,mode=700,inode64,noswap
├─/opt                                        /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/opt]                    btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=257,subvol=/opt
├─/root                                       /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/root]                   btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=258,subvol=/root
├─/srv                                        /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/srv]                    btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=259,subvol=/srv
├─/usr/local                                  /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/usr/local]              btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=260,subvol=/usr/local
├─/var/crash                                  /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/crash]              btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=264,subvol=/var/crash
├─/var/cache                                  /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/cache]              btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=263,subvol=/var/cache
├─/var/lib/flatpak                            /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/flatpak]        btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=274,subvol=/var/lib/flatpak
├─/var/lib/libvirt/images                     /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/libvirt/images] btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=265,subvol=/var/lib/libvirt/images
├─/var/lib/machines                           /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/machines]       btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=2646,subvol=/var/lib/machines
├─/var/lib/mailman                            /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/mailman]        btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=273,subvol=/var/lib/mailman
├─/var/lib/mariadb                            /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/mariadb]        btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=268,subvol=/var/lib/mariadb
├─/var/lib/mysql                              /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/mysql]          btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=269,subvol=/var/lib/mysql
├─/var/lib/named                              /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/named]          btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=267,subvol=/var/lib/named
├─/var/log                                    /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/log]                btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=271,subvol=/var/log
├─/var/lib/pgsql                              /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/lib/pgsql]          btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=270,subvol=/var/lib/pgsql
├─/var/spool                                  /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/spool]              btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=266,subvol=/var/spool
├─/var/opt                                    /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/opt]                btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=272,subvol=/var/opt
├─/var/swap                                   /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/swap]               btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=275,subvol=/var/swap
├─/var/tmp                                    /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869[/var/tmp]                btrfs           rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvolid=262,subvol=/var/tmp
├─/boot                                       /dev/nvme0n1p1                                                                 vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro,discard
└─/tmp                                        tmpfs                                                                          tmpfs           rw,nosuid,nodev,nr_inodes=1048576,inode64
```

# Installing NixOS on a virtual machine

Even though I have decided to install NixOS right on the bare metal, writing Nix files from scratch, without proper knowledge, was not going to be easy. Therefore, I first installed NixOS on a virtual machine using [their installation media](https://nixos.org/download/#nixos-iso "Download | Nix & NixOS") to see its configuration.

![NixOS Installer showing summary of the installation procedure](https://images.lyuk98.com/fc844255-4654-4269-b5e5-c7bbb1f16aae.avif "NixOS installation summary")

Installing NixOS using their graphical installer was easy. When the system was ready, I went to `/etc/nixos` and read two Nix files `configuration.nix` and `hardware-configuration.nix`.

```nix
{ config, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "nixos";
  networking.networkmanager.enable = true;

  time.timeZone = "Asia/Seoul";
  i18n.defaultLocale = "en_GB.UTF-8";
  i18n.extraLocaleSettings = {
    LC_ADDRESS = "en_GB.UTF-8";
    LC_IDENTIFICATION = "en_GB.UTF-8";
    LC_MEASUREMENT = "en_GB.UTF-8";
    LC_MONETARY = "en_GB.UTF-8";
    LC_NAME = "en_GB.UTF-8";
    LC_NUMERIC = "en_GB.UTF-8";
    LC_PAPER = "en_GB.UTF-8";
    LC_TELEPHONE = "en_GB.UTF-8";
    LC_TIME = "en_GB.UTF-8";
  };

  services.xserver.enable = true;

  services.xserver.displayManager.gdm.enable = true;
  services.xserver.desktopManager.gnome.enable = true;

  services.xserver.xkb = {
    layout = "us";
    variant = "";
  };

  services.printing.enable = true;

  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
  };

  users.users.lyuk98 = {
    isNormalUser = true;
    description = "lyuk98";
    extraGroups = [ "networkmanager" "wheel" ];
    packages = with pkgs; [ ];
  };

  programs.firefox.enable = true;

  nixpkgs.config.allowUnfree = true;
  environment.systemPackages = with pkgs; [ ];
  system.stateVersion = "24.11";
}
```

```nix
{ config, lib, pkgs, modulesPath, ... }:

{
  imports = [ (modulesPath + "/profiles/qemu-guest.nix") ];

  boot.initrd.availableKernelModules =
    [ "ahci" "xhci_pci" "virtio_pci" "sr_mod" "virtio_blk" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ "kvm-intel" ];
  boot.extraModulePackages = [ ];

  fileSystems."/" = {
    device = "/dev/disk/by-uuid/23c86168-05fe-4e13-90b4-0e12d24b7ace";
    fsType = "ext4";
  };

  fileSystems."/boot" = {
    device = "/dev/disk/by-uuid/FF2C-9E3B";
    fsType = "vfat";
    options = [ "fmask=0077" "dmask=0077" ];
  };

  swapDevices = [ ];

  networking.useDHCP = lib.mkDefault true;

  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
}
```

I later referred to these configuration files several times.

# Writing Nix files

Studying [basics of the Nix language](https://nix.dev/tutorials/nix-language.html "Nix language basics — nix.dev  documentation") could have been a better way to start for someone else, but I first tried to see how it was used in practice. I came across [a repository with templates](https://github.com/Misterio77/nix-starter-configs "Misterio77/nix-starter-configs: Simple and documented config templates to help you get started with NixOS + home-manager + flakes. All the boilerplate you need!"), together with [the author's personal configurations](https://github.com/Misterio77/nix-config "Misterio77/nix-config: Personal nixos and home-manager configurations."), which became a great starting point.

A month later, the device was finally able to boot from my configurations. I have learned a lot, and the following are some of them that I consider important.

## Partitioning

My aim during the switch to NixOS was to keep `/home` intact. To do so, I decided to create subvolumes inside the existing Btrfs partition without reformatting.

The idea of [impermanence](https://wiki.nixos.org/wiki/Impermanence "Impermanence - NixOS Wiki") came across while I was searching for possible setups. Even though I did not use [the module](https://github.com/nix-community/impermanence "nix-community/impermanence: Modules to help you handle persistent state on systems with ephemeral root storage [maintainer=@talyz]") at the end, partly due to my unwillingness to declare specific paths to preserve, I still applied the idea of impermanent system by mounting `/` to `tmpfs`.

> Such a setup is possible because NixOS only needs `/boot` and `/nix` in order to boot, all other system files are simply links to files in `/nix`. `/boot` and `/nix` still need to be stored on a hard drive or SSD.

The following is [the file system configuration](https://github.com/lyuk98/nixos-config/blob/a1d2d2fae42ae6155955422cf937b6442430cb44/hosts/framework/hardware-configuration.nix "nixos-config/hosts/framework/hardware-configuration.nix at a1d2d2fae42ae6155955422cf937b6442430cb44 · lyuk98/nixos-config") used for the current system state. I decided to preserve `/var` across reboots to keep some persistent data.

```nix
# File systems to be mounted
fileSystems = {
  "/boot" = {
    device = "/dev/disk/by-uuid/090C-E895";
    fsType = "vfat";
  };

  "/" = {
    device = "none";
    fsType = "tmpfs";
    options = [
      "defaults"
      "size=20%"
      "mode=755"
    ];
  };

  "/home" = {
    device = "/dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869";
    fsType = "btrfs";
    options = [
      "defaults"
      "subvol=home"
      "compress=zstd"
    ];
  };

  "/nix" = {
    device = "/dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869";
    fsType = "btrfs";
    options = [
      "defaults"
      "subvol=nix"
      "compress=zstd"
    ];
  };

  "/var" = {
    device = "/dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869";
    fsType = "btrfs";
    options = [
      "defaults"
      "subvol=var"
      "compress=zstd"
    ];
  };
};
```

## SOPS

My goal was to define as many parts of the system as possible. I could set sensitive information like passwords as well, but I did not think storing them in plaintext is a good idea. [The configuration I referred to](https://github.com/Misterio77/nix-config "Misterio77/nix-config: Personal nixos and home-manager configurations.") have used [sops-nix](https://github.com/Mic92/sops-nix "Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops") to encrypt them, so I also decided to do so.

There are two methods of encryption: GPG and [age](https://github.com/FiloSottile/age "FiloSottile/age: A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability."). I ended up not using GPG, but I nevertheless prepared both methods.

I first retrieved the fingerprint of my personal GPG key.

```
[lyuk98@framework ~]$ echo $(gpg --fingerprint D60E735C | head -2 | tail -1 | tr -d '[:space:]')
270CB11B1189E79A17DCB7831BDAFDC5D60E735C
```

The age key was created as well.

```
[lyuk98@framework nixos-config]$ mkdir -p ~/.config/sops/age
[lyuk98@framework nixos-config]$ nix-shell -p age --run "age-keygen -o ~/.config/sops/age/keys.txt"
Public key: age14edzmqc4r07gp9lkj8z4gchccs373s8lcdrw69d6964tallpuuzqausgmk
```

Public keys were then placed into `.sops.yaml` [at the root of the repository](https://github.com/lyuk98/nixos-config/blob/a1d2d2fae42ae6155955422cf937b6442430cb44/.sops.yaml "nixos-config/.sops.yaml at a1d2d2fae42ae6155955422cf937b6442430cb44 · lyuk98/nixos-config").

```yaml
keys:
  # Users
  - &users:
    - &lyuk98 270CB11B1189E79A17DCB7831BDAFDC5D60E735C
  # Hosts
  - &hosts:
    - &framework age14edzmqc4r07gp9lkj8z4gchccs373s8lcdrw69d6964tallpuuzqausgmk

creation_rules:
  # Secrets specific to host "framework"
  - path_regex: hosts/framework/secrets.ya?ml$
    key_groups:
    - age:
      - *framework
      pgp:
      - *lyuk98

  # Secrets for user "lyuk98" to be used across hosts
  - path_regex: hosts/common/users/lyuk98/secrets.ya?ml$
    key_groups:
    - age:
      - *framework
      pgp:
      - *lyuk98
```

It was now time to create secrets. I started with password of my user. With `mkpasswd`, a hashed password was created.

(The hash I actually used is different; the result below is the hash of the password `password`.)

```
[lyuk98@framework nixos-config]$ mkpasswd
Password: 
$y$j9T$O5BvDo7mdLNITy4otZk3W0$eFEFPuEdKJowvw7MfzfbIflbHV5vSreWXCwPaH34Td5
```

The hashed password was then written to `secrets.yaml` using `sops`.

```
[lyuk98@framework nixos-config]$ nix-shell -p sops --run "sops hosts/common/users/lyuk98/secrets.yaml"
```

Issuing `sops` opened a text editor, where I put the following:

```yaml
lyuk98-password: $y$j9T$O5BvDo7mdLNITy4otZk3W0$eFEFPuEdKJowvw7MfzfbIflbHV5vSreWXCwPaH34Td5
```

The resultant file was automatically encrypted with some apparent metadata attached.

```yaml
lyuk98-password: ENC[AES256_GCM,data:HqFR/AgXhTkcZwLdcn+vM8QIDvbKa8oUDdBRUsbmUcAlWTfhd8UdnTsvUEtWQkU4j3PG27Yo9GVoCRulDkbyKfgyggInWfhJ1Q==,iv:jmq7cw5SB1TokApgdY6tIqdXVrIi7zb6ur62m9BtKF4=,tag:S3narIG/Fm4HfXgItb4VXQ==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age14edzmqc4r07gp9lkj8z4gchccs373s8lcdrw69d6964tallpuuzqausgmk
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSBMN0tCWmxyb09aaWhBaGFo
            NDcvUTU4ekJSQlQ1WFVmOUcvQXJ6OTdHVVFrClEvOWp5SmR6R0pEc1V6U3FLaktN
            djFaSmlVY0kzUFhoTnBvajlSNEk2czAKLS0tIDdQS2FzVHFxRkxXT1FVQXNlQ1Uw
            L0dRWFJwamZJU2NwTlYxU2RjVFBCN0UKIvCJlOGnDpbRCAKau7e+ijfc9NgRA+uF
            3vKSsSSOeJih4uccoKT2lfPgP4T4vzcJnyD9vz7S7Nmpurvedkla+g==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2025-05-12T08:09:45Z"
    mac: ENC[AES256_GCM,data:v+b37D4dEVDr78/E4wMWAM2pH2D+m55hTAsXFmUt2fRAmeHzrVDi4PWItG4Mm3AHJ9KSgO2Aj2WE5IFg0OSEI4ehJJGL/hp1L61YGwXOXrZ9+iiBda/fiQwAVIuEojHTkxxJcyRR1blUoMScBqSUOg3SihU579oimC1J06aiH+I=,iv:D2LjheIGKK2F81I8mGPf74PpFBZYVCAnNP8W6Y70laI=,tag:hDau1W8H8fKVWH3D0qg2uw==,type:str]
    pgp:
        - created_at: "2025-05-12T08:09:20Z"
          enc: |-
            -----BEGIN PGP MESSAGE-----

            hF4D4TeHAuIWmuESAQdASOhjLuuIuejuTAiW0OtSqTCmk0w1MvWupjo9+6Pa3UUw
            YIzRTwn4KT/3cXGnIJ3DQsvPMuaMkzIgb1AFx7jhonYdABS0mDOn5c4X+7d7c1uX
            0lEBTdohXDwViilR0eaUmabxjPbVP7GTTzxVYqVAcoWHNrD4X43nBYHpZvCE8PVZ
            J9ByWvrHRdDKvP07sX8phKPe+aqBMdE5Bs6jGZS3p4PgsTg=
            =aJCD
            -----END PGP MESSAGE-----
          fp: 270CB11B1189E79A17DCB7831BDAFDC5D60E735C
    unencrypted_suffix: _unencrypted
    version: 3.9.4
```

I added another secret in a different directory to include `machine-id`.

```
[lyuk98@framework nixos-config]$ nix-shell -p sops --run "sops hosts/framework/secrets.yaml"
```

```yaml
machine-id: ENC[AES256_GCM,data:KjgyeUiMflGF0u2uvCaKfvw7bKpNH29tVRxaOzt9tfY=,iv:gpc9+uwpNtuNU4SG7S5XCDRU4oWBrqRZdXr7fhDmVKA=,tag:QzNtVBmFV73w/rldwLNP2Q==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age:
        - recipient: age14edzmqc4r07gp9lkj8z4gchccs373s8lcdrw69d6964tallpuuzqausgmk
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+IFgyNTUxOSAxTlBGZTRkU280QzdTRDda
            NnhmVksxajZkcTdjVGM4RFkyR3VOYklGcFJvCk9pZ0xubjFOdzg4SFhUMEdJck42
            UXZWbGpKM1E3MXh5ZFRCUkZFNG9pMncKLS0tIHVPRnZ4Um9DUVlpZjlFa1laZ1dE
            R01HbWtQbWFLUml2YW5PTWF4bHBGUHcKuAKSzrOaoI0E8gtryfjQDtTka3IEB15M
            8CwBCR/iaXoLKiPhcYSbTMQBiUB50Ah52FcHJ2KLyjNlGjmU+psunQ==
            -----END AGE ENCRYPTED FILE-----
    lastmodified: "2025-05-12T08:18:04Z"
    mac: ENC[AES256_GCM,data:A8laj76aYG+K6rP4KsVssJsdS2y0/5j1Dtj20ORr0Hxdh9JHAikFF6mrXEGZ2QBDE8qhXKoblpfE8yEBr1K9UPmrBbHEfEVyc4mXOSVa6oqIBe4LmEM7ckU7AfRmj1ph9/f16A7os1Vh9HPwaGhFqan9lnCj5S7xOzS4jiAb3LY=,iv:iAhuPqNq9hwfs4Btj16S/WpWJ0G4QkUkKTYUPmOdLag=,tag:SLWAZ2MpagHPs4+Q25pYcA==,type:str]
    pgp:
        - created_at: "2025-05-12T08:17:34Z"
          enc: |-
            -----BEGIN PGP MESSAGE-----

            hF4D4TeHAuIWmuESAQdAtgfUKZnJXtiu2NZILkPtRpzo0gxjF4gEMGEi5zDDGmcw
            dXmXFoA0mvh3pU7BkIhoA40faJ/klhLw3pu1GLxOgix+So73SZuE8thWv+oYslwG
            0lEBKTr+3DokW1Tb5Jq3iKcpWeT47t2obBxBHNkfO5jaARPHTR8+8pwZrODOaT7e
            3Se3PP7LmI4myOiizNpn0gjkQyqc6GsAQzNCFCG5dXETWvs=
            =5exl
            -----END PGP MESSAGE-----
          fp: 270CB11B1189E79A17DCB7831BDAFDC5D60E735C
    unencrypted_suffix: _unencrypted
    version: 3.9.4
```

The secrets are there, but how are they going to be decrypted? It took a while before I could finally understand that they are decrypted upon every activation.

Thinking the decryption would take place every boot, I needed a place to store the keys. Since `/var` will be preserved, I put my age key there.

```
[lyuk98@framework ~]$ sudo mkdir -p /var/lib/sops-nix
[lyuk98@framework ~]$ sudo cp ~/.config/sops/age/keys.txt /var/lib/sops-nix/keys.txt
[lyuk98@framework ~]$ sudo chmod 0700 /var/lib/sops-nix
```

[An appropriate configuration](https://github.com/lyuk98/nixos-config/blob/a1d2d2fae42ae6155955422cf937b6442430cb44/hosts/common/core/sops.nix "nixos-config/hosts/common/core/sops.nix at a1d2d2fae42ae6155955422cf937b6442430cb44 · lyuk98/nixos-config") was also made to look for an age key in that location.

```nix
{ inputs, lib, ... }:
{
  imports = [ inputs.sops-nix.nixosModules.sops ];

  # Specify path of the age key to decrypt secrets with
  # This file needs to be present to decrypt secrets during activation
  sops.age.keyFile = lib.mkDefault "/var/lib/sops-nix/keys.txt";
}
```

## Declarative Wi-Fi configurations for NetworkManager

Other configurations I came across used systemd-networkd or wpa_supplicant for networking, but I wanted to continue using NetworkManager, especially due to its integration with GNOME. It apparently became possible to declare networks using configuration `networking.networkmanager.ensureProfiles.profiles`, but a problem with it was that I had no apparent way to hide SSID.

Until I find a better way, I decided to simply encrypt entire `.nmconnection` files. To prevent SSIDs from being visible, I renamed each connection with its UUID property.

```
[lyuk98@framework ~]$ sudo cp --recursive /etc/NetworkManager/system-connections .
[lyuk98@framework ~]$ sudo chown --recursive lyuk98 system-connections/
[lyuk98@framework ~]$ cd system-connections/
[lyuk98@framework system-connections]$ for connection in *.nmconnection; do mv "$connection" $(grep --only-matching --perl-regexp '(?<=uuid=).*' "$connection").nmconnection; done
```

`.sops.yaml` was then updated to allow encryption of those files.

```yaml
creation_rules:
  # ...

  # NetworkManager connections
  - path_regex: hosts/common/optional/system-connections/.+\.nmconnection$
    key_groups:
    - age:
      - *framework
      pgp:
      - *lyuk98
```

After definition of a creation rule, the files were placed and encrypted.

```
[lyuk98@framework nixos-config]$ cp ~/system-connections/* hosts/common/optional/system-connections/
[lyuk98@framework nixos-config]$ for connection in hosts/common/optional/system-connections/*.nmconnection; do nix-shell -p sops --run "sops encrypt --in-place $connection"; done
```

Since `system-connections` directory was to be only accessible by `root`, I manually ensured its permission with `systemd-tmpfiles`. I later reverted the change after realising that it is [done automatically](https://github.com/NixOS/nixpkgs/blob/69b630d89352730abb90be82d448af1cb5cac190/nixos/modules/services/networking/networkmanager.nix#L591 "nixpkgs/nixos/modules/services/networking/networkmanager.nix at 69b630d89352730abb90be82d448af1cb5cac190 · NixOS/nixpkgs"), however.

## Secure Boot with Lanzaboote

Despite its criticisms, I wanted to continue using the system with Secure Boot enabled. I used [Lanzaboote](https://github.com/nix-community/lanzaboote "nix-community/lanzaboote: Secure Boot for NixOS [maintainers=@blitz @raitobezarius @nikstur]"), as most guides for NixOS mentioned it.

[The quick start guide for Lanzaboote](https://github.com/nix-community/lanzaboote/blob/e65210d77b44ef1b45505f4b22c1365c1371444b/docs/QUICK_START.md "lanzaboote/docs/QUICK_START.md at e65210d77b44ef1b45505f4b22c1365c1371444b · nix-community/lanzaboote") used [sbctl](https://github.com/Foxboron/sbctl "Foxboron/sbctl: :computer: :key: Secure Boot key manager") for generation of Secure Boot keys. I followed both the aforementioned guide and [ArchWiki](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl "Unified Extensible Firmware Interface/Secure Boot - ArchWiki").

To start, I first [put the device in Setup Mode](https://community.frame.work/t/secureboot-setup-mode/14889/2 "Secureboot setup mode - Framework Laptop 13 - Framework Community"). I did not bother backing them up since I intended to replace my existing Secure Boot keys.

The existing pacman hook for issuing `kernel-install` was removed. It was convenient, but it is no longer needed.

```
[lyuk98@framework ~]$ sudo pacman -Rns pacman-hook-kernel-install
```

I then installed sbctl and made sure that the device is in Setup Mode.

```
[lyuk98@framework ~]$ sudo pacman -S sbctl
[lyuk98@framework ~]$ sbctl status
Installed:	✗ sbctl is not installed
Setup Mode:	✗ Enabled
Secure Boot:	✗ Disabled
Vendor Keys:	none
```

New set of keys were created and were automatically enrolled, together with the ones from Microsoft and the OEM (which is Framework in this case).

```
[lyuk98@framework ~]$ sudo sbctl create-keys
Created Owner UUID 5f0d4030-4870-4fce-ab2f-e25c34e60ebe
Creating secure boot keys...✓ 
Secure boot keys created!
[lyuk98@framework ~]$ sudo sbctl enroll-keys --microsoft --firmware-builtin
Enrolling keys to EFI variables...
With vendor keys from microsoft...
With vendor certificates built into the firmware...✓ 
Enrolled keys to the EFI variables!
```

It could now be seen that sbctl was ready.

```
[lyuk98@framework ~]$ sbctl status
Installed:	✓ sbctl is installed
Owner GUID:	5f0d4030-4870-4fce-ab2f-e25c34e60ebe
Setup Mode:	✓ Disabled
Secure Boot:	✗ Disabled
Vendor Keys:	microsoft builtin-db builtin-KEK
```

# Building from the configuration

When the code was ready enough for me, I tried to see what would happen if I were to build the system. I did not want to pull the trigger just yet, so I did `dry-build`.

```
[lyuk98@framework nixos-config]$ nix-shell -p nixos-rebuild --run "nixos-rebuild dry-build --flake ."
```

Doing so generated a lot of errors from my mistakes. After correcting them, however, something like the following could be seen:

```
[lyuk98@framework nixos-config]$ nix-shell -p nixos-rebuild --run "nixos-rebuild dry-build --flake ." 2>&1 | head -10
building the system configuration...
warning: Git tree '/home/lyuk98/nixos-config' is dirty
these 407 derivations will be built:
  /nix/store/01xk6zlzyv4kmkn3jfyfzvm73i3j11fz-nameservers.drv
  /nix/store/029p5zmbqykcb8w0hm7yv0hplmhdaj1z-evolution-with-plugins.drv
  /nix/store/03jpx2zdnkny8ja8sy9ikfx3ikkqzjqd-initrd-release.drv
  /nix/store/06j0rkhhw7qia6knms2wbfz510iy5h71-nixos-tmpfiles.d.drv
  /nix/store/wl4nlyaaz21hd28kixasixrf99gqy9i8-locales-setup-hook.sh.drv
  /nix/store/a534j021idk040xx503qpaa7q0c6x95w-glibc-locales-2.40-66.drv
  /nix/store/w8jrwsqi64x5knlp4lixs8kbvs2m2gs1-unit-script-nixos-activation-start.drv
```

It seemed the building process will probably succeed. It was now time to mess with the system.

# Installing NixOS

## Activating a new home environment with Home Manager

Together with NixOS configurations, I have also written [Home Manager](https://github.com/nix-community/home-manager "nix-community/home-manager: Manage a user environment using Nix  [maintainer=@rycee]") configurations to make a not-yet-completely-declarative home environment. Before switching to the new OS, I first switched to the new home.

```
[lyuk98@framework nixos-config]$ git add .
[lyuk98@framework nixos-config]$ nix-shell -p home-manager --run "home-manager switch --flake ."
```

The process was unexpectedly painless; I was already using Home Manager in Arch Linux.

## Activating a new OS

I once again used [the NixOS installation media](https://nixos.org/download/#nixos-iso "Download | Nix & NixOS") to boot into the live environment. Fedora Media Writer was used to write to my portable USB storage.

![Fedora Media Writer showing a successful result of writing the installation ISO image](https://images.lyuk98.com/d91c9b1d-fed5-4068-ab74-824755478330.avif "The media was ready")

Even though `sbctl status` previously said Secure Boot was disabled, it was in fact not the case. It prevented the live installer from booting up, so I went to firmware setup and temporarily disabled the security measure.

With the live environment up, I closed the installer window and issued a few commands to open my encrypted storage, which was then mounted to `/mnt`.

```
[nixos@nixos:~]$ sudo cryptsetup open /dev/disk/by-uuid/95582132-bd92-4355-8afc-f17636c73869 luks-95582132-bd92-4355-8afc-f17636c73869 --type luks2
[nixos@nixos:~]$ sudo mount /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869 /mnt
[nixos@nixos:~]$ cd /mnt/
```

I then renamed the existing `var` to `var2`. Since the data is preserved, I could still revert to Arch Linux if something was to go wrong.

```
[nixos@nixos:/mnt]$ sudo mv var var2
```

New subvolumes for the new OS were created. I disabled copy-on-write for the whole `var` this time.

```
[nixos@nixos:/mnt]$ sudo btrfs subvolume create nix
[nixos@nixos:/mnt]$ sudo btrfs subvolume create var
[nixos@nixos:/mnt]$ sudo chattr +C var
```

While I was at it, I also created a new swap file.

```
[nixos@nixos:/mnt]$ sudo mkdir -p var/swap
[nixos@nixos:/mnt]$ sudo btrfs filesystem mkswapfile --size 64g --uuid clear var/swap/swapfile
[nixos@nixos:/mnt]$ sudo chmod 0700 var/swap
```

NixOS uses `dd` and `mkswap` to create a new swap file if existing one's size is different from the configuration. I had a look into [how the size was checked](https://github.com/NixOS/nixpkgs/blob/374e6bcc403e02a35e07b650463c01a52b13a7c8/nixos/modules/config/swap.nix#L294) and applied the calculation to the one I have just created.

```
[nixos@nixos:/mnt]$ echo $(( $(sudo stat -c "%s" var/swap/swapfile 2>/dev/null || echo 0) / 1024 / 1024 ))
65536
```

`hardware-configuration.nix` was then updated to reflect the size of the new file.

```nix
# Use swap file
swapDevices = [{
  device = "/var/swap/swapfile";

  # Set swap file size in megabytes
  size = 65536;

  # Follow default discard policy by swapon(8)
  discardPolicy = "both";
}];
```

I was almost starting afresh, but I still wanted to preserve some data.

- I kept `/var/log` to keep system logs from the previous system (just in case I want to look at it for no reason).
- I kept `/var/lib/bluetooth` to preserve paired Bluetooth devices.
- I kept `/var/lib/sbctl` so that Lanzaboote can still sign EFI binaries using Secure Boot keys inside.
- I kept `/var/lib/sops-nix` to be able to decrypt secrets upon activation.
- I kept `/var/lib/tailscale` to preserve state data related to Tailscale.
- I kept `/var/lib/vnstat` to preserve previous network usage statistics.

```
[nixos@nixos:/mnt]$ sudo mkdir -p var/lib
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never var2/log var/
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never @/var/lib/bluetooth var/lib/
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never @/var/lib/sbctl var/lib/
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never @/var/lib/sops-nix var/lib/
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never @/var/lib/tailscale var/lib/
[nixos@nixos:/mnt]$ sudo cp --archive --reflink=never @/var/lib/vnstat var/lib/
```

With the actual installation ready, I `umount`ed the device and `mount`ed it again using different subvolumes.

```
[nixos@nixos:~]$ sudo umount /mnt
[nixos@nixos:~]$ sudo mount --mkdir /dev/disk/by-uuid/090C-E895 /mnt/boot
[nixos@nixos:~]$ sudo mount --mkdir -o defaults,compress=zstd,subvol=home /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869 /mnt/home
[nixos@nixos:~]$ sudo mount --mkdir -o defaults,compress=zstd,subvol=nix /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869 /mnt/nix
[nixos@nixos:~]$ sudo mount --mkdir -o defaults,compress=zstd,subvol=var /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869 /mnt/var
```

It was finally time to pull the trigger. The new system was then ready.

```
[nixos@nixos:/mnt/home/lyuk98/nixos-config]$ git add .
[nixos@nixos:/mnt/home/lyuk98/nixos-config]$ sudo nixos-install --flake .#framework --no-root-password
```

# Post-installation configurations

Booting into NixOS was successful, even though [forgetting to enable firmware](https://github.com/lyuk98/nixos-config/blob/a1d2d2fae42ae6155955422cf937b6442430cb44/hosts/common/core/firmware.nix "nixos-config/hosts/common/core/firmware.nix at a1d2d2fae42ae6155955422cf937b6442430cb44 · lyuk98/nixos-config") left me struggling without internet connection and graphics for a while. I re-enabled Secure Boot and automatic decryption with TPM afterwards.

```
[lyuk98@framework ~]$ sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/disk/by-uuid/95582132-bd92-4355-8afc-f17636c73869
[lyuk98@framework ~]$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/disk/by-uuid/95582132-bd92-4355-8afc-f17636c73869
```

With [my repository](https://github.com/lyuk98/nixos-config "lyuk98/nixos-config: NixOS Configurations") live and public, I could now build and switch using the new configuration without having them present on my device.

```
[lyuk98@framework:~]$ sudo nixos-rebuild boot --flake github:lyuk98/nixos-config
[lyuk98@framework:~]$ home-manager switch --flake github:lyuk98/nixos-config
```

## The clean-up

With NixOS working well, I no longer needed to keep the existing Arch Linux installation. I first removed the existing kernel images.

```
[lyuk98@framework:~]$ sudo rm /boot/EFI/Linux/*-6.12.28-1-lts.efi
[lyuk98@framework:~]$ sudo rm /boot/EFI/Linux/*-6.14.5-arch1-1.efi
```

I then deleted remaining data from the previous system. There was no going back any more.

```
[lyuk98@framework:~]$ sudo mount --mkdir /dev/mapper/luks-95582132-bd92-4355-8afc-f17636c73869 /mnt
[lyuk98@framework:~]$ cd /mnt/
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete opt
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete root
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete srv
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete usr/local
[lyuk98@framework:/mnt]$ sudo rmdir usr
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete --recursive @
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete var2/lib/libvirt/images
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete var2/lib/*
[lyuk98@framework:/mnt]$ sudo btrfs subvolume delete var2/*
[lyuk98@framework:/mnt]$ sudo rm --recursive var2
[lyuk98@framework:/mnt]$ cd
[lyuk98@framework:~]$ sudo umount /mnt
```

# What now?

![Running screenFetch on the new system](https://images.lyuk98.com/05362102-5aa9-4293-8c24-e3956e747b74.avif "Running screenFetch on the new system")

I have a lot to do, actually.

There are still many parts (mostly in Home Manager configurations) that I could not declaratively manage just yet. On top of that, even though I continued using Flatpak, it does not seem to fit well with the declarative nature of this system. Considering those concerns, the next step would be switching from Flatpak to Nixpkgs.

Switching to NixOS on my Raspberry Pi, [which currently hosts PeerTube](https://lyuk98.com/60610/self-hosting-peertube-with-tailscale "Self-hosting PeerTube with Tailscale"), is also something I plan to do in the future. It could be an opportunity to apply something new, such as declarative partitioning with [disko](https://github.com/nix-community/disko "nix-community/disko: Declarative disk partitioning and formatting using nix [maintainers=Lassulus Enzime iFreilicht Mic92 phaer]").

[The repository](https://github.com/lyuk98/nixos-config "lyuk98/nixos-config: NixOS Configurations") will continue to be maintained as long as I keep using NixOS. If I know more about Nix than I do now, it will probably look much more different than how it does at the time of writing.
