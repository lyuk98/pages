---
date: '2025-11-05'
title: Building a project (#2) - Preparing a server
description: As an effort to host OpenStack, I prepared a device to run the service with.
slug: building-a-project-2-preparing-a-server
---

This is a part of series *Building a project*, where I try to make something using [OpenStack](https://www.openstack.org/ "Open Source Cloud Computing Infrastructure - OpenStack").

1. [Learning about OpenStack](https://lyuk98.com/60219/building-a-project-1-learning-about-openstack "Building a project (#1) - Learning about OpenStack")
2. Preparing a server
3. [Attempting to run Kubernetes and OpenStack](https://lyuk98.com/67660/building-a-project-3-attempting-to-run-kubernetes-and-openstack "Building a project (#3) - Attempting to run Kubernetes and OpenStack")

---

# Introduction

After a long, long hiatus, I decided give this project a push. Although I stopped after introducing myself to a very basic overview of OpenStack, I still heard about it from time to time.

Furthermore, I came across [a blog post](https://ubuntu.com/blog/kubernetes-vs-openstack "Kubernetes vs OpenStack: which one to choose? | Ubuntu") comparing Kubernetes and OpenStack, where I read a paragraph that intrigued me:

> ### How to stack the stack?
> 
> Wait  a second. So is the desired setup Kubernetes on OpenStack on Kubernetes?
> 
> You guessed right! Even though it sounds awkward, this architecture setup has the most advantages, effectively enabling you to combine the best of both worlds on one platform. [...]

It became less about which one to use, and more about how to use both.

# The environment

I happened to be in possession of an old Dell XPS 13, so I decided to use it for hosting all kinds of services. It has a decent performance despite its age, which is an obvious advantage over my Raspberry Pi.

The choice of the operating system was, obviously, NixOS. Setting my bias aside, all I need to enable Kubernetes is apparently just a few lines of code:

```nix
services.kubernetes = {
  roles = [
    "master"
    "node"
  ];
};
```

While I have not tried it myself yet, it looked like the easiest way for me to set up Kubernetes among the guides I have read online.

On top of server-related stuff, I also wanted the machine to have a desktop environment in case I happen to do casual web browsing there. My choice here was, just like what it has been since the past decade, [GNOME](https://www.gnome.org/ "GNOME -- An independent computing platform for everyone"). I might try [COSMIC](https://system76.com/cosmic "COSMIC") one day, however.

# Writing NixOS configuration

Like when I have [set up instances for Ente](https://lyuk98.com/65671/hosting-ente-with-terraform-vault-and-nixos "Hosting Ente with Terraform, Vault, and NixOS"), I wrote some configuration for the new device. As always, the starting point was `flake.nix`, where I [created a new NixOS host](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/flake.nix#L163-L166 "nixos-config/flake.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config").

```nix
nixosConfigurations = {
  # existing hosts

  # Dell XPS 13 9350
  xps13 = nixpkgs.lib.nixosSystem {
    modules = [ ./hosts/xps13 ];
    specialArgs = { inherit inputs outputs; };
  };
};
```

Other changes were mostly made within [the host-specific directory](https://github.com/lyuk98/nixos-config/tree/9c13b9c80d7a277a02e9c7dee35757ff48883e68/hosts/xps13 "nixos-config/hosts/xps13/default.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config"); some notable ones are mentioned below.

## Preparing secrets

Just like my other devices, the new one also required some secrets. As usual, [sops-nix](https://github.com/Mic92/sops-nix "Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops") was my choice for storing them encrypted.

For some reason, I wanted to specify `machine-id`; a new one was created by running `dbus-uuidgen`.

```
[lyuk98@framework:~]$ dbus-uuidgen
7e7f3bdd40f715ee577580a868f47504
```

I wanted the device to connect to tailnet automatically. Like what I would do to servers, I [created an OAuth credential](https://login.tailscale.com/admin/settings/trust-credentials/add "New credential - Tailscale").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/aa362569-d6cd-4d99-88d0-d81f8ae5646c.avif">
  <img src="https://images.lyuk98.com/92ea56d9-68e1-4c7a-8a18-8f3cc066a773.avif" alt="A dialog for adding a new credential at Tailscale. Checkbox &quot;Write&quot; is checked for the &quot;Auth Keys&quot; scope." title="Adding a new credential">
</picture>

They were then written to `secrets.yaml` inside the host-specific directory. Please note that the values below are not the actual data that ended up in my repository.

```yaml
machine-id: 7e7f3bdd40f715ee577580a868f47504
tailscale-auth-key: tskey-client-kiVq4NfjyH11CNTRL-CuSLnQJwGL7vj61JRFG3L7eMKZa25oyH
```

To encrypt the data, I created an [age](https://github.com/FiloSottile/age "FiloSottile/age: A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability.") key for the host. It was saved to a temporary directory for now.

```
[lyuk98@framework:~]$ nix shell nixpkgs#age
[lyuk98@framework:~]$ tmp=$(mktemp --directory)
[lyuk98@framework:~]$ age-keygen --output $tmp/keys.txt
Public key: age1eqaqhyzdznd22j43j43e8qpra6z849hqljg9h3xruz0g0pmypassetfhdw
```

`.sops.yaml` [at the root of the repository](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/.sops.yaml "nixos-config/.sops.yaml at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config") was edited to add the new public key:

```yaml
keys:
  # Users
  - &users:
    - &lyuk98 270CB11B1189E79A17DCB7831BDAFDC5D60E735C
  # Hosts
  - &hosts:
    - &framework age14edzmqc4r07gp9lkj8z4gchccs373s8lcdrw69d6964tallpuuzqausgmk
    - &vault age1p0rc7s7r9krcqr8uy6dr8wlutyk9668a429y9k27xhfwtgwudgpq9e9ehq
    - &xps13 age1eqaqhyzdznd22j43j43e8qpra6z849hqljg9h3xruz0g0pmypassetfhdw
```

...and also to modify `creation_rules`:

```yaml
creation_rules:
  # other hosts

  # Secrets specific to host "xps13"
  - path_regex: hosts/xps13/secrets.ya?ml$
    key_groups:
    - age:
      - *xps13
      pgp:
      - *lyuk98

  # Secrets for user "lyuk98" to be used across hosts
  - path_regex: hosts/common/users/lyuk98/secrets.ya?ml$
    key_groups:
    - age:
      - *framework
      - *xps13
      pgp:
      - *lyuk98

  # NetworkManager connections
  - path_regex: hosts/common/optional/system-connections/.+\.nmconnection$
    key_groups:
    - age:
      - *framework
      - *xps13
      pgp:
      - *lyuk98
```

`secrets.yaml` was then encrypted in place using `sops`.

```
[lyuk98@framework:~/nixos-config]$ sops encrypt --in-place hosts/xps13/secrets.yaml
[lyuk98@framework:~/nixos-config]$ sops updatekeys --yes hosts/common/users/lyuk98/secrets.yaml
[lyuk98@framework:~/nixos-config]$ for connection in hosts/common/optional/system-connections/*.nmconnection; do sops updatekeys --yes $connection; done
```

## Creating a ZFS partition with [disko](https://github.com/nix-community/disko "nix-community/disko: Declarative disk partitioning and formatting using nix [maintainers=@Lassulus @Enzime @iFreilicht @Mic92 @phaer]")

As an added challenge, I decided to use [ZFS](https://openzfs.org "OpenZFS") for storing data. It is a file system that took me a while to get used to; eventually, though, I ended up with [a disko configuration](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/hosts/xps13/disko-config.nix "nixos-config/hosts/xps13/disko-config.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config") I could get along with.

A separate EFI system partition was still required, so I could not use the whole disk for ZFS. Two partitions were therefore [declared](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/hosts/xps13/disko-config.nix#L15-L43 "nixos-config/hosts/xps13/disko-config.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config") within `disko.devices.disk.main`.

```nix
{
  disko.devices = {
    disk = {

      # Primary disk
      main = {
        type = "disk";
        device = "/dev/nvme0n1";

        # GPT (partition table) as the disk's content
        content = {
          type = "gpt";

          # List of partitions
          partitions = {
            # EFI system partition
            esp = {
              priority = 100;
              end = "1G";
              type = "EF00";

              # FAT filesystem
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
                mountOptions = [
                  "defaults"
                  "umask=0077"
                ];
              };
            };

            # ZFS partition
            zfs = {
              size = "100%";

              content = {
                type = "zfs";
                pool = "zroot";
              };
            };
          };
        };
      };
    };

  };
}
```

ZFS has a concept of storage pools, which can consist of multiple physical devices. Together with partitions, one was also [defined](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/hosts/xps13/disko-config.nix#L48-L141 "nixos-config/hosts/xps13/disko-config.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config") within the same file. The new pool would employ transparent compression and native encryption using a passphrase.

```nix
{
  disko.devices = {
 
    zpool = {
      # The root zpool
      zroot = {
        type = "zpool";

        rootFsOptions = {
          # Do not mount root filesystem
          mountpoint = "none";

          # Enable transparent compression with zstd
          compression = "zstd";

          # Use ACL on datasets from this pool
          acltype = "posixacl";
          xattr = "sa";

          # Disable automatic snapshot
          "com.sun:auto-snapshot" = "false";
        };

        options = {
          # Force the pool to use 4,096 (2^12) byte sectors
          ashift = "12";
        };

        datasets = {
          # The root dataset
          "root" = {
            type = "zfs_fs";

            options = {
              # Use the default encryption method since OpenZFS 0.8.4
              encryption = "aes-256-gcm";

              # Use passphrase key format
              keyformat = "passphrase";
              keylocation = "prompt";

              # Do not mount this dataset
              mountpoint = "none";
            };
          };

          # Dataset for /nix
          "root/nix" = {
            type = "zfs_fs";

            options = {
              mountpoint = "/nix";
            };
            mountpoint = "/nix";
          };

          # Dataset for /persist
          "root/persist" = {
            type = "zfs_fs";

            options = {
              mountpoint = "/persist";
            };
            mountpoint = "/persist";
          };

          # Dataset for swap volume
          "root/swap" = {
            type = "zfs_volume";

            size = "16G";
            content = {
              type = "swap";
            };

            options = {
              # Set block size to system page size (4KiB)
              volblocksize = "4096";

              # Use zero-length encoding (ZLE) for compression
              compression = "zle";

              # Force data to be immediately flushed
              logbias = "throughput";
              sync = "always";

              # Prevent storing swap data in memory
              primarycache = "metadata";
              secondarycache = "none";

              # Disable automatic snapshot
              "com.sun:auto-snapshot" = "false";
            };
          };
        };
      };
    };

  };
}
```

Lastly, `tmpfs` [was used](https://github.com/lyuk98/nixos-config/blob/9c13b9c80d7a277a02e9c7dee35757ff48883e68/hosts/xps13/disko-config.nix#L143-L153 "nixos-config/hosts/xps13/disko-config.nix at 9c13b9c80d7a277a02e9c7dee35757ff48883e68 · lyuk98/nixos-config") for `/`:

```nix
{
  disko.devices = {

    nodev = {
      # Impermanent root with tmpfs
      "/" = {
        fsType = "tmpfs";
        mountOptions = [
          "defaults"
          "size=50%"
          "mode=0755"
        ];
      };
    };
  };
}
```

# Installing NixOS

The installation process became somewhat different from what I have initially anticipated, which supposedly involves [disko-install](https://github.com/nix-community/disko/blob/6f4cf5abbe318e4cd1e879506f6eeafd83f7b998/docs/disko-install.md "disko/docs/disko-install.md at 6f4cf5abbe318e4cd1e879506f6eeafd83f7b998 · nix-community/disko"). During failed attempts, it soon became clear that not having enough memory prevented me from building the system locally. To save as much memory space as possible, I downloaded a ["minimal installation ISO image"](https://nixos.org/download/#nixos-iso "Download | Nix & NixOS").

It was then written to a portable storage using Fedora Media Writer.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/c116f785-d6be-412c-aa52-2aaf9bb27e06.avif">
  <img src="https://images.lyuk98.com/08a1d01f-d0e7-4637-b092-8e4c6a5e8501.avif" alt="A dialog for choosing &quot;Write Options&quot; at Fedora Media Writer. A file named &quot;nixos-minimal-25.05.811874.daf6dc47aa4b-x86_64-linux.iso&quot; is selected, and &quot;USB SanDisk 3.2Gen1 (988.3 GB)&quot; is selected as a USB Drive." title="Writing the image">
</picture>

The soon-to-be-wiped laptop was then rebooted. Before booting into the installer, I disabled Secure Boot and put the firmware into Setup Mode by deleting all existing keys.

After booting, the device connected to the internet [using wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant#Connecting_with_wpa_cli "wpa_supplicant - ArchWiki").

```
[nixos@nixos:~]$ sudo systemctl start wpa_supplicant.service
[nixos@nixos:~]$ wpa_cli
```

The root user's password was then set, so that SSH connections to the device can be made.

```
[nixos@nixos:~]$ sudo passwd root
```

Meanwhile, on my (more powerful) machine, the age key that was saved to a temporary directory was copied to a new temporary directory; it was to prepare for the next step.

```
[lyuk98@framework:~]$ dir=$(mktemp --directory)
[lyuk98@framework:~]$ install -D --mode 0600 $tmp/keys.txt $dir/persist/var/lib/sops-nix/keys.txt
```

With [nixos-anywhere](https://github.com/nix-community/nixos-anywhere/ "nix-community/nixos-anywhere: Install NixOS everywhere via SSH [maintainers=@Mic92 @Lassulus @phaer @Enzime @a-kenji]"), an actual installation was performed.

```
[lyuk98@framework:~/nixos-config]$ nix run github:nix-community/nixos-anywhere -- \
  --flake .#xps13 \
  --target-host root@<ip-address> \
  --extra-files $dir \
  --phases disko,install
```

Among phases `kexec`, `disko`, `install`, and `reboot`, I only specified phases `disko` and `install`. There was no need to `kexec` into an installer since the device has already booted into one, and I omitted `reboot` to perform some steps after installation. I did not actually do anything afterwards, though, and the only thing the absence of the `reboot` phase made me do was to reboot the machine in my own terms.

## Enabling Secure Boot with Lanzaboote

> Despite its criticisms, I wanted to continue using the system with Secure Boot enabled. I used [Lanzaboote](https://github.com/nix-community/lanzaboote "nix-community/lanzaboote: Secure Boot for NixOS [maintainers=@blitz @raitobezarius @nikstur]"), as most guides for NixOS mentioned it.

Just like [my previous self](https://lyuk98.com/62055/switching-from-arch-linux-to-nixos "Switching from Arch Linux to NixOS"), [Lanzaboote](https://github.com/nix-community/lanzaboote "nix-community/lanzaboote: Secure Boot for NixOS [maintainers=@blitz @raitobezarius @nikstur]") was chosen to enable Secure Boot on the new device.

Following [the quickstart guide](https://github.com/nix-community/lanzaboote/blob/cbe4178bf694a9081d51e8ac55e5baadc1d4fa40/docs/QUICK_START.md "lanzaboote/docs/QUICK_START.md at cbe4178bf694a9081d51e8ac55e5baadc1d4fa40 · nix-community/lanzaboote"), the first thing to do was to create new Secure Boot keys.

```
[lyuk98@xps13:~]$ nix shell nixpkgs#sbctl
[lyuk98@xps13:~]$ sudo sbctl create-keys
```

I then remotely applied [the change](https://github.com/lyuk98/nixos-config/commit/1b962bbac6ae7abafdfb08a3000a1dc5b4038137 "Enable Lanzaboote for XPS 13 · lyuk98/nixos-config@1b962bb") that disables systemd-boot and enables Lanzaboote.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild switch \
  --target-host lyuk98@xps13 \
  --sudo \
  --ask-sudo-password \
  --flake .#xps13
```

After it was done, I realised that the Secure Boot keys were wiped because it was not set to be preserved across reboots. Because the aforementioned change now does so, all I had to do was to create another set of keys:

```
[lyuk98@xps13:~]$ sudo sbctl create-keys
```

...and to apply the same configuration, which would sign EFI binaries with the new keys:

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild switch \
  --target-host lyuk98@xps13 \
  --sudo \
  --ask-sudo-password \
  --flake .#xps13
```

I then made sure that enabling Secure Boot would not prevent me from booting into NixOS.

```
[lyuk98@xps13:~]$ sudo sbctl verify
Verifying file database and EFI images in /boot...
✓ /boot/EFI/BOOT/BOOTX64.EFI is signed
✓ /boot/EFI/Linux/nixos-generation-1-45wzdjf3iqke66uzyktrhh45qjb5zanyml2hxfk7bj5cnuxvzorq.efi is signed
✓ /boot/EFI/Linux/nixos-generation-2-tgt7vtvzoif4nstwjunxjnyqc7q7rqi3ctwimgntsihp5mtjzu3q.efi is signed
✓ /boot/EFI/Linux/nixos-generation-3-64edmqoob3gj6omtovcypgiew5r6ncvnyymlynoymcph5bhvnowa.efi is signed
✗ /boot/EFI/nixos/kernel-6.12.55-zxhzas57mq7ufqrlt4l4moejma4fin7avasznetmjbh3wtcwffoq.efi is not signed
✓ /boot/EFI/systemd/systemd-bootx64.efi is signed
```

With everything ready, the keys were enrolled to the firmware.

```
[lyuk98@xps13:~]$ sudo sbctl enroll-keys --firmware-builtin --microsoft
```

It is supposed to enable Secure Boot on its own, but I had to manually do so.

## Automatic decryption with Clevis

With the presence of Trusted Platform Module (TPM), I can unlock the storage on boot without entering its passphrase. I have always used systemd-cryptenroll for this purpose, but it could apparently only be applied to LUKS-encrypted volumes.

[Clevis](https://github.com/latchset/clevis "latchset/clevis: Automated Encryption Framework") was chosen as an alternative. Because the passphrase could be piped to the unlocking process:

```
[root@xps13:~]# systemd-ask-password | zfs load-key -n zroot/root
🔐 Password: (no echo)               
1 / 1 key(s) successfully verified
```

...I could let Clevis encrypt one and decrypt it in initrd to unlock the storage. In NixOS, this could easily be achieved by [adding a few lines of code](https://github.com/lyuk98/nixos-config/blob/6e3ebf70a77cc545b7a470346dd1df46d19ba14d/hosts/xps13/clevis.nix "nixos-config/hosts/xps13/clevis.nix at 6e3ebf70a77cc545b7a470346dd1df46d19ba14d · lyuk98/nixos-config"):

```nix
{
  boot.initrd.clevis = {
    # Enable Clevis in initrd
    enable = true;

    # Unlock the device at boot using secret
    devices."zroot/root" = {
      secretFile = "${./root.jwe}";
    };
  };
}
```

`root.jwe` was created from the new device, which is just the decryption passphrase encrypted with TPM. The PCR value of `7` makes the decryption (of the passphrase) dependent of the Secure Boot settings.

```
[lyuk98@xps13:~]$ nix shell nixpkgs#clevis
[lyuk98@xps13:~]$ systemd-ask-password | clevis encrypt tpm2 '{"pcr_ids": "7"}' > root.jwe
```

The encrypted key was then transferred to my repository.

```
[lyuk98@framework:~/nixos-config]$ scp lyuk98@xps13:~/root.jwe hosts/xps13/root.jwe
```

The new configuration was then applied to the target machine.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild switch \
  --target-host lyuk98@xps13 \
  --sudo \
  --ask-sudo-password \
  --flake .#xps13
```

# Disconnecting the battery

The laptop was now going to be plugged into a charger at all times, and I did not want it to be done with the battery still attached. As a result, I partially followed [a guide from iFixit](https://www.ifixit.com/Guide/Dell+XPS+13+Battery+Replacement/76060 "Dell XPS 13 Battery Replacement - iFixit Repair Guide") to disconnect the battery. As an owner of a Framework Laptop 13, I was glad to find out that I do not need any tool other than [a Framework Screwdriver](https://frame.work/products/framework-screwdriver?v=FRANFWET01 "Framework | Framework Screwdriver").

First, I removed eight screws from the bottom of the laptop.

![The laptop flipped upside down](https://images.lyuk98.com/1a0e87a5-f35c-4bde-8f14-abf0073d7ad0.avif)

The flap in the middle was then opened, and a screw hidden underneath was removed.

![A flap in the middle of the laptop's bottom cover is partially opened, with a screwdriver keeping it from closing. The screwdriver is placed on top of a screw underneath the flap.](https://images.lyuk98.com/5d43cbe9-0b19-4634-91c0-3950c54625bf.avif)

Opening the cover took some force. After it was done, I could see the connector:

![A battery connector inside the laptop, which is still connected](https://images.lyuk98.com/c66f6c39-812f-4436-b784-2d3d36cc5079.avif)

...which I disconnected using the spudger end of my screwdriver:

![A disconnected battery connector inside the laptop](https://images.lyuk98.com/71a2e842-431a-494b-b6d4-7201bd86421a.avif)

I did not remove the battery itself. The cover was reattached right away.

# The next step

With the server ready, the next step would be to run Kubernetes. I have not figured out how to run OpenStack on top of it yet, but I probably will as I work on it.
