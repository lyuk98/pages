---
tags:
  - Amazon Web Services
  - Backblaze B2
  - Cloudflare
  - DigitalOcean
  - Ente
  - GitHub Actions
  - HashiCorp Vault
  - NixOS
  - SOPS
  - Tailscale
  - Terraform
tocopen: true
date: '2025-09-28'
title: Hosting Ente with Terraform, Vault, and NixOS
description: Ente, the private photo storage service, was self-hosted in a hard way using declarative methods of resource management.
---

![Logo of Ente](https://images.lyuk98.com/0998d5a9-bafe-4aa2-be99-d49144204337.avif)

[Ente](https://ente.io/ "Ente Photos: Store and share your photos with absolute privacy") is a platform to host photos that uses end-to-end encryption to store its data.

I have more than a decade of photos saved with [Google Photos](https://www.google.com/photos/about/ "Google Photos: Edit, organise, search and back up your photos"), but I wanted to keep them in a place that is designed to be private. The problem, however, was that the collection grew to a few terabytes over the years, and putting them on a privacy-respecting image platform was either very expensive or outright impossible. To work around the problem, I decided to self-host Ente.

<h1 id="the-plan">The plan</h1>

<h2 id="the-plan-infrastructure">The infrastructure</h2>

Unlike the [PeerTube instance I self-hosted](https://lyuk98.com/60610/self-hosting-peertube-with-tailscale "Self-hosting PeerTube with Tailscale"), which is only reachable via [Tailscale](https://tailscale.com/ "Tailscale · Best VPN Service for Secure Networks"), I wanted my image platform to be accessible via the internet (because Android [cannot connect to Tailscale and my VPN service simultaneously](https://tailscale.com/kb/1105/other-vpns "Can I use Tailscale alongside other VPNs? · Tailscale Docs")). The easiest way for me to expose a service to the internet was by using a cloud provider, which I eventually ended up doing.

To run the service, the following cloud providers were used:

- Amazon Web Services: for a virtual private server (Amazon Lightsail) for hosting Ente's Museum (API server), web server, and a PostgreSQL database instance
- Backblaze B2: for managed object storage

I initially considered OVHcloud for a VPS, but the order for one was cancelled without a satisfying explanation. It is apparently [not an isolated incident](https://www.reddit.com/r/ovh/comments/you396/order_cancelled_no_explanation/ "Order cancelled, no explanation! : r/ovh"); rather than trying to sort it out, I switched to Amazon Web Services instead. I initially hesitated to do so, probably due to my baseless reluctance to use something from a US tech giant, but I thought I will learn more about it and make an educated decision in the future.

Another thing I have considered was using a dedicated database instance, which I felt could be a safer place to store data. My naive self thought I will not be using the service often enough, used Scaleway's [Serverless SQL Database](https://www.scaleway.com/serverless-sql-database/ "Serverless SQL Database | Scaleway"), and finished deploying with it, only to realise its true cost a week later. I changed my plan and decided to run database locally within the Lightsail instance, while separating its persistent data using [a Lightsail disk](https://docs.aws.amazon.com/lightsail/latest/userguide/elastic-block-storage-and-ssd-disks-in-amazon-lightsail.html "Expand storage and performance with Lightsail block storage disks - Amazon Lightsail").

<h2 id="the-plan-code">Using code for almost everything</h2>

Ente provides a simple script from [their guide](https://help.ente.io/self-hosting/ "Quickstart - Self-hosting | Ente Help") to quickly start up a self-hosted server. However, even though I could just run the script and add a few tweaks to get the server running, I did not want to imperatively do so and repeat the process in case the system malfunctions.

Fortunately for me, [Nixpkgs](https://github.com/NixOS/nixpkgs "NixOS/nixpkgs: Nix Packages collection & NixOS") (which implements [NixOS](https://nixos.org/ "Nix & NixOS | Declarative builds and deployments")) contains [modules](https://github.com/NixOS/nixpkgs/pull/406847 "nixos/ente: init modules by oddlama · Pull Request #406847 · NixOS/nixpkgs") that can help run the service in a simple manner. With them, I could make the system declarative (down to the disk layout) and practically stateless.

This was also an opportunity for me to expand my knowledge. NixOS allowed me to manage an operating system with code, and with [Terraform](https://www.hashicorp.com/products/terraform "HashiCorp Terraform | Infrastructure as code provisioning"), I could extend this ability to infrastructure management.

<h1 id="vault">Setting up secret management with Vault</h1>

I typically use [Sops](https://getsops.io/ "SOPS: Secrets OPerationS") to encrypt secrets, which then end up in a Git repository. However, with Terraform involved, I knew it will be a pain to update secrets that way. A way to dynamically retrieve them was needed, and I chose [Vault](https://www.hashicorp.com/products/vault "HashiCorp Vault | Identity-based secrets management"), another product from HashiCorp, to do so.

<h2 id="vault-host">Hosting Vault</h2>

I initially wanted to lessen the maintenance burden by using a managed Vault instance. However, upon checking the price for a Vault Dedicated cluster, I realised that it was an overkill for my use case. As a result, I chose to host one myself.

After facing difficulties with installing NixOS on low-end Lightsail instances, I switched to [DigitalOcean Droplets](https://www.digitalocean.com/products/droplets "DigitalOcean Droplets | Scalable Cloud Compute Starting at $4/mo"). To create one with NixOS, I had to upload an image with my configuration applied.

<h2 id="vault-nixos">Preparing NixOS configurations</h2>

I [have already written](https://lyuk98.com/62055/switching-from-arch-linux-to-nixos "Switching from Arch Linux to NixOS") NixOS configurations before, so I did not have to write everything from scratch. However, my goal was to make the new system more declarative than before, so [impermanence](https://github.com/nix-community/impermanence "nix-community/impermanence: Modules to help you handle persistent state on systems with ephemeral root storage [maintainer=@talyz]") (for controlling which data stays across reboots) and [disko](https://github.com/nix-community/disko/ "nix-community/disko: Declarative disk partitioning and formatting using nix [maintainers=Lassulus Enzime iFreilicht Mic92 phaer]") (for declarative partitioning) were set up.

Implementing impermanence was not difficult, though I had to consider my existing non-impermanent system that was not ready for impermanence just yet. I [added an input](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/flake.nix#L46 "nixos-config/flake.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") at `flake.nix` and [created a module](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/common/optional/persistence.nix "nixos-config/hosts/common/optional/persistence.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") `persistence.nix` for host-agnostic configurations, containing a few directories that should not be wiped every boot.

```nix
{
  # Flake description...

  inputs = {
    # Preceding flake inputs...

    # Used for declaration of impermanent systems
    impermanence.url = "github:nix-community/impermanence";

    # Succeeding flake inputs...
  };

  # Flake outputs follow...
}
```

```nix
{ inputs, ... }:
{
  imports = [ inputs.impermanence.nixosModules.impermanence ];

  environment.persistence = {
    # Enable persistence storage at /persist
    "/persist" = {
      # Prevent bind mounts from being shown as mounted storage
      hideMounts = true;

      # Create bind mounts for given directories
      directories = [
        "/var/lib/nixos"
        "/var/lib/systemd"
        "/var/log"
      ];
    };
  };

  # Require /persist for booting
  fileSystems."/persist".neededForBoot = true;
}
```

Using disko was next. [The configuration](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/vault/disko-config.nix "nixos-config/hosts/vault/disko-config.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") at Vault-specific `disko-config.nix` contains declaration of an EFI system partition, a `tmpfs` mounted at `/`, and a Btrfs partition, with a swap file and subvolumes to be mounted at `/nix` and `/persist`.

```nix
{
  disko.devices = {
    disk = {

      # Primary disk
      main = {
        type = "disk";
        device = "/dev/vda";
        imageSize = "4G";

        # GPT (partition table) as the disk's content
        content = {
          type = "gpt";

          # List of partitions
          partitions = {
            # BIOS boot partition
            boot = {
              priority = 100;
              size = "1M";
              type = "EF02";
            };
            # EFI system partition
            esp = {
              priority = 100;
              end = "500M";
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

            # Root partition
            root = {
              size = "100%";

              # Btrfs filesystem
              content = {
                type = "btrfs";

                # Override existing partition
                extraArgs = [
                  "-f"
                ];

                # Subvolumes
                subvolumes = {
                  # /nix
                  "/nix" = {
                    mountOptions = [
                      "defaults"
                      "compress=zstd"
                      "x-systemd.growfs"
                    ];
                    mountpoint = "/nix";
                  };

                  # Persistent data
                  "/persist" = {
                    mountOptions = [
                      "defaults"
                      "compress=zstd"
                    ];
                    mountpoint = "/persist";
                  };

                  # Swap file
                  "/var/swap" = {
                    mountpoint = "/var/swap";
                    swap = {
                      swapfile.size = "1G";
                    };
                  };
                };
              };
            };
          };
        };
      };
    };

    nodev = {
      # Impermanent root with tmpfs
      "/" = {
        fsType = "tmpfs";
        mountOptions = [
          "defaults"
          "size=25%"
          "mode=0755"
        ];
      };
    };
  };
}
```

For `/`, it was important to note that `tmpfs` mounts with a default permission of `1777`. It worked fine in practice, but the SSH daemon refused to operate in certain cases. The solution, as this [online discussion](https://discourse.nixos.org/t/ssh-sshd-3771-authentication-refused-bad-ownership-or-modes-for-directory/25662 "SSH: sshd[3771]: Authentication refused: bad ownership or modes for directory / - Help - NixOS Discourse") pointed out, was to add a mount option `mode=0755`, which is the permission `/` is expected to have.

<h3 id="vault-nixos-sops">Encrypting secrets with Sops</h3>

For Vault itself, just like [when I installed NixOS](https://lyuk98.com/62055/switching-from-arch-linux-to-nixos "Switching from Arch Linux to NixOS") for the first time, [sops-nix](https://github.com/Mic92/sops-nix "Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops") was used to encrypt my secrets.

A new [age](https://github.com/FiloSottile/age "FiloSottile/age: A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability.") key for the new instance was first generated; it was to be copied over later on.

```
[lyuk98@framework:~]$ nix shell nixpkgs#age
[lyuk98@framework:~]$ age-keygen --output keys.txt
Public key: age1p0rc7s7r9krcqr8uy6dr8wlutyk9668a429y9k27xhfwtgwudgpq9e9ehq
```

I [edited](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/.sops.yaml "nixos-config/.sops.yaml at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") existing `.sops.yaml` to add the new public key and specify where the secrets will be.

```yaml
keys:
  # Hosts
  - &hosts:
    - &vault age1p0rc7s7r9krcqr8uy6dr8wlutyk9668a429y9k27xhfwtgwudgpq9e9ehq

creation_rules:
  # Secrets specific to host "vault"
  - path_regex: hosts/vault/secrets.ya?ml$
    key_groups:
    - age:
      - *vault
```

After adding a few secrets to a file, I encrypted it using `sops`.

```
[lyuk98@framework:~/nixos-config]$ sops encrypt --in-place hosts/vault/secrets.yaml
```

<h3 id="vault-nixos-tailscale">Setting up Tailscale</h3>

I did not feel that the instance needs to be exposed to the internet, so I used [Tailscale](https://tailscale.com/ "Tailscale · Best VPN Service for Secure Networks") to limit access. I could run `tailscale up` and connect to tailnet manually, but I wanted to automate this process.

For NixOS, automatic startups can be achieved by providing [an option](https://search.nixos.org/options?channel=unstable&show=services.tailscale.authKeyFile "NixOS Search - Options - services.tailscale.authKeyFile") `services.tailscale.authKeyFile`, and I needed an [OAuth client](https://tailscale.com/kb/1215/oauth-clients "OAuth clients · Tailscale Docs") with `auth_keys` [scope](https://tailscale.com/kb/1215/oauth-clients#scopes "OAuth clients · Tailscale Docs") for that. Moreover, for clients to be given such a scope, they also needed to be assigned one or more [tags](https://tailscale.com/kb/1068/tags "Group devices with tags · Tailscale Docs"). From the visual access controls editor, I [added tags](https://login.tailscale.com/admin/acls/visual/tags/add "Create tag - Tailscale") named `museum` and `vault`. Some existing tags that I have previously created [while working on PeerTube](https://lyuk98.com/60610/self-hosting-peertube-with-tailscale "Self-hosting PeerTube with Tailscale") (`ci` and `webserver`) were also to be used later on.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/bf3b98a8-e994-4cb8-9438-efd22b5d89cf.avif">
  <img src="https://images.lyuk98.com/a2dbc47e-cced-4d62-af96-39f7c154177d.avif" alt="A visual interface for managing access controls within Tailscale that shows tags named tag:caddy, tag:ci, tag:museum, tag:peertube, tag:vault, and tag:webserver" title="List of tags currently present">
</picture>

With the tags in place, [an access rule was added](https://login.tailscale.com/admin/acls/visual/general-access-rules/add "Add rule - Tailscale") to allow members of the tailnet to access Vault

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/be8f1333-cf7a-4061-b83c-11aa0e8d403f.avif">
  <img src="https://images.lyuk98.com/3e11dc9a-bc79-497d-ab69-a341d7860d2a.avif" alt="An interface for adding an access rule. Source is set as autogroup:member, destination as tag:vault, and port and protocol as tcp:8200." title="Creating a new access rule">
</picture>

[Creation of an OAuth client](https://login.tailscale.com/admin/settings/oauth "OAuth clients - Tailscale") followed. The tags `vault` and `webserver` were assigned for the `auth_keys` scope.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/a578421d-2d6e-4d7e-9566-276157e8824a.avif">
  <img src="https://images.lyuk98.com/73eb4290-9385-4bd4-bb92-0f8308dcc641.avif" alt="An interface for adding an OAuth client at Tailscale. Write access to the scope Auth Keys is checked, and for tags, tag:vault and tag:webserver are added." title="Creating a new OAuth client">
</picture>

The created secret was added to `secrets.yaml`. The key value was set to `tailscale-auth-key`, so that I can refer to it as `sops.secrets.tailscale-auth-key` upon writing NixOS configurations.

```
[lyuk98@framework:~/nixos-config]$ sops edit hosts/vault/secrets.yaml
```

```yaml
# Existing secret...
tailscale-auth-key: tskey-client-<ID>-<secret>
```

Vault-specific [Tailscale configuration](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/vault/tailscale.nix "nixos-config/hosts/vault/tailscale.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") was written afterwards.

```nix
{ config, ... }:
{
  # Get auth key via sops-nix
  sops.secrets.tailscale-auth-key = {
    sopsFile = ./secrets.yaml;
  };

  # Apply host-specific Tailscale configurations
  services.tailscale = {
    # Provide auth key to issue `tailscale up` with
    authKeyFile = config.sops.secrets.tailscale-auth-key.path;

    # Enable Tailscale SSH and advertise tags
    extraUpFlags = [
      "--advertise-tags=tag:webserver,tag:vault"
      "--hostname=vault"
      "--ssh"
    ];

    # Use routing features for servers
    useRoutingFeatures = "server";
  };
}
```

<h2 id="vault-custom-image">Creating a custom image</h2>

There were some ways to turn my NixOS configuration into [an image that DigitalOcean accepts](https://docs.digitalocean.com/products/custom-images/how-to/upload/#image-requirements "How to Upload Custom Images | DigitalOcean Documentation"):

- Using [a DigitalOcean NixOS module](https://github.com/NixOS/nixpkgs/blob/77fce1dc584c4ea42f182bb3f45b3f40f283e18f/nixos/modules/virtualisation/digital-ocean-image.nix "nixpkgs/nixos/modules/virtualisation/digital-ocean-image.nix at 77fce1dc584c4ea42f182bb3f45b3f40f283e18f · NixOS/nixpkgs") and building with `nix build .#nixosConfigurations.vault.config.system.build.digitalOceanImage`
- Including [nixos-generators](https://github.com/nix-community/nixos-generators "nix-community/nixos-generators: Collection of image builders [maintainer=@Lassulus]") and running `nix build .#nixosConfigurations.vault.config.formats.raw-efi`

However, I faced a major problem: they attempted to create an image with their own partition layout and `/` was assumed to be an ext4 file system. The assumption conflicted with my existing configuration, that used Btrfs and root-on-tmpfs setup, and was unsuitable for my use case.

Fortunately, disko provided [a way](https://github.com/nix-community/disko/blob/aba0ae38df216a3e0983388c7702ef7fcf93d5e4/docs/disko-images.md "disko/docs/disko-images.md at aba0ae38df216a3e0983388c7702ef7fcf93d5e4 · nix-community/disko") to generate a `.raw` image with my own partition layout.

First, the script to generate the image was built.

```
[lyuk98@framework:~/nixos-config]$ nix build .#nixosConfigurations.vault.config.system.build.diskoImagesScript
```

I ran the resultant script afterwards. The age key generated for the instance was also copied over with the flag `--post-format-files`. The process involved booting into a virtual machine and installing NixOS to the image.

```
[lyuk98@framework:~/nixos-config]$ ./result --post-format-files ~/keys.txt persist/var/lib/sops-nix/keys.txt
```

The script ran successfully after a minute or so. I then compressed the image to reduce the size of what will be uploaded to DigitalOcean.

```
[lyuk98@framework:~/nixos-config]$ bzip2 -9 main.raw
```

<h2 id="vault-create-droplet">Creating a Droplet with the image</h2>

I first uploaded the custom image at the [Control Panel](https://cloud.digitalocean.com/images/custom_images "Control Panel - DigitalOcean").

![A dialog for uploading an image. Image name is set to vault, and the distribution is set to Unknown.](https://images.lyuk98.com/7386d660-cc7d-4862-8a35-9714b561662c.avif "Uploading an image")

When it was done, I [created a droplet](https://cloud.digitalocean.com/droplets/new "Create Droplets - DigitalOcean") with the image.

![An interface for creation of a droplet, where the image to create one with was set to vault with Unknown OS](https://images.lyuk98.com/42c8c1e7-690a-4b9c-a848-e3e3612055b4.avif "Creating a droplet with the image")

The instance was running, but it did not utilise more than [how much it was allocated](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/vault/disko-config.nix#L9 "nixos-config/hosts/vault/disko-config.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") during image generation. There were some ways to grow a root partition, but given my unusual setup (where `/` is tmpfs), I did not feel they will work. In the end, I used `growpart` to imperatively resize the partition.

```
[root@vault:~]# nix shell nixpkgs#cloud-utils
[root@vault:~]# growpart /dev/vda 3
[root@vault:~]# btrfs filesystem resize max /nix
```

With the system ready, I moved onto the Vault service.

<h2 id="vault-run">Running Vault</h2>

The first thing I did was to decide how to store data. With impermanence set up on NixOS, I either had to persist the storage path or use a different backend. I have already been using [Backblaze B2](https://www.backblaze.com/cloud-storage "Low Cost, High Performance S3 Compatible Object Storage"), so I opted to use the [S3 storage backend](https://developer.hashicorp.com/vault/docs/configuration/storage/s3 "S3 configuration | Vault | HashiCorp Developer"). The drawback was the lack of [high availability](https://developer.hashicorp.com/vault/docs/concepts/ha "High Availability | Vault | HashiCorp Developer"), but with just a single service using Vault, I did not think it mattered that much.

I went my Backblaze account and created a bucket (which name is not actually `vault-bucket`).

![A dialog for creating a bucket. The name is set to vault-bucket, files are set to be private, default encryption is enabled, and object lock is disabled.](https://images.lyuk98.com/8b3094bd-2cf7-49cb-a3a4-f0d7d8bfdb8c.avif "Creating a bucket")

An application key to access the bucket was also created afterwards.

![A dialog for creating an application key. The name of key is vault, allowing access only to vault-bucket, while granting both read and write access.](https://images.lyuk98.com/260dee56-af52-4882-a675-67a694264daa.avif "Creating an application key")

At where my NixOS configuration for Vault is, I created `vault.hcl` containing sensitive information. It was to be encrypted and passed on to [the option](https://search.nixos.org/options?channel=unstable&show=services.vault.extraSettingsPaths "NixOS Search - Options - services.vault.extraSettingsPaths") `services.vault.extraSettingsPaths`.

```hcl
api_addr = "http://<API address>:8200"

storage "s3" {
  bucket              = "vault-bucket"
  endpoint            = "https://s3.<region>.backblazeb2.com"
  region              = "<region>"
  access_key          = "<application key ID>"
  secret_key          = "<application key>"
  s3_force_path_style = "true"
}
```

To allow `vault.hcl` to be encrypted, I added a creation rule at `.sops.yaml`.

```yaml
creation_rules:
  # Secrets specific to host "vault"
  - path_regex: hosts/vault/secrets.ya?ml$
    key_groups:
    - age:
      - *vault
  - path_regex: hosts/vault/vault.hcl
    key_groups:
    - age:
      - *vault
```

The settings file could then be encrypted in place.

```
[lyuk98@framework:~/nixos-config]$ sops encrypt --in-place hosts/vault/vault.hcl
```

When the secret was ready, I [wrote](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/vault/vault.nix "nixos-config/hosts/vault/vault.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") `vault.nix` containing Vault-related configuration.

```nix
{
  pkgs,
  lib,
  config,
  ...
}:
{
  sops.secrets = {
    # Get secret Vault settings
    vault-settings = {
      format = "binary";

      # Change ownership of the secret to user `vault`
      owner = config.users.users.vault.name;
      group = config.users.groups.vault.name;

      sopsFile = ./vault.hcl;
    };
  };

  # Allow unfree package for Vault
  nixpkgs.config.allowUnfreePredicate =
    pkg:
    builtins.elem (lib.getName pkg) [
      "vault"
      "vault-bin"
    ];

  services.vault = {
    # Enable Vault daemon
    enable = true;

    # Use binary version of Vault from Nixpkgs
    package = pkgs.vault-bin;

    # Listen to all available interfaces
    address = "[::]:8200";

    # Use S3 as a storage backend
    storageBackend = "s3";

    # Add secret Vault settings
    extraSettingsPaths = [
      config.sops.secrets.vault-settings.path
    ];

    # Enable Vault UI
    extraConfig = ''
      ui = true
    '';
  };
}
```

The new configuration was applied. I could not successfully issue `nixos-rebuild switch` as it apparently conflicted with what Cloud-init performed after boot, so I rebooted the instance after running `nixos-rebuild boot`.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild boot --target-host root@vault --flake .#vault
[lyuk98@framework:~/nixos-config]$ ssh root@vault reboot
```

After making sure `vault.service` was running, I initialised Vault by running `vault operator init`. Five unsealing keys were generated as a result.

```
[root@vault:~]# NIXPKGS_ALLOW_UNFREE=1 nix shell --impure nixpkgs#vault-bin
[root@vault:~]# vault operator init -address http://127.0.0.1:8200
```

I checked the status of the Vault from my device afterwards to verify that I can access the Vault via tailnet.

```
[lyuk98@framework:~]$ vault status -address "http://<API address>:8200"
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    0/3
Unseal Nonce       n/a
Version            1.19.4
Build Date         2025-05-14T13:04:47Z
Storage Type       s3
HA Enabled         false
```

<h2 id="vault-postinstall">Post-installation configuration</h2>

First, I set the `VAULT_ADDR` environment variable so that I no longer have to supply `-address` parameter.

```
[lyuk98@framework:~]$ export VAULT_ADDR="http://<API address>:8200"
```

Before doing anything with the Vault, it had to be unsealed. Doing so required entering at least three of the five unsealing keys. I ran `vault operator unseal` three times, entering a different key each time.

```
[lyuk98@framework:~]$ vault operator unseal
[lyuk98@framework:~]$ vault operator unseal
[lyuk98@framework:~]$ vault operator unseal
```

With the root token initially provided during `vault operator init`, I logged into the Vault by running `vault login`.

```
[lyuk98@framework:~]$ vault login
```

<h1 id="bootstrap-imperative">Bootstrapping (the imperative part)</h1>

The Vault was ready, and I started preparing for granting Terraform access to various cloud providers.

However, I started wondering: how should the access credentials themselves (that are used for Terraform) be generated? I felt most of them have to be created imperatively, but I wanted to create a separate Terraform configuration for what I could declaratively manage.

<h2 id="bootstrap-imperative-vault">Enabling Vault's capabilities</h2>

I anticipated several key/value secrets to be stored on Vault. To prepare for them, KV secrets engine was enabled.

```
[lyuk98@framework:~]$ vault secrets enable kv
Success! Enabled the kv secrets engine at: kv/
```

For Ente's Museum to use Vault, I thought using [AppRole](https://developer.hashicorp.com/vault/docs/auth/approle "Use AppRole authentication | Vault | HashiCorp Developer") authentication made sense. It was therefore also enabled.

```
[lyuk98@framework:~]$ vault auth enable approle
Success! Enabled approle auth method at: approle/
```

<h2 id="bootstrap-imperative-b2">Enabling access to Backblaze B2</h2>

I had to consider how to store the Terraform state, and since I already use Backblaze B2, I decided to use the `s3` backend. The goal was to create a bucket (to store state files) and two application keys: one for accessing state and another for creating application keys.

I started by providing my master application key to the Backblaze B2 command-line tool.

```
[lyuk98@framework:~]$ nix shell nixpkgs#backblaze-b2
[lyuk98@framework:~]$ backblaze-b2 account authorize
```

A bucket (which name is not actually `terraform-state-ente`) to store state files was created next.

```
[lyuk98@framework:~]$ backblaze-b2 bucket create \
  terraform-state-ente \
  allPrivate \
  --default-server-side-encryption SSE-B2
```

As a way to access state files, an application key, which access is restricted to the specified bucket, was created.

```
[lyuk98@framework:~]$ backblaze-b2 key create \
  --bucket terraform-state-ente \
  --name-prefix terraform-ente-bootstrap \
  terraform-state-ente-bootstrap \
  deleteFiles,listBuckets,listFiles,readFiles,writeFiles
```

A separate application key, for Terraform to create an application key, was also created.

```
[lyuk98@framework:~]$ backblaze-b2 key create \
  terraform-ente-bootstrap \
  deleteKeys,listBuckets,listKeys,writeKeys
```

The [capabilities](https://www.backblaze.com/docs/cloud-storage-application-key-capabilities "Cloud Storage Application Key Capabilities") for the two application keys are results of trial and error; I was not able to find proper documentation describing what are necessary.

I wrote a file `bootstrap.json`, containing two application keys, to prepare the data that will be sent over.

```json
{
  "b2_application_key": "<application key>",
  "b2_application_key_id": "<application key ID>",
  "b2_state_application_key": "<application key for writing Terraform state>",
  "b2_state_application_key_id": "<application key ID for writing Terraform state>"
}
```

The data was then saved into Vault.

```
[lyuk98@framework:~]$ edit bootstrap.json
[lyuk98@framework:~]$ vault kv put -mount=kv ente/bootstrap @bootstrap.json
Success! Data written to: kv/ente/bootstrap
```

I did not want some details to be visible (that some may find acceptable). As a result, I wrote them into a file, which was also sent to Vault.

```json
{
  "b2_bucket": "terraform-state-ente",
  "b2_endpoint": "https://s3.<region>.backblazeb2.com"
}
```

```
[lyuk98@framework:~]$ edit backend.json
[lyuk98@framework:~]$ vault kv put -mount=kv ente/b2/tfstate-bootstrap @backend.json
Success! Data written to: kv/ente/b2/tfstate-b2-ente
```

<h2 id="bootstrap-imperative-github">Configuring OpenID Connect for GitHub Actions</h2>

GitHub Actions was my choice of tool for deploying Terraform configurations. To access Amazon Web Services, I could, just like what many people I found online did, use access keys. However, I was recommended every step of the way to look for alternative methods instead of [creating long-term credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html "Manage access keys for IAM users - AWS Identity and Access Management").

> As a [best practice](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html), use temporary security credentials (such as IAM roles) instead of creating long-term credentials like access keys. Before creating access keys, review the [alternatives to long-term access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/security-creds-programmatic-access.html#security-creds-alternatives-to-long-term-access-keys).

To eliminate the need of static credentials, I opted to use OpenID Connect (OIDC). With [the guide](https://docs.github.com/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws "Configuring OpenID Connect in Amazon Web Services - GitHub Docs") provided by GitHub, I took steps to grant GitHub Actions workflows access to AWS.

I used [AWS CLI](https://aws.amazon.com/cli/ "AWS CLI") to perform tasks, and to do so, I ran `aws sso login`. Prior to logging in, setting up [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html "What is IAM Identity Center? - AWS IAM Identity Center") was needed.

```
[lyuk98@framework:~]$ nix shell nixpkgs#awscli2
[lyuk98@framework:~]$ aws sso login
```

After logging myself in, GitHub was added as an OIDC provider.

```
[lyuk98@framework:~]$ aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

A role that GitHub Actions workflows will assume was to be created. In order to prevent anything else from assuming the role, a trust policy document based on [the guide](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws#configuring-the-role-and-trust-policy "Configuring OpenID Connect in Amazon Web Services - GitHub Docs") was created, allowing access to workflows running for my repository.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::<account ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:lyuk98/terraform-ente-bootstrap:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The trust policy was written to file and was referenced upon the creation of the role.

```
[lyuk98@framework:~]$ edit terraform-ente-bootstrap-trust-policy.json
[lyuk98@framework:~]$ aws iam create-role \
  --role-name terraform-ente-bootstrap \
  --assume-role-policy-document file://terraform-ente-bootstrap-trust-policy.json
```

I initially wanted to let Vault's [AWS secrets engine](https://developer.hashicorp.com/vault/docs/secrets/aws "AWS secrets engine | Vault | HashiCorp Developer") provide access credentials for the separate role that can perform actual provisioning, but implementing it felt a bit too complex to me at the time I was working on it. As a result, a policy for that purpose was created directly for the role that the workflow will assume.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:ListInstanceProfilesForRole",
        "iam:DeleteRolePolicy",
        "iam:ListAttachedRolePolicies",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:UpdateRole",
        "iam:PutRolePolicy",
        "iam:ListRolePolicies",
        "iam:GetRolePolicy"
      ],
      "Resource": [
        "arn:aws:iam::<account ID>:role/terraform-ente"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iam:GetOpenIDConnectProvider",
      "Resource": "arn:aws:iam::<account ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    {
      "Effect": "Allow",
      "Action": "iam:ListOpenIDConnectProviders",
      "Resource": "arn:aws:iam::<account ID>:oidc-provider/*"
    }
  ]
}
```

The role was then created which, with the policy attached, can create new roles and policies.

```
[lyuk98@framework:~]$ edit terraform-ente-bootstrap-policy.json
[lyuk98@framework:~]$ aws iam create-policy \
  --policy-name terraform-ente-bootstrap \
  --policy-document file://terraform-ente-bootstrap-policy.json
[lyuk98@framework:~]$ aws iam attach-role-policy \
  --role-name terraform-ente-bootstrap \
  --policy-arn arn:aws:iam::<account ID>:policy/terraform-ente-bootstrap
```

<h2 id="bootstrap-imperative-aws-auth">Setting up Vault's AWS authentication method</h2>

I have noticed that one of several ways Vault can grant access to clients is by [checking if they have right IAM credentials from AWS](https://developer.hashicorp.com/vault/docs/auth/aws "AWS - Auth Methods | Vault | HashiCorp Developer"). Since GitHub Actions will have access to the cloud provider, I did not have to manage separate credentials for Vault.

For Vault to use the AWS API, and eventually for me to use the authentication method, I had to provide IAM credentials. Using access keys was not favourable, but since the other option, using [plugin Workload Identity Federation (WIF)](https://developer.hashicorp.com/vault/docs/auth/aws#plugin-workload-identity-federation-wif "AWS - Auth Methods | Vault | HashiCorp Developer"), was only available with Vault Enterprise, I went with the former option.

Before creating a user, an IAM group was created.

```
[lyuk98@framework:~]$ aws iam create-group --group-name vault-iam-group
```

[An example](https://developer.hashicorp.com/vault/docs/auth/aws#recommended-vault-iam-policy "AWS - Auth Methods | Vault | HashiCorp Developer") showed the recommended IAM policy for Vault, so I took it and modified it a bit. In particular, the `sts:AssumeRole` stanza was removed since [cross account access](https://developer.hashicorp.com/vault/docs/auth/aws#cross-account-access "AWS - Auth Methods | Vault | HashiCorp Developer") was not needed in my case.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "iam:GetInstanceProfile",
        "iam:GetUser",
        "iam:GetRole"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageOwnAccessKeys",
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:GetAccessKeyLastUsed",
        "iam:GetUser",
        "iam:ListAccessKeys",
        "iam:UpdateAccessKey"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

The policy document was created and attached to the new IAM group.

```
[lyuk98@framework:~]$ edit vault-auth-iam-policy.json
[lyuk98@framework:~]$ aws iam create-policy \
  --policy-name vault-auth-iam-policy \
  --policy-document file://vault-auth-iam-policy.json
[lyuk98@framework:~]$ aws iam attach-group-policy \
  --group-name vault-iam-group \
  --policy-arn arn:aws:iam::<account ID>:policy/vault-auth-iam-policy
```

A new IAM user was then created and added to the IAM group.

```
[lyuk98@framework:~]$ aws iam create-user --user-name vault-iam-user
[lyuk98@framework:~]$ aws iam add-user-to-group \
  --group-name vault-iam-group \
  --user-name vault-iam-user
```

With the user present, an access key was created.

```
[lyuk98@framework:~]$ aws iam create-access-key --user-name vault-iam-user
```

Vault's AWS authentication method was subsequently enabled and the access key was provided to it.

```
[lyuk98@framework:~]$ vault auth enable aws
Success! Enabled aws auth method at: aws/
[lyuk98@framework:~]$ vault write auth/aws/config/client \
  access_key=<AWS access key ID> \
  secret_key=<AWS secret access key>
Success! Data written to: auth/aws/config/client
```

<h2 id="bootstrap-imperative-aws-role">Creating a policy and a role for AWS authentication</h2>

I have written enough policies already, but there had to be another. This time, it was for Vault: I had to limit access to just the necessary resources that Terraform needs. Specifically, clients could only read some secrets, write some for actual provisioning, and create new policies.

```hcl
# Permission to create child tokens
path "auth/token/create" {
  capabilities = ["create", "update"]
}

# Permission to access mounts
path "sys/mounts/auth/aws" {
  capabilities = ["read"]
}

path "sys/mounts/auth/token" {
  capabilities = ["read"]
}

# Permission to read keys for bootstrapping
path "kv/ente/bootstrap" {
  capabilities = ["read"]
}

path "kv/ente/b2/tfstate-bootstrap" {
  capabilities = ["read"]
}

# Permission to read and modify other B2 application keys for bootstrapping
path "kv/ente/b2/tfstate-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

# Permission to read and modify application keys for cloud providers
path "kv/ente/b2/terraform-b2-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

# Permission to read and modify AWS roles
path "auth/aws/role/terraform-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

# Permission to read and modify policies
path "sys/policies/acl/terraform-vault-auth-token-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

path "sys/policies/acl/terraform-state-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

path "sys/policies/acl/terraform-vault-acl-ente" {
  capabilities = ["create", "read", "update", "delete"]
}

path "sys/policies/acl/terraform-aws-museum" {
  capabilities = ["create", "read", "update", "delete"]
}

path "sys/policies/acl/terraform-b2-ente" {
  capabilities = ["create", "read", "update", "delete"]
}
```

The policy was saved into Vault.

```
[lyuk98@framework:~]$ edit terraform-ente-bootstrap.hcl && vault policy fmt terraform-ente-bootstrap.hcl
Success! Formatted policy: terraform-ente-bootstrap.hcl
[lyuk98@framework:~]$ vault policy write terraform-ente-bootstrap terraform-ente-bootstrap.hcl
Success! Uploaded policy: terraform-ente-bootstrap
```

The creation of a role within the AWS auth method itself followed. Anyone successfully gaining access to Vault with this role was to be bound by the policy I just wrote above.

```
[lyuk98@framework:~]$ vault write auth/aws/role/terraform-ente-bootstrap \
  auth_type=iam \
  policies=terraform-ente-bootstrap \
  bound_iam_principal_arn=arn:aws:iam::<AWS account ID>:role/terraform-ente-bootstrap
Success! Data written to: auth/aws/role/terraform-ente-bootstrap
```

<h1 id="bootstrap-declarative">Bootstrapping (the declarative part)</h1>

I started writing down Terraform code at [a GitHub repository](https://github.com/lyuk98/terraform-ente-bootstrap "lyuk98/terraform-ente-bootstrap: Terraform configurations (bootstrapping for Ente)") I have created.

<h2 id="bootstrap-declarative-terraform">Preparing Terraform configurations</h2>

To keep track of what resources are active, the `s3` [backend was configured](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/main.tf#L2-L11 "terraform-ente-bootstrap/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"). Since alternative S3-compatible services are [only supported on a "best effort" basis](https://developer.hashicorp.com/terraform/language/backend/s3#support-for-s3-compatible-storage-providers "Backend Type: s3 | Terraform | HashiCorp Developer"), using Backblaze B2 required skipping a few checks that were for Amazon S3.

```terraform
terraform {
  backend "s3" {
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
    region                      = "us-west-002"

    use_path_style = true
    key            = "terraform-ente-bootstrap.tfstate"
  }
}
```

Setting `use_path_style` was probably not necessary, but the `region`, even if it is not checked at all, still needed to be specified. Since multiple repositories are storing states, `key` was set as where the state file will be saved. Other parameters, such as `bucket` and `access_key`, were not set here; they were instead to be separately provided during the `terraform init` stage.

[Specifications for the providers](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/providers.tf "terraform-ente-bootstrap/providers.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap") were next. I did not write any arguments for them; they were to be provided either as environment variables or files (like `~/.vault-token`).

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.13"
    }
    b2 = {
      source  = "Backblaze/b2"
      version = "~> 0.10"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 5.3"
    }
  }
}

provider "aws" {}

provider "b2" {}

provider "vault" {}
```

To make configurations separate among cloud providers, I [divided them into modules](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/main.tf#L14-L31 "terraform-ente-bootstrap/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"). In retrospect, this was perhaps not necessary, but this experience allowed me to realise just that.

```terraform
module "aws" {
  source = "./modules/aws"
}

module "b2" {
  source         = "./modules/b2"
  # Input follows...
}

module "vault" {
  source = "./modules/vault"
  # Inputs follow...
}
```

<h3 id="bootstrap-declarative-terraform-aws">Bootstrapping Amazon Web Services</h3>

The goal for [this module](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/aws/main.tf "terraform-ente-bootstrap/modules/aws/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap") was to create an IAM role that the provisioning Terraform workflow will assume. A policy to allow creation of Lightsail resources, as well as a trust policy to allow access only from specific GitHub Actions workflows, were defined.

```terraform
# Policy for Terraform configuration (Museum)
resource "aws_iam_role_policy" "ente" {
  name = "terraform-ente"
  role = aws_iam_role.ente.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeAvailabilityZones"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "lightsail:AttachDisk",
          "lightsail:DeleteInstance",
          "lightsail:PutInstancePublicPorts",
          "lightsail:StartInstance",
          "lightsail:StopInstance",
          "lightsail:DeleteKeyPair",
          "lightsail:RebootInstance",
          "lightsail:OpenInstancePublicPorts",
          "lightsail:CloseInstancePublicPorts",
          "lightsail:DeleteDisk",
          "lightsail:DetachDisk",
          "lightsail:UpdateInstanceMetadataOptions"
        ]
        Resource = [
          "arn:aws:lightsail:*:${data.aws_caller_identity.current.account_id}:Disk/*",
          "arn:aws:lightsail:*:${data.aws_caller_identity.current.account_id}:KeyPair/*",
          "arn:aws:lightsail:*:${data.aws_caller_identity.current.account_id}:Instance/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "lightsail:CreateKeyPair",
          "lightsail:ImportKeyPair",
          "lightsail:GetInstancePortStates",
          "lightsail:GetInstances",
          "lightsail:GetKeyPair",
          "lightsail:GetDisks",
          "lightsail:CreateDisk",
          "lightsail:CreateInstances",
          "lightsail:GetInstance",
          "lightsail:GetDisk",
          "lightsail:GetKeyPairs"
        ]
        Resource = "*"
      }
    ]
  })
}

# Role to assume during deployment (Terraform)
resource "aws_iam_role" "ente" {
  name = "terraform-ente"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = "sts:AssumeRoleWithWebIdentity"
        Principal = {
          Federated = "${data.aws_iam_openid_connect_provider.github.arn}"
        }
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = [
              "sts.amazonaws.com"
            ]
            "token.actions.githubusercontent.com:sub" = [
              "repo:lyuk98/terraform-ente:ref:refs/heads/main"
            ]
          }
        }
      }
    ]
  })
}
```

Some data sources, `aws_caller_identity` and `aws_iam_openid_connect_provider`, were used to substitute the account ID and ARNs.

The IAM role's ARN was declared as an [output for the module](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/aws/outputs.tf "terraform-ente-bootstrap/modules/aws/outputs.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"), which was to be taken by Vault.

```terraform
output "aws_role_arn" {
  value       = aws_iam_role.ente.arn
  description = "ARN of the IAM role for Terraform"
  sensitive   = true
}
```

<h3 id="bootstrap-declarative-terraform-b2">Bootstrapping Backblaze B2</h3>

With [this module](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/b2/main.tf "terraform-ente-bootstrap/modules/b2/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"), two application keys were defined: one for creating buckets and another for accessing Terraform state file.

```terraform
locals {
  # Capabilities of application keys for accessing Terraform state files
  state_key_capabilities = [
    "deleteFiles",
    "listBuckets",
    "listFiles",
    "readFiles",
    "writeFiles"
  ]
}

# Key for creating buckets (Backblaze B2)
resource "b2_application_key" "terraform_b2_ente" {
  capabilities = [
    "deleteBuckets",
    "deleteKeys",
    "listBuckets",
    "listKeys",
    "readBucketEncryption",
    "writeBucketEncryption",
    "writeBucketRetentions",
    "writeBuckets",
    "writeKeys"
  ]
  key_name = "terraform-b2-ente"
}

# Get information about the bucket to store state
data "b2_bucket" "terraform_state" {
  bucket_name = var.tfstate_bucket
}

# Key for accessing Terraform state
resource "b2_application_key" "terraform_state_ente" {
  capabilities = local.state_key_capabilities
  key_name     = "terraform-state-ente"
  bucket_id    = data.b2_bucket.terraform_state.bucket_id
  name_prefix  = "terraform-ente"
}
```

Since it is not aware of which bucket (for Terraform state) to create an application key against, the name was [declared as a variable](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/b2/variables.tf "terraform-ente-bootstrap/modules/b2/variables.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"), which another module (Vault) was going to provide.

```terraform
variable "tfstate_bucket" {
  type        = string
  description = "Name of the bucket to store Terraform state"
  sensitive   = true
}
```

To let Vault save application keys, they were [set as outputs](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/b2/outputs.tf "terraform-ente-bootstrap/modules/b2/outputs.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap").

```terraform
output "application_key_b2" {
  value       = b2_application_key.terraform_b2_ente.application_key
  description = "Application Key for Backblaze B2"
  sensitive   = true
}

output "application_key_id_b2" {
  value       = b2_application_key.terraform_b2_ente.application_key_id
  description = "Application Key ID for Backblaze B2"
  sensitive   = true
}

output "application_key_tfstate_ente" {
  value       = b2_application_key.terraform_state_ente.application_key
  description = "Application Key for accessing Terraform state"
  sensitive   = true
}

output "application_key_id_tfstate_ente" {
  value       = b2_application_key.terraform_state_ente.application_key_id
  description = "Application Key ID for accessing Terraform state"
  sensitive   = true
}
```

<h3 id="bootstrap-declarative-terraform-vault">Bootstrapping Vault</h3>

Vault had to write information that providers for AWS and Backblaze B2 produced, so they were first set as variables.

```terraform
variable "b2_application_key" {
  type        = string
  description = "Application Key for Backblaze B2"
  sensitive   = true
}

variable "b2_application_key_id" {
  type        = string
  description = "Application Key ID for Backblaze B2"
  sensitive   = true
}

variable "b2_application_key_tfstate_ente" {
  type        = string
  description = "Application Key for accessing Terraform state"
  sensitive   = true
}

variable "b2_application_key_id_tfstate_ente" {
  type        = string
  description = "Application Key ID for accessing Terraform state"
  sensitive   = true
}

variable "aws_role_arn" {
  type        = string
  description = "ARN of the IAM role for Terraform"
  sensitive   = true
}
```

The bucket name [that I have previously saved](#bootstrap-imperative-b2) in Vault [was retrieved](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/vault/main.tf#L10-L13 "terraform-ente-bootstrap/modules/vault/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap") and [was set as an output](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/vault/outputs.tf "terraform-ente-bootstrap/modules/vault/outputs.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap"), so that the object storage provider can create an application key with it.

```terraform
# Read Vault for Terraform state storage information
data "vault_kv_secret" "terraform_state" {
  path = "kv/ente/b2/tfstate-bootstrap"
}

output "terraform_state" {
  value = {
    bucket   = data.vault_kv_secret.terraform_state.data.b2_bucket
    endpoint = data.vault_kv_secret.terraform_state.data.b2_endpoint
  }
  description = "Terraform state storage information"
  sensitive   = true
}
```

The application keys were then declared to be written to the secret storage.

```terraform
# Application Key for Backblaze B2
resource "vault_kv_secret" "application_key_b2" {
  path = "kv/ente/b2/terraform-b2-ente"
  data_json = jsonencode({
    application_key    = var.b2_application_key,
    application_key_id = var.b2_application_key_id
  })
}

# Application Key for accessing Terraform state
resource "vault_kv_secret" "application_key_tfstate_ente" {
  path = "kv/ente/b2/tfstate-ente"
  data_json = jsonencode({
    application_key    = var.b2_application_key_tfstate_ente
    application_key_id = var.b2_application_key_id_tfstate_ente
  })
}
```

New policies for Vault were then defined. I used data source `vault_policy_document` to prepare policy documents without having to write them as multiline strings.

```terraform
# Vault policy document (Vault token)
data "vault_policy_document" "auth_token" {
  rule {
    path         = "auth/token/create"
    capabilities = ["create", "update"]
    description  = "Allow creating child tokens"
  }
}

# Vault policy document (Terraform state)
data "vault_policy_document" "terraform_state" {
  rule {
    path         = data.vault_kv_secret.terraform_state.path
    capabilities = ["read"]
    description  = "Allow reading Terraform state information"
  }
  rule {
    path         = vault_kv_secret.application_key_tfstate_ente.path
    capabilities = ["read"]
    description  = "Allow reading B2 application key for writing Terraform state"
  }
}

# Vault policy document (Vault policy)
data "vault_policy_document" "acl" {
  rule {
    path         = "sys/policies/acl/museum"
    capabilities = ["create", "read", "update", "delete"]
    description  = "Allow creation of policy to access credentials for Museum"
  }
}

# Vault policy document (Museum)
data "vault_policy_document" "aws_museum" {
  rule {
    path         = "sys/mounts/auth/approle"
    capabilities = ["read"]
    description  = "Allow reading configuration of AppRole authentication method"
  }
  rule {
    path         = "kv/ente/aws/museum"
    capabilities = ["create", "read", "update", "delete"]
    description  = "Allow creation of credentials for Museum"
  }
  rule {
    path         = "kv/ente/cloudflare/certificate"
    capabilities = ["read"]
    description  = "Allow reading certificate and certificate key"
  }
  rule {
    path         = "auth/approle/role/museum"
    capabilities = ["create", "read", "update", "delete"]
    description  = "Allow creation of AppRole for Museum"
  }
  rule {
    path         = "auth/approle/role/museum/*"
    capabilities = ["create", "read", "update", "delete"]
    description  = "Allow access to AppRole information for Museum"
  }
}

# Vault policy document (Backblaze B2)
data "vault_policy_document" "b2_ente" {
  rule {
    path         = vault_kv_secret.application_key_b2.path
    capabilities = ["read"]
    description  = "Allow reading application key for Backblaze B2"
  }
  rule {
    path         = "kv/ente/b2/ente-b2"
    capabilities = ["create", "read", "update", "delete"]
    description  = "Allow creation of access credentials for Backblaze B2"
  }
}
```

The policy documents were then used to create actual Vault policies.

```terraform
# Policy to grant creation of child tokens
resource "vault_policy" "auth_token" {
  name   = "terraform-vault-auth-token-ente"
  policy = data.vault_policy_document.auth_token.hcl
}

# Policy to grant access to Terraform state information
resource "vault_policy" "terraform_state" {
  name   = "terraform-state-ente"
  policy = data.vault_policy_document.terraform_state.hcl
}

# Policy to grant creation of a Vault role
resource "vault_policy" "acl" {
  name   = "terraform-vault-acl-ente"
  policy = data.vault_policy_document.acl.hcl
}

# Policy to write (Museum)
resource "vault_policy" "aws_museum" {
  name   = "terraform-aws-museum"
  policy = data.vault_policy_document.aws_museum.hcl
}

# Policy to write (Backblaze B2)
resource "vault_policy" "b2" {
  name   = "terraform-b2-ente"
  policy = data.vault_policy_document.b2_ente.hcl
}
```

Lastly, a role for the actual provisioning workflow to authenticate against [was defined](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/modules/vault/main.tf#L143-L156 "terraform-ente-bootstrap/modules/vault/main.tf at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap").

```terraform
# Mount AWS authentication backend
data "vault_auth_backend" "aws" {
  path = "aws"
}

# Vault role for Terraform configurations
resource "vault_aws_auth_backend_role" "ente" {
  backend   = data.vault_auth_backend.aws.path
  role      = "terraform-ente"
  auth_type = "iam"
  token_policies = [
    vault_policy.auth_token.name,
    vault_policy.terraform_state.name,
    vault_policy.acl.name,
    vault_policy.aws_museum.name,
    vault_policy.b2.name
  ]
  bound_iam_principal_arns = [var.aws_role_arn]
}
```

<h2 id="bootstrap-declarative-github">Deploying with GitHub Actions</h2>

Defining GitHub Actions workflow was the last in step. There were some secrets that were needed *before* authenticating to Vault, which were added as [repository secrets](https://docs.github.com/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets "Using secrets in GitHub Actions - GitHub Docs").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/1f7b6140-878e-4e34-a2c1-17e8e5506bbc.avif">
  <img src="https://images.lyuk98.com/76f78a06-87ef-4ea9-8965-b8523ed8bc90.avif" alt="An interface for managing Actions secrets and variables. Repository secrets are populated." title="The repository secrets">
</picture>

- `AWS_ROLE_TO_ASSUME` to the ARN of the IAM role [created earlier](#bootstrap-imperative-github)
- `TS_OAUTH_CLIENT_ID` and `TS_OAUTH_SECRET` to a Tailscale OAuth client with `auth_keys` scope, which I have [separately created](https://login.tailscale.com/admin/settings/oauth "OAuth clients - Tailscale")
- `VAULT_ADDR` to the address of the Vault instance

I wrote [a workflow file](https://github.com/lyuk98/terraform-ente-bootstrap/blob/88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55/.github/workflows/deploy.yml "terraform-ente-bootstrap/.github/workflows/deploy.yml at 88f7c82a2e4304bdfff368b96f5d4ab98e8ccd55 · lyuk98/terraform-ente-bootstrap") afterwards, starting with some basic configurations. The secret `VAULT_ADDR` was set as an environment variable all throughout the deployment process.

```yaml
name: Deploy Terraform configuration

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: Deploy Terraform configuration
    permissions:
      id-token: write
    env:
      VAULT_ADDR: ${{ secrets.VAULT_ADDR }}

    steps:
```

For first steps, the runner was to authenticate to Tailscale and AWS.

```yaml
- name: Set up Tailscale
  uses: tailscale/github-action@v3
  with:
    oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
    oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
    tags: tag:ci

- name: Configure AWS credentials
  id: aws-credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-region: us-east-1
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    output-credentials: true
    mask-aws-account-id: true
```

Vault was to be installed next. I wanted to use [Vault's own GitHub Action](https://github.com/marketplace/actions/hashicorp-vault "HashiCorp Vault · Actions · GitHub Marketplace"), but support for AWS authentication method [was not present](https://github.com/hashicorp/vault-action/issues/199 "[FEAT] Support IAM / EC2 auth methods · Issue #199 · hashicorp/vault-action") at the time I was working on this; taking the naive approach, I followed [an installation guide](https://developer.hashicorp.com/vault/install#linux "Install | Vault | HashiCorp Developer") to install the software.

```yaml
- name: Install Vault
  run: |
    wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt-get update && sudo apt-get install vault
```

With Vault present, logging in to the service and reading some secrets, which are to be masked using `add-mask` [workflow command](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands "Workflow commands for GitHub Actions - GitHub Docs"), followed. Some were to be set as environment variables, and others as outputs that succeeding steps can refer to.

```yaml
- name: Log in to Vault
  run: |
    vault login \
      -no-print \
      -method=aws \
      region=us-east-1 \
      role=terraform-ente-bootstrap \
      aws_access_key_id=${{ steps.aws-credentials.outputs.aws-access-key-id }} \
      aws_secret_access_key=${{ steps.aws-credentials.outputs.aws-secret-access-key }} \
      aws_security_token=${{ steps.aws-credentials.outputs.aws-session-token }}

- name: Get secrets from Vault
  id: vault-secrets
  run: |
    b2_application_key_id=$(vault kv get -field=b2_application_key_id -mount=kv ente/bootstrap)
    b2_application_key=$(vault kv get -field=b2_application_key -mount=kv ente/bootstrap)

    echo "::add-mask::$b2_application_key_id"
    echo "::add-mask::$b2_application_key"

    b2_state_application_key_id=$(vault kv get -field=b2_state_application_key_id -mount=kv ente/bootstrap)
    b2_state_application_key=$(vault kv get -field=b2_state_application_key -mount=kv ente/bootstrap)
    b2_state_bucket=$(vault kv get -field=b2_bucket -mount=kv ente/b2/tfstate-bootstrap)
    b2_state_endpoint=$(vault kv get -field=b2_endpoint -mount=kv ente/b2/tfstate-bootstrap)

    echo "::add-mask::$b2_state_application_key_id"
    echo "::add-mask::$b2_state_application_key"
    echo "::add-mask::$b2_state_bucket"
    echo "::add-mask::$b2_state_endpoint"

    echo "B2_APPLICATION_KEY_ID=$b2_application_key_id" >> $GITHUB_ENV
    echo "B2_APPLICATION_KEY=$b2_application_key" >> $GITHUB_ENV

    echo "b2-state-application-key-id=$b2_state_application_key_id" >> $GITHUB_OUTPUT
    echo "b2-state-application-key=$b2_state_application_key" >> $GITHUB_OUTPUT
    echo "b2-state-bucket=$b2_state_bucket" >> $GITHUB_OUTPUT
    echo "b2-state-endpoint=$b2_state_endpoint" >> $GITHUB_OUTPUT
```

After checking out the repository, setting up Terraform was next. Initialising Terraform would be done with secrets that were saved in Vault.

```yaml
- uses: actions/checkout@v4
  with:
    ref: main

- name: Set up Terraform
  uses: hashicorp/setup-terraform@v3

- name: Terraform fmt
  id: terraform-fmt
  run: terraform fmt -check
  continue-on-error: true

- name: Terraform init
  id: terraform-init
  run: |
    terraform init \
      -backend-config="bucket=${{ steps.vault-secrets.outputs.b2-state-bucket }}" \
      -backend-config="endpoints={s3=\"${{ steps.vault-secrets.outputs.b2-state-endpoint }}\"}" \
      -backend-config="access_key=${{ steps.vault-secrets.outputs.b2-state-application-key-id }}" \
      -backend-config="secret_key=${{ steps.vault-secrets.outputs.b2-state-application-key }}" \
      -input=false

- name: Terraform validate
  id: terraform-validate
  run: terraform validate
```

Lastly, creating a plan and applying it would mark the end of the workflow.

```yaml
- name: Terraform plan
  id: terraform-plan
  run: terraform plan -input=false -out=tfplan

- name: Terraform apply
  id: terraform-apply
  run: terraform apply -input=false tfplan
```

<h1 id="ente-imperative">Deploying Ente (the imperative part)</h1>

Most of the access credentials and whatnot to deploy Ente was ready, but there were a few things that I ended up creating manually. Those resources, which are from Cloudflare and Tailscale, were either:

- [complex to implement](https://registry.terraform.io/providers/cloudflare/cloudflare/5.10.1/docs/resources/api_token "cloudflare_api_token | Resources | cloudflare/cloudflare | Terraform | Terraform Registry") without knowing more about the provider's API specification
- [impossible to apply](https://registry.terraform.io/providers/cloudflare/cloudflare/5.10.1/docs/resources/ruleset "cloudflare_ruleset | Resources | cloudflare/cloudflare | Terraform | Terraform Registry"), because every rule for my domain would then need to be managed by Terraform
- [outright discouraging me](https://registry.terraform.io/providers/hashicorp/tls/4.1.0/docs/resources/private_key "tls_private_key | Resources | hashicorp/tls | Terraform | Terraform Registry") to use one
- [perfectly fine](https://registry.terraform.io/providers/tailscale/tailscale/0.22.0/docs/resources/oauth_client "tailscale_oauth_client | Resources | tailscale/tailscale | Terraform | Terraform Registry"), but I was unable to motivate myself to manage one through Terraform

<h2 id="ente-imperative-cloudflare">Preparing Cloudflare</h2>

<h3 id="ente-imperative-cloudflare-token">Creating an API token</h3>

A token was needed to make DNS records. I could easily create one by going to the [Cloudflare dashboard](https://dash.cloudflare.com/profile/api-tokens "API Tokens | Cloudflare").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/8b1c6e47-292d-4c41-9f41-6633b95c03c0.avif">
  <img src="https://images.lyuk98.com/b67b7bab-f762-48f5-9ab8-2e8206ccb0f2.avif" alt="Creation of a user API token, where the name is terraform-ente and a permission for editing DNS information is granted" title="Creation of a user API token">
</picture>

<h3 id="ente-imperative-cloudflare-cert">Creating an origin certificate</h3>

Using [proxied DNS records](https://developers.cloudflare.com/dns/proxy-status/#proxied-records "Proxy status · Cloudflare DNS docs") from Cloudflare was something I could tolerate. With encryption mode for SSL/TLS also set as [Full (strict)](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/ "Full (strict) - SSL/TLS encryption modes · Cloudflare SSL/TLS docs"), my method for achieving HTTPS support became a bit different from usual (which most likely involves [Certbot](https://certbot.eff.org/ "Certbot")).

The traffic between users and Cloudflare was already being encrypted, and I ensured the same was true for the traffic between Cloudflare and my server by creating [an origin certificate](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/ "Cloudflare origin CA · Cloudflare SSL/TLS docs").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/bd8d6030-eb90-406c-b26e-8c7e240feded.avif">
  <img src="https://images.lyuk98.com/c77c41b7-9656-422b-8981-c5e0aea47812.avif" alt="Creation of an origin certificate. An option, Generate private key and CSR with Cloudflare, is selected, and the private key type is set to ECC." title="Creation of an origin certificate">
</picture>

Hostnames included [all subdomains](#ente-declarative-terraform-cloudflare) that the service required. When one was created, the certificate, along with its private key, were saved into Vault.

```
[lyuk98@framework:~]$ edit certificate
[lyuk98@framework:~]$ edit certificate_key
[lyuk98@framework:~]$ vault kv put -mount=kv ente/cloudflare/certificate certificate=@certificate certificate_key=@certificate_key
Success! Data written to: kv/ente/cloudflare/certificate
```

<h2 id="ente-imperative-tailscale">Creating an OAuth client at Tailscale</h2>

One was needed for Terraform to create what the instance itself will use to connect to the tailnet. The catch here, though, was that the scope (`auth_keys`) and tags (`tag:webserver` and `tag:museum`) that Museum itself will be using, on top of the scope needed to do the job (`oauth_keys`), were apparently needed.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/6922538b-f3af-4ba8-9f3c-b7f37d41cb28.avif">
  <img src="https://images.lyuk98.com/b425e8b7-a3e1-426d-81b7-d056ed0c8eaf.avif" alt="An interface for adding an OAuth client at Tailscale. Write access to scopes OAuth Keys and Auth Keys is enabled, and for tags, tag:webserver and tag:museum are added." title="Creating a new OAuth client">
</picture>

<h1 id="ente-declarative">Deploying Ente (the declarative part)</h1>

<h2 id="ente-declarative-terraform">Writing Terraform configurations</h2>

Within [a repository](https://github.com/lyuk98/terraform-ente "lyuk98/terraform-ente: Terraform configurations (Ente)") I have created, I wrote Terraform configurations for deploying Ente. Unlike the last time, I decided not to take the modular approach, instead using multiple files for each cloud provider.

[Variables were first declared](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/variables.tf "terraform-ente/variables.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente"). They were later going to be provided via GitHub Actions.

```terraform
variable "aws_region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region"
  sensitive   = true
}

variable "cloudflare_zone_id" {
  type        = string
  description = "Zone ID for Cloudflare domain"
  sensitive   = true
}
```

Necessary providers were next in line. For Tailscale, the scopes that will be utilised during the deployment were required to be specified.

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.13"
    }
    b2 = {
      source  = "Backblaze/b2"
      version = "~> 0.10"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 5.10"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
    tailscale = {
      source  = "tailscale/tailscale"
      version = "~> 0.21"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 5.3"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "b2" {}

provider "cloudflare" {}

provider "random" {}

provider "tailscale" {
  scopes = ["oauth_keys", "auth_keys"]
}

provider "vault" {}
```

Backend configuration was almost the same with the [bootstrapping configuration](#bootstrap-declarative-terraform), except for the `key`.

```terraform
terraform {
  backend "s3" {
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
    region                      = "us-west-002"

    use_path_style = true
    key            = "terraform-ente.tfstate"
  }
}
```

<h3 id="ente-declarative-terraform-tailscale">Tailscale</h3>

The Tailscale provider was used to do one thing: [creating an OAuth client](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/tailscale.tf "terraform-ente/tailscale.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") for the server.

```terraform
# Create a Tailscale OAuth client
resource "tailscale_oauth_client" "museum" {
  description = "museum"
  scopes      = ["auth_keys"]
  tags        = ["tag:museum", "tag:webserver"]
}
```

<h3 id="ente-declarative-terraform-b2">Backblaze B2</h3>

[An object storage bucket](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/b2.tf#L9-L28 "terraform-ente/b2.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") for storing photos was declared. To make its name unique, a random suffix was appended. [A CORS rule](https://help.ente.io/self-hosting/administration/object-storage#cors-cross-origin-resource-sharing "Configuring Object Storage - Self-hosting | Ente Help") was added as well, since I was not able to download photos without it.

```terraform
# Add random suffix to bucket name
resource "random_bytes" "bucket_suffix" {
  length = 4
}

# Add bucket for photo storage
resource "b2_bucket" "ente" {
  bucket_name = sensitive("ente-${random_bytes.bucket_suffix.hex}")
  bucket_type = "allPrivate"

  cors_rules {
    allowed_operations = [
      "s3_head",
      "s3_put",
      "s3_delete",
      "s3_post",
      "s3_get"
    ]
    allowed_origins = ["*"]
    cors_rule_name  = "ente-cors-rule"
    max_age_seconds = 3000
    allowed_headers = ["*"]
    expose_headers  = ["Etag"]
  }
}
```

For Ente to be able to access the bucket, [an application key](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/b2.tf#L30-L44 "terraform-ente/b2.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") was also declared.

```terraform
# Create application key for accessing the bucket
resource "b2_application_key" "ente" {
  capabilities = [
    "bypassGovernance",
    "deleteFiles",
    "listFiles",
    "readFiles",
    "shareFiles",
    "writeFileLegalHolds",
    "writeFileRetentions",
    "writeFiles"
  ]
  key_name  = "ente"
  bucket_id = b2_bucket.ente.bucket_id
}
```

<h3 id="ente-declarative-terraform-vault">Vault</h3>

[Random bytes](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L80-L92 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") that are generated within Terraform [were set to be saved](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/vault.tf#L1-L13 "terraform-ente/vault.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") into Vault.

```terraform
# Encryption key for Museum
resource "random_bytes" "encryption_key" {
  length = 32
}

resource "random_bytes" "encryption_hash" {
  length = 64
}

# JWT secrets
resource "random_bytes" "jwt_secret" {
  length = 32
}

# Write secret containing connection details
resource "vault_kv_secret" "museum" {
  path = "kv/ente/aws/museum"
  data_json = jsonencode({
    key = {
      encryption = random_bytes.encryption_key.base64
      hash       = random_bytes.encryption_hash.base64
    }
    jwt = {
      secret = random_bytes.jwt_secret.base64
    }
  })
}
```

The same applied to [the application key](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/vault.tf#L15-L25 "terraform-ente/vault.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") for the object storage, which the instance will be using.

```terraform
# Write application key to Vault
resource "vault_kv_secret" "application_key" {
  path = "kv/ente/b2/ente-b2"
  data_json = jsonencode({
    key      = b2_application_key.ente.application_key_id
    secret   = b2_application_key.ente.application_key
    endpoint = data.b2_account_info.account.s3_api_url
    region   = "us-west-002"
    bucket   = b2_bucket.ente.bucket_name
  })
}
```

What followed was [a policy](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/vault.tf#L27-L50 "terraform-ente/vault.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") granting access to the credentials as well as the origin certificate that I have manually created.

```terraform
# Prepare policy document
data "vault_policy_document" "museum" {
  rule {
    path         = vault_kv_secret.museum.path
    capabilities = ["read"]
    description  = "Allow access to credentials for Museum"
  }
  rule {
    path         = vault_kv_secret.application_key.path
    capabilities = ["read"]
    description  = "Allow access to secrets for object storage"
  }
  rule {
    path         = "kv/ente/cloudflare/certificate"
    capabilities = ["read"]
    description  = "Allow access to TLS certificate data"
  }
}

# Write policy allowing Museum to read secrets
resource "vault_policy" "museum" {
  name   = "museum"
  policy = data.vault_policy_document.museum.hcl
}
```

Lastly, [an AppRole and a SecretID](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/vault.tf#L57-L68 "terraform-ente/vault.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") for authorising Museum access to Vault was set to be created.

```terraform
# Mount AppRole auth backend
data "vault_auth_backend" "approle" {
  path = "approle"
}

# Create an AppRole for Museum to retrieve secrets with
resource "vault_approle_auth_backend_role" "museum" {
  backend        = data.vault_auth_backend.approle.path
  role_name      = "museum"
  token_policies = [vault_policy.museum.name]
}

# Create a SecretID for the Vault AppRole
resource "vault_approle_auth_backend_role_secret_id" "museum" {
  backend   = vault_approle_auth_backend_role.museum.backend
  role_name = vault_approle_auth_backend_role.museum.role_name
}
```

<h3 id="ente-declarative-terraform-aws">Amazon Web Services</h3>

[A Lightsail disk](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L11-L16 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") was declared to be used as PostgreSQL database storage.

```terraform
# Create a persistent disk for PostgreSQL storage
resource "aws_lightsail_disk" "postgresql" {
  name              = "ente-postgresql"
  size_in_gb        = 8
  availability_zone = aws_lightsail_instance.museum.availability_zone
}
```

[An SSH key pair](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L18-L21 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") was declared as well. This would later allow [nixos-anywhere](https://github.com/nix-community/nixos-anywhere "nix-community/nixos-anywhere: Install NixOS everywhere via SSH [maintainers=@Mic92 @Lassulus @phaer @Enzime @a-kenji]") to access the host, but would no longer be used after the installation.

```terraform
# Add a public SSH key
resource "aws_lightsail_key_pair" "museum" {
  name = "museum-key-pair"
}
```

[A dual-stack Lightsail instance](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L23-L31 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") to host Ente was then specified. A small script (`user_data`) allows logging in as `root` via SSH, which is what nixos-anywhere [currently assumes to be the case](https://github.com/nix-community/nixos-anywhere/pull/550#discussion_r2136060869 "make installer with sudo-only access work and re-enable ssh-ng by Mic92 · Pull Request #550 · nix-community/nixos-anywhere").

```terraform
# Create a Lightsail instance
resource "aws_lightsail_instance" "museum" {
  name              = "museum"
  availability_zone = data.aws_availability_zones.available.names[0]
  blueprint_id      = "debian_12"
  bundle_id         = "small_3_0"
  key_pair_name     = aws_lightsail_key_pair.museum.name
  user_data         = "cp /home/admin/.ssh/authorized_keys /root/.ssh/authorized_keys"
}
```

With both the instance and the disk ready, the latter [would be attached](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L33-L52 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") to the former. Choosing `disk_path` did not matter, since it is not recognised during installation and I had to resort to `/dev/nvme1n1`, anyway.

```terraform
# Attach disk to the instance
resource "aws_lightsail_disk_attachment" "ente_postgresql" {
  disk_name     = aws_lightsail_disk.postgresql.name
  instance_name = aws_lightsail_instance.museum.name
  disk_path     = "/dev/xvdf"

  # Recreate disk attachment upon replacement of either the instance or the disk
  lifecycle {
    replace_triggered_by = [
      aws_lightsail_instance.museum.created_at,
      aws_lightsail_disk.postgresql.created_at
    ]
  }

  # Mark disk attachment dependent on the instance and the disk
  depends_on = [
    aws_lightsail_instance.museum,
    aws_lightsail_disk.postgresql
  ]
}
```

This attachment relied on names of dependencies, which do not change even if resources are replaced. Therefore, I manually added a condition to trigger a replacement. `depends_on` was also manually specified, so that the attachment will happen *after* the creation of both dependencies.

[Three public ports](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L54-L78 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") (for SSH, HTTPS, and Tailscale) for the instance were declared afterwards.

```terraform
# Open instance ports
resource "aws_lightsail_instance_public_ports" "museum" {
  instance_name = aws_lightsail_instance.museum.name

  # SSH
  port_info {
    protocol  = "tcp"
    from_port = 22
    to_port   = 22
  }

  # HTTPS
  port_info {
    protocol  = "tcp"
    from_port = 443
    to_port   = 443
  }

  # Tailscale
  port_info {
    protocol  = "udp"
    from_port = 41641
    to_port   = 41641
  }
}
```

Finally, [nixos-anywhere was set up](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/aws.tf#L94-L127 "terraform-ente/aws.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") to install NixOS to the new Lightsail instance.

```terraform
# Build the NixOS system configuration
module "nixos_system" {
  source    = "github.com/nix-community/nixos-anywhere//terraform/nix-build"
  attribute = "github:lyuk98/nixos-config#nixosConfigurations.museum.config.system.build.toplevel"
}

# Build the NixOS partition layout
module "nixos_partitioner" {
  source    = "github.com/nix-community/nixos-anywhere//terraform/nix-build"
  attribute = "github:lyuk98/nixos-config#nixosConfigurations.museum.config.system.build.diskoScript"
}

# Install NixOS to Lightsail instance
module "nixos_install" {
  source = "github.com/nix-community/nixos-anywhere//terraform/install"

  nixos_system      = module.nixos_system.result.out
  nixos_partitioner = module.nixos_partitioner.result.out

  target_host     = aws_lightsail_instance.museum.public_ip_address
  target_user     = aws_lightsail_instance.museum.username
  ssh_private_key = aws_lightsail_key_pair.museum.private_key

  instance_id = aws_lightsail_instance.museum.public_ip_address

  extra_files_script = "${path.module}/deploy-secrets.sh"
  extra_environment = {
    MODULE_PATH                   = path.module
    VAULT_ROLE_ID                 = vault_approle_auth_backend_role.museum.role_id
    VAULT_SECRET_ID               = vault_approle_auth_backend_role_secret_id.museum.secret_id
    TAILSCALE_OAUTH_CLIENT_ID     = tailscale_oauth_client.museum.id
    TAILSCALE_OAUTH_CLIENT_SECRET = tailscale_oauth_client.museum.key
  }
}
```

The `extra_files_script` was specified to save secrets that will be used to unlock Vault. I passed the data as `extra_environment`, which the script would later pick up as environment variables.

Secrets are written inside `/persist` since it is where persistent data are saved with [impermanence](https://github.com/nix-community/impermanence "nix-community/impermanence: Modules to help you handle persistent state on systems with ephemeral root storage [maintainer=@talyz]") applied.

```sh
#!/usr/bin/env bash

install --directory --mode 751 persist/var/lib/secrets

secrets_dir="$MODULE_PATH/secrets"
mkdir --parents "$secrets_dir"

echo -n $VAULT_ROLE_ID > "$secrets_dir/vault-role-id"
echo -n $VAULT_SECRET_ID > "$secrets_dir/vault-secret-id"
echo -n $TAILSCALE_OAUTH_CLIENT_ID > "$secrets_dir/tailscale-oauth-client-id"
echo -n $TAILSCALE_OAUTH_CLIENT_SECRET > "$secrets_dir/tailscale-oauth-client-secret"

install --mode 600 \
  "$secrets_dir/vault-role-id" \
  "$secrets_dir/vault-secret-id" \
  "$secrets_dir/tailscale-oauth-client-id" \
  "$secrets_dir/tailscale-oauth-client-secret" \
  persist/var/lib/secrets
```

I edited the script and made sure the GitHub Actions runner can execute it.

```
[lyuk98@framework:~/terraform-ente]$ edit deploy-secrets.sh
[lyuk98@framework:~/terraform-ente]$ chmod +x deploy-secrets.sh
```

<h3 id="ente-declarative-terraform-cloudflare">Cloudflare</h3>

DNS records to access various parts of the service [were defined](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/cloudflare.tf "terraform-ente/cloudflare.tf at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente"). I initially wanted to use second-level subdomains (such as `albums.ente.<domain>`), but later realised that Cloudflare's Universal SSL certificate only [covers root and first-level subdomains](https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/limitations/ "Limitations for Universal SSL · Cloudflare SSL/TLS docs"). Purchasing [Advanced Certificate Manager](https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/#advanced-certificate-manager "Advanced certificates · Cloudflare SSL/TLS docs") was not ideal, so I ended up using first-level subdomains (like `ente-albums.<domain>`).

The following subdomains were defined:

- `ente-accounts`: Ente Accounts
- `ente-cast`: Ente Cast
- `ente-albums`: Ente Albums
- `ente-photos`: Ente Photos
- `ente-api`: Ente API, also known as Museum

An `A` record was written like the following:

```terraform
# A record for Ente accounts
resource "cloudflare_dns_record" "accounts" {
  name    = "ente-accounts"
  ttl     = 1
  type    = "A"
  zone_id = data.cloudflare_zone.ente.zone_id
  content = aws_lightsail_instance.museum.public_ip_address
  proxied = true
}
```

`AAAA` records were written as well. What attribute contains the Lightsail instance's public IPv6 address was not clear by reading [the provider's documentation](https://registry.terraform.io/providers/hashicorp/aws/6.14.0/docs/resources/lightsail_instance#attribute-reference "aws_lightsail_instance | Resources | hashicorp/aws | Terraform | Terraform Registry"), but using `ipv6_addresses` worked.

```terraform
# AAAA record for Ente albums
resource "cloudflare_dns_record" "albums_aaaa" {
  name    = "ente-albums"
  ttl     = 1
  type    = "AAAA"
  zone_id = data.cloudflare_zone.ente.zone_id
  content = aws_lightsail_instance.museum.ipv6_addresses[0]
  proxied = true
}
```

<h2 id="ente-declarative-nixos">Writing NixOS configurations</h2>

I started by copying existing configuration from the Vault instance.

<h3 id="ente-declarative-nixos-disko">Modifying the disko configuration</h3>

The disko configuration was more or less the same, except the additional storage disk [that I had to accommodate](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/museum/disko-config.nix#L89-L116 "nixos-config/hosts/museum/disko-config.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config"). I enabled compression (like I did to other partitions) to save space.

```nix
{
  disko.devices = {
    disk = {

      # Primary disk...

      # Secondary storage for persistent data
      storage = {
        type = "disk";
        device = "/dev/nvme1n1";

        # GPT (partition table) as the disk's content
        content = {
          type = "gpt";

          # List of partitions
          partitions = {
            # PostgreSQL storage
            postgresql = {
              size = "100%";

              content = {
                type = "btrfs";

                mountpoint = "/var/lib/postgresql";
                mountOptions = [
                  "defaults"
                  "compress=zstd"
                ];
              };
            };
          };
        };
      };
    };

    # tmpfs root follows...
  };
}
```

<h3 id="ente-declarative-nixos-vault">Accessing secrets with Vault</h3>

Using Vault within NixOS required me to set up a service that would fetch secrets. I could see a few choices, but I went for [one from Determinate Systems](https://github.com/DeterminateSystems/nixos-vault-service "DeterminateSystems/nixos-vault-service"). It runs [Vault Agent](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent "What is Vault Agent? | Vault | HashiCorp Developer") under the hood, and injects secrets into existing systemd services. Since the NixOS module that implements Ente uses one for hosting, it was a perfect way to integrate Vault in my case.

At `flake.nix`, I added an input for the Vault service.

```nix
{
  # Flake description...

  inputs = {
    # Preceding inputs...

    # NixOS Vault service
    nixos-vault-service = {
      url = "github:DeterminateSystems/nixos-vault-service";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  # Flake outputs follow...
}
```

I then wrote [a module](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/museum/vault-agent.nix "nixos-config/hosts/museum/vault-agent.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") `vault-agent.nix`, importing the module for Vault Agent as a start.

```nix
imports = [ inputs.nixos-vault-service.nixosModules.nixos-vault-service ];
```

Vault is licensed under [Business Source License](https://www.hashicorp.com/bsl "Business Source License 1.1"), making it source-available but non-free. [Hydra](https://github.com/NixOS/hydra "NixOS/hydra: Hydra, the Nix-based continuous build system") does not build unfree software, so I had to build it manually. It also meant that I had to specifically allow it to be used at all.

```nix
# Allow unfree package (Vault)
nixpkgs.config.allowUnfreePredicate =
  pkg:
  builtins.elem (lib.getName pkg) [
    "vault"
    "vault-bin"
  ];
```

During the testing, building Vault took more than how long I could tolerate and sometimes even failed. [A binary version](https://github.com/NixOS/nixpkgs/blob/8af13b1251ff261d26ec9c22cb6dd2c2944b5364/pkgs/by-name/va/vault-bin/package.nix#L60 "nixpkgs/pkgs/by-name/va/vault-bin/package.nix at 8af13b1251ff261d26ec9c22cb6dd2c2944b5364 · NixOS/nixpkgs") fortunately exists, but the module by Determinate Systems [was hardcoded](https://github.com/DeterminateSystems/nixos-vault-service/blob/4a25afe87cd725e758e1e3a2c24028885fa21d39/module/implementation.nix#L69 "nixos-vault-service/module/implementation.nix at 4a25afe87cd725e758e1e3a2c24028885fa21d39 · DeterminateSystems/nixos-vault-service") to use the source version. To circumvent this problem, I added an overlay to Nixpkgs, essentially telling anyone who uses `vault` (the source version) to use `vault-bin` (the binary version) instead.

```nix
# Use binary version of Vault to avoid building the package
nixpkgs.overlays = [
  (final: prev: { vault = prev.vault-bin; })
];
```

(`vault-bin` technically still *builds* but does so by [fetching binary](https://github.com/NixOS/nixpkgs/blob/8af13b1251ff261d26ec9c22cb6dd2c2944b5364/pkgs/by-name/va/vault-bin/package.nix#L11-L34 "nixpkgs/pkgs/by-name/va/vault-bin/package.nix at 8af13b1251ff261d26ec9c22cb6dd2c2944b5364 · NixOS/nixpkgs") from HashiCorp)

Configuration for Vault Agent was added next. I referred to [available options](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent#configuration-file-options "What is Vault Agent? | Vault | HashiCorp Developer") to set up the Vault address and [auto-authentication](https://developer.hashicorp.com/vault/docs/agent-and-proxy/autoauth "What is Auto-authentication? | Vault | HashiCorp Developer"), where I used AppRole method in this case.

```nix
detsys.vaultAgent = {
  defaultAgentConfig = {
    # Preceding Vault configuration...
    auto_auth.method = [
      {
        type = "approle";
        config = {
          remove_secret_id_file_after_reading = false;
          role_id_file_path = "/var/lib/secrets/vault-role-id";
          secret_id_file_path = "/var/lib/secrets/vault-secret-id";
        };
      }
    ];
  };

  # systemd services follow...
};
```

Properties `role_id_file_path` and `secret_id_file_path` refer to files that [Terraform will provide](#ente-declarative-terraform-aws) during deployment.

I provided templates, which contains nothing else but the secret itself, for the agent to use while writing files with fetched secrets. I noticed that Nginx also wants an SSL certificate for hosts that it will serve, so it was taken into consideration.

```nix
detsys.vaultAgent = {
  # Vault Agent config...

  systemd.services = {
    ente = {
      # Enable Vault integration with Museum
      enable = true;

      secretFiles = {
        # Restart the service in case secrets change
        defaultChangeAction = "restart";

        # Get secrets from Vault
        files = {
          s3_key.template = ''
            {{ with secret "kv/ente/b2/ente-b2" }}{{ .Data.key }}{{ end }}
          '';
          s3_secret.template = ''
            {{ with secret "kv/ente/b2/ente-b2" }}{{ .Data.secret }}{{ end }}
          '';
          s3_endpoint.template = ''
            {{ with secret "kv/ente/b2/ente-b2" }}{{ .Data.endpoint }}{{ end }}
          '';
          s3_region.template = ''
            {{ with secret "kv/ente/b2/ente-b2" }}{{ .Data.region }}{{ end }}
          '';
          s3_bucket.template = ''
            {{ with secret "kv/ente/b2/ente-b2" }}{{ .Data.bucket }}{{ end }}
          '';

          key_encryption.template = ''
            {{ with secret "kv/ente/aws/museum" }}{{ .Data.key.encryption }}{{ end }}
          '';
          key_hash.template = ''
            {{ with secret "kv/ente/aws/museum" }}{{ .Data.key.hash }}{{ end }}
          '';
          jwt_secret.template = ''
            {{ with secret "kv/ente/aws/museum" }}{{ .Data.jwt.secret }}{{ end }}
          '';

          "tls.cert".template = ''
            {{ with secret "kv/ente/cloudflare/certificate" }}{{ .Data.certificate }}{{ end }}
          '';
          "tls.key".template = ''
            {{ with secret "kv/ente/cloudflare/certificate" }}{{ .Data.certificate_key }}{{ end }}
          '';
        };
      };
    };

    nginx = {
      # Enable Vault integration with Nginx
      enable = true;

      secretFiles = {
        # Reload unit in case secrets change
        defaultChangeAction = "reload";

        # Get secrets from Vault
        files = {
          certificate.template = ''
            {{ with secret "kv/ente/cloudflare/certificate" }}{{ .Data.certificate }}{{ end }}
          '';
          certificate-key.template = ''
            {{ with secret "kv/ente/cloudflare/certificate" }}{{ .Data.certificate_key }}{{ end }}
          '';
        };
      };
    };
  };
};
```

Lastly, with Vault behind the tailnet, I made sure the affected services start after the instance can connect to the secret storage.

```nix
# Start affected services after tailscaled starts since it is needed to connect to Vault
systemd.services =
  lib.genAttrs
    [
      "ente"
      "nginx"
    ]
    (name: {
      after = [ config.systemd.services.tailscaled-autoconnect.name ];
    });
```

<h3 id="ente-declarative-nixos-tailscale">Connecting to the tailnet</h3>

The [Museum-specific Tailscale configuration](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/hosts/museum/tailscale.nix "nixos-config/hosts/museum/tailscale.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") remained unchanged, except for three things: the hostname, the advertised tags, and the path to the OAuth client secret.

```nix
{
  # Apply host-specific Tailscale configurations
  services.tailscale = {
    # Provide auth key to issue `tailscale up` with
    authKeyFile = "/var/lib/secrets/tailscale-oauth-client-secret";

    # Enable Tailscale SSH and advertise tags
    extraUpFlags = [
      "--advertise-tags=tag:museum,tag:webserver"
      "--hostname=museum"
      "--ssh"
    ];

    # Use routing features for servers
    useRoutingFeatures = "server";
  };
}
```

<h3 id="ente-declarative-nixos-museum">Configuring Ente</h3>

I roughly followed [a quickstart guide](https://github.com/NixOS/nixpkgs/blob/84d7ec6875a450471b5daa93254183ac4651e922/nixos/modules/services/web-apps/ente.md "nixpkgs/nixos/modules/services/web-apps/ente.md at 84d7ec6875a450471b5daa93254183ac4651e922 · NixOS/nixpkgs") with a few changes:

- Secrets are from Vault Agent
- `credentials-dir`, where Ente will read the SSL certificate from, was set to the parent directory of service-specific secrets
- `s3.b2-eu-cen.are_local_buckets` was set to `false` because it obviously is not

```nix
services.ente = {
  # Preceding configurations...

  api = {
    # Preceding API configurations...

    settings =
      let
        files = config.detsys.vaultAgent.systemd.services.ente.secretFiles.files;
      in
      {
        # Get credentials from where the certificate is
        credentials-dir = builtins.dirOf files."tls.cert".path;

        # Manage object storage settings
        s3 = {
          b2-eu-cen = {
            # Indicate that this is not a local MinIO bucket
            are_local_buckets = false;

            # Set sensitive values
            key._secret = files.s3_key.path;
            secret._secret = files.s3_secret.path;
            endpoint._secret = files.s3_endpoint.path;
            region._secret = files.s3_region.path;
            bucket._secret = files.s3_bucket.path;
          };
        };

        # Manage key-related settings
        key = {
          encryption._secret = files.key_encryption.path;
          hash._secret = files.key_hash.path;
        };
        jwt.secret._secret = files.jwt_secret.path;

        # Internal settings...
      };
  };
};
```

The SSL certificate was then once again provided to Nginx.

```nix
services.nginx = {
  # Use recommended proxy settings
  recommendedProxySettings = true;

  virtualHosts =
    let
      domains = config.services.ente.web.domains;
      secrets = config.detsys.vaultAgent.systemd.services.nginx.secretFiles.files;
    in
    lib.genAttrs
      [
        domains.api
        domains.accounts
        domains.cast
        domains.albums
        domains.photos
      ]
      (name: {
        # Add certificates to supported endpoints
        sslCertificate = secrets.certificate.path;
        sslCertificateKey = secrets.certificate-key.path;
      });
};
```

<h2 id="ente-declarative-github">The GitHub Actions workflow</h2>

Repository secrets were first added to the [new repository](https://github.com/lyuk98/terraform-ente "lyuk98/terraform-ente: Terraform configurations (Ente)").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/7a9c3304-0122-44b1-8e4c-77b2a7ca18cf.avif">
  <img src="https://images.lyuk98.com/7fad6618-b378-4912-95f4-2ecf4989508c.avif" alt="An interface for managing repository secrets that have been populated" title="The repository secrets">
</picture>

- `AWS_REGION` is used to specify where to deploy Lightsail resources to.
- `AWS_ROLE_TO_ASSUME` is an ARN of the role that the workflow will assume during deployment.
- `CLOUDFLARE_API_TOKEN` is, obviously, a token for accessing Cloudflare API.
- `CLOUDFLARE_ZONE_ID` is an ID representing my [zone](https://developers.cloudflare.com/fundamentals/concepts/accounts-and-zones/#zones "Accounts, zones, and profiles · Cloudflare Fundamentals docs") (or domain).
- `TF_TAILSCALE_OAUTH_CLIENT_ID` and `TF_TAILSCALE_OAUTH_CLIENT_SECRET` are used by Terraform to create resources at Tailscale.
- `TS_OAUTH_CLIENT_ID` and `TS_OAUTH_SECRET`, unlike the abovementioned OAuth client, are only used to establish connection to the tailnet, and thus Vault.
- `VAULT_ADDR` sets where to look for Vault.

A workflow file for GitHub Actions [was created](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente"). Almost everything was copied over from the bootstrapping stage, except for a few things.

First, more environment variables were declared using the new secrets.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    name: Deploy Terraform configuration
    permissions:
      id-token: write
    env:
      VAULT_ADDR: ${{ secrets.VAULT_ADDR }}

      TAILSCALE_OAUTH_CLIENT_ID: ${{ secrets.TF_TAILSCALE_OAUTH_CLIENT_ID }}
      TAILSCALE_OAUTH_CLIENT_SECRET: ${{ secrets.TF_TAILSCALE_OAUTH_CLIENT_SECRET }}
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}

      TF_VAR_aws_region: ${{ secrets.AWS_REGION }}
      TF_VAR_cloudflare_zone_id: ${{ secrets.CLOUDFLARE_ZONE_ID }}

    steps:
      # Deployment steps...
```

Authentication to [AWS](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml#L32-L39 "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") and subsequently [Vault](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml#L50-L59 "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") remains the same, but [a step was added](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml#L41-L42 "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") to install Nix to the environment. It was needed for installing NixOS using nixos-anywhere.

```yaml
- name: Install Nix
  uses: cachix/install-nix-action@v31
```

Application keys for Backblaze B2 are then [fetched from Vault](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml#L61-L86 "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente"). Except for the location of the secrets, what happens remains the same with [the bootstrapping workflow](#bootstrap-declarative-github).

```yaml
- name: Get secrets from Vault
  id: vault-secrets
  run: |
    b2_application_key_id=$(vault kv get -field=b2_application_key_id -mount=kv ente/bootstrap)
    b2_application_key=$(vault kv get -field=b2_application_key -mount=kv ente/bootstrap)

    echo "::add-mask::$b2_application_key_id"
    echo "::add-mask::$b2_application_key"

    b2_state_application_key_id=$(vault kv get -field=b2_state_application_key_id -mount=kv ente/bootstrap)
    b2_state_application_key=$(vault kv get -field=b2_state_application_key -mount=kv ente/bootstrap)
    b2_state_bucket=$(vault kv get -field=b2_bucket -mount=kv ente/b2/tfstate-bootstrap)
    b2_state_endpoint=$(vault kv get -field=b2_endpoint -mount=kv ente/b2/tfstate-bootstrap)

    echo "::add-mask::$b2_state_application_key_id"
    echo "::add-mask::$b2_state_application_key"
    echo "::add-mask::$b2_state_bucket"
    echo "::add-mask::$b2_state_endpoint"

    echo "B2_APPLICATION_KEY_ID=$b2_application_key_id" >> $GITHUB_ENV
    echo "B2_APPLICATION_KEY=$b2_application_key" >> $GITHUB_ENV

    echo "b2-state-application-key-id=$b2_state_application_key_id" >> $GITHUB_OUTPUT
    echo "b2-state-application-key=$b2_state_application_key" >> $GITHUB_OUTPUT
    echo "b2-state-bucket=$b2_state_bucket" >> $GITHUB_OUTPUT
    echo "b2-state-endpoint=$b2_state_endpoint" >> $GITHUB_OUTPUT
```

The remainder of [the workflow definition](https://github.com/lyuk98/terraform-ente/blob/edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc/.github/workflows/deploy.yml "terraform-ente/.github/workflows/deploy.yml at edaeb0e403b1e9b2b5b2e31d10caa2527e5acbbc · lyuk98/terraform-ente") are the same, but what it will actually do greatly differ with the changes in Terraform configuration.

<h2 id="ente-declarative-deployment">The manual deployment</h2>

While I could let GitHub Actions set everything up, I did not want it to do so for tens of minutes. I instead let my personal computer, which probably has more resources allocated than [how much the remote runner does](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job#standard-github-hosted-runners-for-public-repositories "Choosing the runner for a job - GitHub Docs"), do exactly that.

Environment variables were first set, just like [how it is done automatically](#ente-declarative-github).

```
[lyuk98@framework:~/terraform-ente]$ export CLOUDFLARE_API_TOKEN=<API token>
[lyuk98@framework:~/terraform-ente]$ # Other environment variables...
```

Some tools were then made available to the shell environment. For some reason, nixos-anywhere expects `jq` to be present, so it was also added.

```
[lyuk98@framework:~/terraform-ente]$ nix shell nixpkgs#awscli2 nixpkgs#jq
```

Access to AWS was granted differently, since I cannot do so using OIDC.

```
[lyuk98@framework:~/terraform-ente]$ aws sso login
```

I already [had access to Vault](#vault-postinstall) prior to this, so another authentication was not necessary.

The step for getting secrets was mostly copied over.

```
[lyuk98@framework:~/terraform-ente]$ b2_application_key_id=$(vault kv get -field=application_key_id -mount=kv ente/b2/terraform-b2-ente)
[lyuk98@framework:~/terraform-ente]$ b2_application_key=$(vault kv get -field=application_key -mount=kv ente/b2/terraform-b2-ente)
[lyuk98@framework:~/terraform-ente]$ b2_state_application_key_id=$(vault kv get -field=application_key_id -mount=kv ente/b2/tfstate-ente)
[lyuk98@framework:~/terraform-ente]$ b2_state_application_key=$(vault kv get -field=application_key -mount=kv ente/b2/tfstate-ente)
[lyuk98@framework:~/terraform-ente]$ b2_state_bucket=$(vault kv get -field=b2_bucket -mount=kv ente/b2/tfstate-bootstrap)
[lyuk98@framework:~/terraform-ente]$ b2_state_endpoint=$(vault kv get -field=b2_endpoint -mount=kv ente/b2/tfstate-bootstrap)
[lyuk98@framework:~/terraform-ente]$ export B2_APPLICATION_KEY_ID=$b2_application_key_id
[lyuk98@framework:~/terraform-ente]$ export B2_APPLICATION_KEY=$b2_application_key
```

Using the credentials from the earlier step, `terraform init` was performed.

```
[lyuk98@framework:~/terraform-ente]$ terraform init \
  -backend-config="bucket=$b2_state_bucket" \
  -backend-config="endpoints={s3=\"$b2_state_endpoint\"}" \
  -backend-config="access_key=$b2_state_application_key_id" \
  -backend-config="secret_key=$b2_state_application_key" \
  -input=false
```

Subsequently, the changes in infrastructure were planned and then applied.

```
[lyuk98@framework:~/terraform-ente]$ terraform plan -input=false -out=tfplan
[lyuk98@framework:~/terraform-ente]$ terraform apply -input=false tfplan
```

Installing NixOS took a while, which output was hidden due to sensitive values, but it eventually finished after a little less than ten minutes.

<h1 id="postinstall">Post-installation configurations</h1>

After a few months I have started working on this, Ente was finally running. I visited the photos page (with the `ente-photos` subdomain) and created an account.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/444bb606-03bb-43ec-9316-b59b1f818ae3.avif">
  <img src="https://images.lyuk98.com/d14b5a8a-4545-4323-a7a1-73227830e933.avif" alt="The signup page for Ente Photos" title="The signup page">
</picture>

The email service was not set up, so no verification code was sent to my email address. Like how [the guide](https://help.ente.io/self-hosting/installation/post-install/#step-1-creating-first-user "Post-installation steps - Self-hosting | Ente Help") pointed out, I searched the journal to view the code.

```
[root@museum:~]# journalctl --unit ente.service
```

The registration was complete afterwards, and photos were now ready to be backed up.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/f79c3443-6d1a-4c07-a79b-c277d75f09ae.avif">
  <img src="https://images.lyuk98.com/34813000-1d00-4823-930d-64d4669b287b.avif" alt="The photos page for Ente Photos. There is no photo and the interface suggests uploading the first one." title="The photos page">
</picture>

However, my account was under a free plan, which had a quota of 10 gigabytes. It was certainly not enough for me, but I could increase the limit to 100 terabytes using [Ente CLI](https://help.ente.io/self-hosting/administration/cli "Ente CLI for Self-hosted Instance - Self-hosting | Ente Help").

To manage users' subscription, I had to become an administrator first. The user ID was needed, and while [the official guide](https://help.ente.io/self-hosting/administration/users#whitelist-admins "User Management - Self-hosting | Ente Help") looks through the database to figure it out, I found it easier to upload a few photos and have a look at the object storage bucket.

```
[lyuk98@framework:~]$ backblaze-b2 ls b2://<Ente bucket>/
```

The only entry from the result was my user ID. It was added to my NixOS configuration.

```nix
services.ente.api.settings.internal.admin = <user ID>;
```

The change in configuration was applied afterwards.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild switch --target-host root@museum --flake .#museum
```

It was now time to work with the CLI. I wrote `~/.ente/config.yaml` to specify a custom API endpoint and added my account.

```
[lyuk98@framework:~]$ nix shell nixpkgs#ente-cli
[lyuk98@framework:~]$ mkdir --parents ~/.ente
[lyuk98@framework:~]$ cat > ~/.ente/config.yaml
endpoint:
  api: https://ente-api.<domain>
[lyuk98@framework:~]$ ente account add
```

My account was then upgraded using `ente admin update-subscription`.

```
[lyuk98@framework:~]$ ente admin update-subscription \
  --admin-user <email> \
  --user <email> \
  --no-limit True
```

<h1 id="finishing-up">Finishing up</h1>

It was a long journey for me to reach this far. I have just started using NixOS when I started working on this; it took a while to start thinking declaratively, but I feel better knowing a bit more about it by experiencing declarative resource management.

Improvements are, in my opinion, still necessary. Being a Terraform first-timer, there may be some best practices I missed out on. I also feel like many steps were still done manually, which I wish to lessen next time.

Lastly, there are a few concerns that I was aware of, but decided not to act upon:

<h2 id="finishing-up-opentofu">A note on OpenTofu (and OpenBao)</h2>

[Earlier in this post](#ente-declarative-nixos-vault), I briefly touched on HashiCorp's switch to [Business Source License](https://www.hashicorp.com/bsl "Business Source License 1.1") that affects Terraform and Vault. While I do not consider myself to be affected by the change, since I am not working on a commercial product that will ["compete with"](https://github.com/hashicorp/terraform/blob/a90ba0bfe4aa8a189fdd571e3d964134eacf591e/LICENSE#L9-L13 "terraform/LICENSE at a90ba0bfe4aa8a189fdd571e3d964134eacf591e · hashicorp/terraform") the company, it still came to me as an uncertainty to the future of the infrastructure-as-code tool.

[OpenTofu](https://opentofu.org/ "OpenTofu"), which started as a fork of Terraform, naturally came to my attention. I did not write the code with interoperability in mind, but I still tried it out on my project, repeating [the manual deployment process](#ente-declarative-deployment) earlier.

```
[lyuk98@framework:~/terraform-ente]$ nix shell nixpkgs#opentofu nixpkgs#jq
[lyuk98@framework:~/terraform-ente]$ # Populate variables...
[lyuk98@framework:~/terraform-ente]$ tofu init \
  -backend-config="bucket=$b2_state_bucket" \
  -backend-config="endpoints={s3=\"$b2_state_endpoint\"}" \
  -backend-config="access_key=$b2_state_application_key_id" \
  -backend-config="secret_key=$b2_state_application_key" \
  -input=false \
  -reconfigure
[lyuk98@framework:~/terraform-ente]$ tofu plan -input=false -out=tfplan
[lyuk98@framework:~/terraform-ente]$ tofu apply -input=false tfplan
```

To my surprise, everything worked as expected. If it does not result in compromising something important (like security), keeping interoperability, while not a priority, could still be on my scope. However, if I were to use the open-source counterpart, I would definitely consider trying out [state file encryption](https://opentofu.org/docs/v1.9/language/state/encryption/ "State and Plan Encryption | OpenTofu").

Unlike OpenTofu, though, I have not given much attention to [OpenBao](https://openbao.org/ "OpenBao"), a fork of Vault, because it apparently lacks AWS authentication method that I rely on.

<h2 id="finishing-up-updates">Keeping instances up to date</h2>

With Vault and Museum, there are now [three NixOS hosts](https://github.com/lyuk98/nixos-config/blob/4ddfa37239908404f776d502b2b4436b430b07f4/flake.nix#L131-L150 "nixos-config/flake.nix at 4ddfa37239908404f776d502b2b4436b430b07f4 · lyuk98/nixos-config") that I manage declaratively. All of them need to stay up to date, and though I can manually do it for now, I thought it would be a natural thing to consider automating such a process. A project I came across, [comin](https://github.com/nlewo/comin "nlewo/comin: GitOps For NixOS Machines"), apparently achieves the goal, but I could not find a way to offload configuration builds to a faster machine during the limited time I took to learn about the project.

However, an interesting [feature](https://github.com/nlewo/comin/tree/b8ab3eccbc3acf171b9ea377e88b8194f81c579d#features "nlewo/comin at b8ab3eccbc3acf171b9ea377e88b8194f81c579d") it mentioned was "Prometheus metrics". I had no practical experience with it, but integrating it seemed like a nice idea on paper. If I am motivated enough, I may be working on it in the future.
