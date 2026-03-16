---
date: '2025-12-01'
title: Building a project (#3) - Attempting to run Kubernetes and OpenStack
---

This is a part of series *Building a project*, where I try to make something using [OpenStack](https://www.openstack.org/ "Open Source Cloud Computing Infrastructure - OpenStack").

1. [Learning about OpenStack](https://lyuk98.com/60219/building-a-project-1-learning-about-openstack "Building a project (#1) - Learning about OpenStack")
2. [Preparing a server](https://lyuk98.com/66893/building-a-project-2-preparing-a-server "Building a project (#2) - Preparing a server")
3. Attempting to run Kubernetes and OpenStack

---

With my server ready, I was excited to get Kubernetes running. Following a guide I came across online, I added a few lines of configuration and applied it right away.

```nix
services.kubernetes = {
  roles = [
    "master"
    "node"
  ];
};
```

However, it was not going to be that easy.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild switch --target-host root@xps13 --flake .#xps13
building the system configuration...
error:
       … while calling the 'head' builtin
         at /nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/lib/attrsets.nix:1696:13:
         1695|           if length values == 1 || pred here (elemAt values 1) (head values) then
         1696|             head values
             |             ^
         1697|           else

       … while evaluating the attribute 'value'
         at /nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/lib/modules.nix:1118:7:
         1117|     // {
         1118|       value = addErrorContext "while evaluating the option `${showOption loc}':" value;
             |       ^
         1119|       inherit (res.defsFinal') highestPrio;

       … while evaluating the option `system.build.toplevel':

       … while evaluating definitions from `/nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/nixos/modules/system/activation/top-level.nix':

       … while evaluating the option `warnings':

       … while evaluating definitions from `/nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/nixos/modules/system/boot/systemd.nix':

       … while evaluating the option `systemd.services.certmgr.serviceConfig':

       … while evaluating definitions from `/nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/nixos/modules/system/boot/systemd.nix':

       … while evaluating the option `systemd.services.certmgr.preStart':

       … while evaluating definitions from `/nix/store/1wnxdqr2n1pj80lirh9pzsymslx8zd9l-source/nixos/modules/services/security/certmgr.nix':

       … while evaluating the option `services.kubernetes.masterAddress':

       (stack trace truncated; use '--show-trace' to show the full, detailed trace)

       error: The option `services.kubernetes.masterAddress' was accessed but has no value defined. Try setting the option.
Command 'nix --extra-experimental-features 'nix-command flakes' build --print-out-paths '.#nixosConfigurations."xps13".config.system.build.toplevel' --no-link' returned non-zero exit status 1.
```

I decided to come back after learning more about it.

---

# The initial setup

This was the code I wrote to get Kubernetes running at all:

```nix
{ config, ... }:
let
  cfg = config.services.kubernetes;
in
{
  services.kubernetes = {
    # Act as both a control plane and a node
    roles = [
      "master"
      "node"
    ];

    # Set CIDR range for pods
    clusterCidr = "fd66:77d6:a28f:8da1::/64";

    # Set address of control plane to hostname (reachable via Tailscale)
    masterAddress = config.networking.hostName;

    # Use the API server's (human-readable) address as if its IP address was not set
    apiserverAddress = "https://${cfg.masterAddress}:${builtins.toString cfg.apiserver.securePort}";

    apiserver = {
      # Advertise device's IP address from Tailscale
      advertiseAddress = "fd7a:115c:a1e0::3137:c03c";

      # Use a custom CIDR range for clusters
      serviceClusterIpRange = "fda3:9e79:d157:f04a::/64";
    };

    pki = {
      certs = {
        # Add IP address of API server for the certificate
        # Nixpkgs does so, but incorrectly by assuming an IPv4 address
        apiServer.hosts = [ "fda3:9e79:d157:f04a::1" ];
      };
    };
  };

  # Use OverlayFS snapshotter plugin to avoid using ZFS's
  virtualisation.containerd.settings = {
    plugins."io.containerd.grpc.v1.cri" = {
      containerd.snapshotter = "overlayfs";
    };
  };
}
```

`services.kubernetes.clusterCidr`, which I thought could be useful together with Tailscale's [subnet router](https://tailscale.com/kb/1019/subnets "Subnet routers · Tailscale Docs"), and `services.kubernetes.pki.certs.apiServer.hosts` could be omitted without affecting the service's ability to run at all. Others did, however, in some ways I did not fully understand.

Because the device can be accessed with its hostname, thanks to [MagicDNS](https://tailscale.com/kb/1081/magicdns "MagicDNS · Tailscale Docs"), it became the control plane's address (`services.kubernetes.masterAddress`). I still had to specify an IP address, but the one from Tailscale (`services.kubernetes.apiserver.advertiseAddress`) was used for this purpose.

Another random range (aside from `services.kubernetes.clusterCidr`) of unique local addresses in CIDR notation (`services.kubernetes.apiserver.serviceClusterIpRange`) was also specified to solve an issue I did not clearly understand.

I had to separately create a separate ZFS dataset if I wanted to make use of containerd's ZFS snapshotter, which I ended up disabling by setting it (`virtualisation.containerd.settings.plugins."io.containerd.grpc.v1.cri".containerd.snapshotter`) back to `overlayfs`.

After applying the configuration with `nixos-rebuild`:

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild test --target-host root@xps13 --flake .#xps13
```

I could see that the control plane was in operation:

```
[lyuk98@xps13:~]$ nix shell nixpkgs#kubectl
[lyuk98@xps13:~]$ sudo kubectl cluster-info --kubeconfig /etc/kubernetes/cluster-admin.kubeconfig
Kubernetes control plane is running at https://xps13:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

After making sure that it works, I scrapped everything I wrote about Kubernetes and switched to [K3s](https://k3s.io/ "K3s").

# Switching to K3s

The reason for the switch was that while I could get Kubernetes running, [options for further declarative configurations with NixOS modules](https://nixos.org/manual/nixos/stable/#sec-kubernetes "NixOS Manual") seemed to be limited. I considered something as error-prone and absurd as writing systemd units myself, but it felt like too much effort for little to no gain.

On the other hand, [NixOS module for K3s](https://github.com/NixOS/nixpkgs/blob/85a6c4a07faa12aaccd81b36ba9bfc2bec974fa1/pkgs/applications/networking/cluster/k3s/README.md "nixpkgs/pkgs/applications/networking/cluster/k3s/README.md at 85a6c4a07faa12aaccd81b36ba9bfc2bec974fa1 · NixOS/nixpkgs") was more configurable. Another added benefit was that it was "optimized for ARM" devices like my Raspberry Pi, which is still [running PeerTube](https://lyuk98.com/60610/self-hosting-peertube-with-tailscale "Self-hosting PeerTube with Tailscale"); I could (one day) use both devices to run whatever I will come up with by the end of this project.

## Preparing Tailscale for the integration

K3s supports [integration with Tailscale](https://docs.k3s.io/networking/distributed-multicloud#integration-with-the-tailscale-vpn-provider-experimental "Distributed hybrid or multicloud cluster | K3s"), albeit in an experimental state at the time of writing. My K3s cluster currently only consists of one device, but if I were to add another one, it would be possible to add those that are not in the same private network.

Before setting up K3s for the integration, I went to [Tailscale admin console](https://login.tailscale.com/admin "Machines - Tailscale") to prepare for it. The first thing I did was to [add tags](https://login.tailscale.com/admin/acls/visual/tags/add "Create tag - Tailscale"):

```jsonc
{
    // Define the tags which can be applied to devices and by which users.
    "tagOwners": {
        "tag:caddy":      ["autogroup:admin"],
        "tag:ci":         ["autogroup:admin"],
        "tag:museum":     ["autogroup:admin"],
        "tag:peertube":   ["autogroup:admin"],
        "tag:vault":      ["autogroup:admin"],
        "tag:webserver":  ["autogroup:admin"],
        "tag:k3s":        ["tag:k3s-server"],
        "tag:k3s-server": ["autogroup:admin"],
    },
}
```

Tags `tag:k3s-server` and `tag:k3s` were created. The former was for server nodes, while the latter was for both server and agent nodes.

To access pods, I, as well as pods themselves, could use [subnet routers](https://tailscale.com/kb/1019/subnets "Subnet routers · Tailscale Docs"). Because Tailscale operates under dual-stack mode, I used both IPv4 and IPv6 CIDRs for the cluster, which I set to `10.42.0.0/16` and `2001:cafe:42::/56`, respectively. They were later to be passed to `--cluster-cidr` option for [the K3s CLI](https://docs.k3s.io/cli/server "server | K3s").

Those routes were [set to be automatically approved](https://login.tailscale.com/admin/acls/visual/auto-approvers-routes/add "Add route - Tailscale") when nodes (with tag `tag:k3s`) advertise them.

```jsonc
{
    "autoApprovers": {
        "routes": {
            "10.42.0.0/16":      ["tag:k3s"],
            "2001:cafe:42::/56": ["tag:k3s"],
        },
    },
}
```

They were also [given friendly names](https://login.tailscale.com/admin/acls/visual/hosts/add "Create host - Tailscale") via hosts, which was totally due to my personal preference.

```jsonc
{
    "hosts": {
        // IPv4 network CIDR to use for pod IPs
        "k3s-cluster-ipv4": "10.42.0.0/16",

        // IPv6 network CIDR to use for pod IPs
        "k3s-cluster-ipv6": "2001:cafe:42::/56",
    },
}
```

The access rules [were then added](https://login.tailscale.com/admin/acls/visual/general-access-rules/add "Add rule - Tailscale"), based on [inbound rules outlined by K3s](https://docs.k3s.io/installation/requirements#inbound-rules-for-k3s-nodes "Requirements | K3s").

```jsonc
{
    "grants": [
        // Required only for HA with embedded etcd
        {
            "src": ["tag:k3s-server"],
            "dst": ["tag:k3s-server"],
            "ip":  ["tcp:2379", "tcp:2380"],
        },
        // K3s supervisor and Kubernetes API Server
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s-server"],
            "ip":  ["tcp:6443"],
        },
        // Required only for Flannel VXLAN
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["udp:8472"],
        },
        // Kubelet metrics
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["tcp:10250"],
        },
        // Required only for Flannel Wireguard with IPv4
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["udp:51820"],
        },
        // Required only for Flannel Wireguard with IPv6
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["udp:51821"],
        },
        // Required only for embedded distributed registry (Spegel)
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["tcp:5001"],
        },
        // Required only for embedded distributed registry (Spegel)
        {
            "src": ["tag:k3s"],
            "dst": ["tag:k3s"],
            "ip":  ["tcp:6443"],
        },
        {
            "src": ["tag:k3s", "host:k3s-cluster-ipv4"],
            "dst": ["host:k3s-cluster-ipv4"],
            "ip":  ["*"],
        },
        {
            "src": ["tag:k3s", "host:k3s-cluster-ipv6"],
            "dst": ["host:k3s-cluster-ipv6"],
            "ip":  ["*"],
        },
    ],
}
```

I doubted their usefulness aside from the last two, but I nevertheless left them there for now and decided to come back when there are problems with such configuration.

The last thing to do was to let the server actually use the newly-created tags. An OAuth credential with `auth_keys` access to them was created to replace the previous one.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/65f494ff-4d5c-4750-ae3c-e66f23b342ed.avif">
  <img src="https://images.lyuk98.com/eeab7cb9-c956-47d4-88ec-cb825b0c7523.avif" alt="A dialog for adding a new credential at Tailscale. Checkbox &quot;Write&quot; is checked for the &quot;Auth Keys&quot; scope, and tags named &quot;tag:webserver&quot;, &quot;tag:k3s-server&quot;, and &quot;tag:k3s&quot; are specified." title="Adding a new credential">
</picture>

The secret `tailscale-auth-key` was then edited to apply the new credential:

```
[lyuk98@framework:~/nixos-config]$ sops edit hosts/xps13/secrets.yaml
```

## Enabling K3s within NixOS

First, I edited [the Tailscale configuration](https://github.com/lyuk98/nixos-config/blob/bd3b8b00da1a2691b9452f5aca3e513d241f4eb9/hosts/xps13/tailscale.nix "nixos-config/hosts/xps13/tailscale.nix at bd3b8b00da1a2691b9452f5aca3e513d241f4eb9 · lyuk98/nixos-config") for the device to advertise the new tags and advertise/accept the new subnet. The option `services.tailscale.authKeyFile` was also removed; the OAuth credential will later be used in another way.

```nix
{ config, ... }:
{
  # Get auth key via sops-nix
  sops.secrets.tailscale-auth-key = {
    sopsFile = ./secrets.yaml;
  };

  # Apply host-specific Tailscale configurations
  services.tailscale = {
    authKeyParameters = {
      # Register as an ephemeral node
      ephemeral = true;
    };

    # Enable Tailscale SSH and advertise tags and subnet routes
    extraUpFlags =
      let
        k3s = config.services.k3s;
        tags = [
          "webserver"
          "k3s-server"
          "k3s"
        ];
      in
      [
        "--accept-routes"
        "--advertise-routes=${builtins.concatStringsSep "," k3s.clusterCidr}"
        "--advertise-tags=${builtins.concatStringsSep "," (builtins.map (tag: "tag:${tag}") tags)}"
        "--ssh"
      ];

    # Use routing features for both clients and servers
    useRoutingFeatures = "both";
  };
}
```

The `services.tailscale.extraUpFlags` option now contains the following values:

```
nix-repl> :lf github:lyuk98/nixos-config/01900f9482f7d5b09badc82fd8792f67ceb9a829
nix-repl> outputs.nixosConfigurations.xps13.config.services.tailscale.extraUpFlags
[
  "--accept-routes"
  "--advertise-routes=10.42.0.0/16,2001:cafe:42::/56"
  "--advertise-tags=tag:webserver,tag:k3s-server,tag:k3s"
  "--ssh"
]
```

I then wrote [the configuration](https://github.com/lyuk98/nixos-config/blob/bd3b8b00da1a2691b9452f5aca3e513d241f4eb9/hosts/xps13/k3s.nix "nixos-config/hosts/xps13/k3s.nix at bd3b8b00da1a2691b9452f5aca3e513d241f4eb9 · lyuk98/nixos-config") that successfully ran K3s.

```nix
{ lib, config, ... }:
let
  cfg = config.services.k3s;

  # Server configuration
  server = {
    address = config.networking.hostName;
    port = 6443;
  };
in
{
  options.services.k3s = {
    # Create options for CIDRs
    clusterCidr = lib.mkOption {
      default = [ "10.42.0.0/16" ];
      example = [ "10.1.0.0/16" ];
      description = "IPv4/IPv6 network CIDRs to use for pod IPs";
      type = lib.types.listOf lib.types.str;
    };
    serviceCidr = lib.mkOption {
      default = [ "10.43.0.0/16" ];
      example = [ "10.0.0.0/24" ];
      description = "IPv4/IPv6 network CIDRs to use for service";
      type = lib.types.listOf lib.types.str;
    };
  };

  config = {
    # Template for file to pass as --vpn-auth-file
    sops.templates."k3s/vpn-auth".content = builtins.concatStringsSep "," [
      "name=tailscale"
      "joinKey=${config.sops.placeholder.tailscale-auth-key}"
      "extraArgs=${builtins.concatStringsSep " " config.services.tailscale.extraUpFlags}"
    ];

    services.k3s = {
      # Enable K3s
      enable = true;

      # Run as a server
      role = "server";

      # Set the API server's address
      serverAddr = "https://${server.address}:${builtins.toString server.port}";

      # Set network CIDRs for pod IPs and service
      clusterCidr = [
        "10.42.0.0/16"
        "2001:cafe:42::/56"
      ];
      serviceCidr = [
        "10.43.0.0/16"
        "2001:cafe:43::/112"
      ];

      # Set extra flags for K3s
      extraFlags = [
        "--cluster-cidr ${builtins.concatStringsSep "," cfg.clusterCidr}"
        "--service-cidr ${builtins.concatStringsSep "," cfg.serviceCidr}"

        # Enable Tailscale integration
        "--vpn-auth-file ${config.sops.templates."k3s/vpn-auth".path}"

        # Enable IPv6 masquerading
        "--flannel-ipv6-masq"
      ];

      # Attempt to detect node system shutdown and terminate pods
      gracefulNodeShutdown.enable = true;

      # Specify container images
      images = [
        cfg.package.airgap-images
      ];
    };

    systemd.services.k3s = {
      # Add Tailscale to PATH for integration
      path = [
        config.services.tailscale.package
      ];

      # Start K3s after tailscaled
      after = [
        config.systemd.services.tailscaled.name
      ];
    };
  };
}
```

The option `services.k3s.clusterCidr` that was used for Tailscale (`services.tailscale.extraUpFlags`) does not actually exist; I instead created one mainly to avoid redundant declarations.

```nix
options.services.k3s = {
  # Create options for CIDRs
  clusterCidr = lib.mkOption {
    default = [ "10.42.0.0/16" ];
    example = [ "10.1.0.0/16" ];
    description = "IPv4/IPv6 network CIDRs to use for pod IPs";
    type = lib.types.listOf lib.types.str;
  };
  serviceCidr = lib.mkOption {
    default = [ "10.43.0.0/16" ];
    example = [ "10.0.0.0/24" ];
    description = "IPv4/IPv6 network CIDRs to use for service";
    type = lib.types.listOf lib.types.str;
  };
};
```

To enable Tailscale integration, I had to pass an option to K3s, which they described [in their documentation](https://docs.k3s.io/networking/distributed-multicloud#integration-with-the-tailscale-vpn-provider-experimental "Distributed hybrid or multicloud cluster | K3s"):

> To deploy K3s with Tailscale integration enabled, you must add the following parameter on each of your nodes:
> 
> ```bash
> --vpn-auth="name=tailscale,joinKey=$AUTH-KEY"
> ```
> 
> or provide that information in a file and use the parameter:
> 
> ```bash
> --vpn-auth-file=$PATH_TO_FILE
> ```

I did not want `joinKey` to be stored in plain text within my Git repository, so my choice was to use `--vpn-auth-file`. With [sops-nix](https://github.com/Mic92/sops-nix "Mic92/sops-nix: Atomic secret provisioning for NixOS based on sops"), I could create a template file containing the secret value, which would be accessed by K3s later on.

```nix
# Template for file to pass as --vpn-auth-file
sops.templates."k3s/vpn-auth".content = builtins.concatStringsSep "," [
  "name=tailscale"
  "joinKey=${config.sops.placeholder.tailscale-auth-key}"
  "extraArgs=${builtins.concatStringsSep " " config.services.tailscale.extraUpFlags}"
];
```

The following, after substitution, was to be provided to K3s as a file:

```
nix-repl> :lf github:lyuk98/nixos-config/01900f9482f7d5b09badc82fd8792f67ceb9a829
nix-repl> outputs.nixosConfigurations.xps13.config.sops.templates."k3s/vpn-auth".content
"name=tailscale,joinKey=<SOPS:b2dbc10e3532d5690fd6ccab1c1d091bc898a9bd106d00c171897f812c75acdd:PLACEHOLDER>,extraArgs=--accept-routes --advertise-routes=10.42.0.0/16,2001:cafe:42::/56 --advertise-tags=tag:webserver,tag:k3s-server,tag:k3s --ssh"
```

I could not find the accepted value for `--vpn-auth` documented anywhere, so I read the code and verified that:

- [K3s internally runs](https://github.com/k3s-io/k3s/blob/v1.34.1%2Bk3s1/pkg/vpn/vpn.go#L62 "k3s/pkg/vpn/vpn.go at v1.34.1+k3s1 · k3s-io/k3s") `tailscale up`, eliminating the need to specify `services.tailscale.authKeyFile`
- `extraArgs` [can be specified](https://github.com/k3s-io/k3s/blob/v1.34.1%2Bk3s1/pkg/vpn/vpn.go#L92 "k3s/pkg/vpn/vpn.go at v1.34.1+k3s1 · k3s-io/k3s") to pass additional parameters to `tailscale up`, which is necessary to advertise tags
- `extraArgs` are [expected to be space-separated values](https://github.com/k3s-io/k3s/blob/v1.34.1%2Bk3s1/pkg/vpn/vpn.go#L155 "k3s/pkg/vpn/vpn.go at v1.34.1+k3s1 · k3s-io/k3s")

The configuration was applied, and the device was rebooted. After a while, I verified that the node is running.

```
[lyuk98@xps13:~]$ sudo kubectl cluster-info --server https://xps13:6443
Kubernetes control plane is running at https://xps13:6443
CoreDNS is running at https://xps13:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://xps13:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

# Deploying OpenStack

What was left now was to deploy OpenStack. Fortunately, for those using Kubernetes, a lot of hard work was already done by the [OpenStack-Helm](https://docs.openstack.org/openstack-helm/latest/ "Welcome to OpenStack-Helm’s documentation! — openstack-helm 2025.2.1.dev21 documentation") project, which provides [Helm charts](https://helm.sh/docs/topics/charts/ "Charts | Helm") for deploying the service over the container orchestration system.

> The OpenStack-Helm charts are published in the [openstack-helm](https://tarballs.opendev.org/openstack/openstack-helm) helm repository. Let’s enable it:
> 
> ```
> helm repo add openstack-helm https://tarballs.opendev.org/openstack/openstack-helm
> ```

Sadly, the very first step was already against my desire to build a fully declarative system. Although I admit not seeing Kubernetes itself (unlike its deployments) being set up in a completely declarative manner, I wanted to see how far I can push my ideal without resorting to persistent storage.

Issuing `helm repo add` and `helm upgrade` was not an option. Luckily, NixOS modules for K3s offers [a way](https://nixos.org/manual/nixos/stable/options#opt-services.k3s.autoDeployCharts "Appendix A. Configuration Options") to automatically deploy provided charts; I could therefore write something like the following:

```nix
{
  # Deploy Helm charts for OpenStack
  services.k3s.autoDeployCharts =
    builtins.mapAttrs
      (
        name: value:
        (
          value
          // {
            name = name;
            repo = "https://tarballs.opendev.org/openstack/openstack-helm";
            createNamespace = true;
            targetNamespace = "openstack";
          }
        )
      )
      {
        # OpenStack backend
        rabbitmq = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-OvRSmzeKtwAQxtuJskKJHdS/Xyfg+1KNaw9YTU6jR2Y=";
        }; # Message broker
        mariadb = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-my94p1+b9SkUeYBWNHXNdkIsdo5JWM7sVQuNBXMO6/s=";
        }; # Backend database
        memcached = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-wCEoCK54KzCSjLCVX+TGNtLoiLpS7l+wDbw950Wzusc=";
        }; # Distributed memory object caching system

        # OpenStack
        keystone = {
          version = "2025.2.1+a1599e717";
          hash = "sha256-Yi6CVnoYhuzLSRmyKHfq4ntqYC/FNZiGSVM3RIzn42g=";
        }; # Identity and authentication service
        heat = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-n5nYTWXXbwMafqL2y5KMUsq+WnLo24fJBgyvHJEGq1Y=";
        }; # Orchestration service
        glance = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-smwqiUE72uuARKIBIQS69pykICs0TQfXUYWAHAeW57w=";
        }; # Image service
        cinder = {
          version = "2025.2.1+a1599e717";
          hash = "sha256-E8SLZcQmTViY1Q/PUxjgay1pdb61+glGosxUcfsFRhs=";
        }; # Block storage service

        # Compute kit backend
        openvswitch = {
          version = "2025.2.1+a1599e717";
          hash = "sha256-Mk7AYhPkvbrb54fYxR606Gg9JHMnFCOCZngWC+HtLik=";
        }; # Networking backend
        libvirt = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-yQVqSXem3aGCfNRAM+qONAyFOlucG6Wfjr5/3ldqZcs=";
        }; # Libvirt service

        # Compute kit
        placement = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-+Ykc8yLPCSPwNeLzWCous3OdDjIBIQM3HsbujGnko4w=";
        }; # Placement service
        nova = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-sQF8ozH9nVA9jXUxUjnWbzB/PSjCKVLqtnL3DiNXFK8=";
        }; # Compute service
        neutron = {
          version = "2025.2.0+a1599e717";
          hash = "sha256-Czm2OdCJefuTtzDgsU4z8Uv5NqgN/YYIRIwEsfaw82g=";
        }; # Networking service

        # Horizon
        horizon = {
          version = "2025.2.1+a1599e717";
          hash = "sha256-XRCP6VFE3Ymw5lYkxymw8cGnUUAEwLTPd34zaVFzndY=";
        }; # Dashboard
      };
}
```

However, this was not at all ideal. For each of the required components, I had to manually search for the latest version and update the hash value; having more than 10 of them did not help, either.

To prevent myself from getting lost while solving this problem, I set my expectation clear: `flake.lock` must be *the* single source of truth for `rev`s and `hash`es of external dependencies. When `nix flake update` is run, all Helm charts should be updated as well.

A lot of time was spent to find out how to make the setup meet my expectation. I looked for a way to use [chart repositories](https://helm.sh/docs/topics/chart_repository "The Chart Repository Guide | Helm"), but I ultimately concluded that it is not possible to retrieve their `index.yaml` in a reproducible manner. Nix-specific projects like [nixhelm](https://github.com/nix-community/nixhelm "nix-community/nixhelm: This is a collection of helm charts in a nix-digestable format. [maintainers=@farcaller, @e1senh0rn]") and [nix-kube-generators](https://github.com/farcaller/nix-kube-generators "farcaller/nix-kube-generators: A set of functions helping to generate k8s yamls.") caught my attention, but attempting to use one for OpenStack was unsuccessful.

```
nix-repl> kubelib.fromHelm {
        >   name = "keystone";
        >   chart = "${openstack-helm}/keystone";
        > }
error:
       … while calling the 'foldl'' builtin
         at /nix/store/jnsjsx2v8c7iy5ds1ds6sfx4gl37xrjw-source/lib/default.nix:124:20:
          123|   */
          124|   fromHelm = args: pkgs.lib.pipe args [ buildHelmChart builtins.readFile fromYAML ];
             |                    ^
          125|

       … while calling the 'readFile' builtin
         at /nix/store/qjg5hnnkydk3mri5k6rydhj08x9s7xya-source/lib/trivial.nix:129:33:
          128|   */
          129|   pipe = builtins.foldl' (x: f: f x);
             |                                 ^
          130|

       (stack trace truncated; use '--show-trace' to show the full, detailed trace)

       error: Cannot build '/nix/store/igmm0rnqr38aaz9yskyyscnpn6sqzzd6-helm--nix-store-rvz13axcj010dgz7iyj3nm25s3rr76ar-source-keystone-keystone.drv'.
       Reason: builder failed with exit code 1.
       Output paths:
         /nix/store/czjx2k3r1cffbw4j471vdxn5ixpn8afi-helm--nix-store-rvz13axcj010dgz7iyj3nm25s3rr76ar-source-keystone-keystone
       Last 2 log lines:
       > Running phase: installPhase
       > Error: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: helm-toolkit
       For full logs, run:
         nix log /nix/store/igmm0rnqr38aaz9yskyyscnpn6sqzzd6-helm--nix-store-rvz13axcj010dgz7iyj3nm25s3rr76ar-source-keystone-keystone.drv
[0 built (1 failed), 0.0 MiB DL]
```

The reason for the failure was that many (if not all) charts had dependencies, especially on [helm-toolkit](https://opendev.org/openstack/openstack-helm/src/commit/a1599e717561ef6ecf7399346e396219289e7f6d/helm-toolkit "openstack-helm/helm-toolkit at a1599e717561ef6ecf7399346e396219289e7f6d - openstack-helm - OpenDev: Free Software Needs Free Tools"); if I wanted to build [Keystone](https://www.openstack.org/software/releases/dalmatian/components/keystone "Open Source Cloud Computing Platform Software - OpenStack"), for example, it meant that `helm-toolkit` should be present in the `charts` directory alongside `Chart.yaml`, which is `openstack-helm/keystone/charts` in this case.

Eventually, I decided to package the charts from source and provide them to [the option](https://nixos.org/manual/nixos/stable/options#opt-services.k3s.autoDeployCharts._name_.package "Appendix A. Configuration Options") `services.k3s.autoDeployCharts.<name>.package`. It accepts `.tgz` archives, so it looked like I could run `helm package` [command](https://helm.sh/docs/helm/helm_package "helm package | Helm"), which output K3s will happily use.

I first added [new inputs](https://github.com/lyuk98/nixos-config/blob/2385dbee5c6691ad0692343415fed21bc179c2fe/flake.nix#L84-L91 "nixos-config/flake.nix at 2385dbee5c6691ad0692343415fed21bc179c2fe · lyuk98/nixos-config") to `flake.nix`. Together with `flake.lock`, I could pinpoint the exact revision of the source.

```nix
# Functions for generating Kubernetes YAMLs
nix-kube-generators.url = "github:farcaller/nix-kube-generators";

# Helm charts for OpenStack
openstack-helm = {
  url = "git+https://opendev.org/openstack/openstack-helm";
  flake = false;
};
```

[A derivation to create chart archives](https://github.com/lyuk98/nixos-config/blob/2385dbee5c6691ad0692343415fed21bc179c2fe/hosts/xps13/openstack.nix#L11-L57 "nixos-config/hosts/xps13/openstack.nix at 2385dbee5c6691ad0692343415fed21bc179c2fe · lyuk98/nixos-config") was written afterwards. It would recursively create dependencies, which would then be copied to the `charts` directory. What was surprising to me was that the name of the dependent archives did not matter; as long as the path after the extraction matches the dependency name, `helm package` accepted them.

```nix
{
  pkgs,
  lib,
  inputs,
  ...
}:
let
  # Import nix-kube-generators
  kubelib = inputs.nix-kube-generators.lib { inherit pkgs; };

  # Create a chart archive
  packageHelmChart =
    { repo, name }:
    pkgs.stdenv.mkDerivation (finalAttrs: {
      name = "${name}.tgz";

      phases = [
        "unpackPhase"
        "patchPhase"
        "buildPhase"
      ];

      src = repo;

      postPatch = ''
        mkdir --parents --verbose ${name}/charts
        ${lib.pipe "${repo}/${name}/Chart.yaml" [
          # Read Chart.yaml
          builtins.readFile

          # Convert YAML to an attribute set
          kubelib.fromYAML
          (docs: builtins.elemAt docs 0)

          # Get dependencies
          (yaml: if (builtins.hasAttr "dependencies" yaml) then yaml.dependencies else [ ])

          # Create chart archives of dependencies
          (builtins.map (
            dependency:
            packageHelmChart {
              inherit repo;
              name = dependency.name;
            }
          ))

          # Command to copy each chart archive to where Helm expects
          (builtins.map (chart: "cp --verbose ${chart} ${name}/charts/"))
          (builtins.concatStringsSep "\n")
        ]}
      '';

      buildPhase = ''
        ${pkgs.kubernetes-helm}/bin/helm package ${name}
        cp --verbose ${name}-*.tgz $out
      '';
    });
in
{ }
```

The charts could then [be defined](https://github.com/lyuk98/nixos-config/blob/2385dbee5c6691ad0692343415fed21bc179c2fe/hosts/xps13/openstack.nix#L60-L117 "nixos-config/hosts/xps13/openstack.nix at 2385dbee5c6691ad0692343415fed21bc179c2fe · lyuk98/nixos-config") without the tedious `hash`-keeping. Some values were also added by roughly following [the guide from OpenStack-Helm](https://docs.openstack.org/openstack-helm/latest/install/openstack.html "Deploy OpenStack — openstack-helm 2025.2.1.dev21 documentation").

```nix
# Deploy Helm charts for OpenStack
services.k3s.autoDeployCharts =
  builtins.mapAttrs
    (
      name: value:
      (
        value
        // {
          package = packageHelmChart {
            repo = "${inputs.openstack-helm}";
            name = name;
          };
          createNamespace = true;
          targetNamespace = "openstack";
        }
      )
    )
    {
      # OpenStack backend
      rabbitmq = {
        values = {
          pod.replicas.server = 1;
        };
      }; # Message broker
      mariadb = {
        values = {
          pod.replicas.server = 1;
        };
      }; # Backend database
      memcached = { }; # Distributed memory object caching system

      # OpenStack
      keystone = { }; # Identity and authentication service
      heat = { }; # Orchestration service
      glance = { }; # Image service
      cinder = { }; # Block storage service

      # Compute kit backend
      openvswitch = { }; # Networking backend
      libvirt = {
        values = {
          conf.ceph.enabled = true;
        };
      }; # Libvirt service

      # Compute kit
      placement = { }; # Placement service
      nova = {
        values = {
          bootstrap.wait_for_computes.enabled = true;
          conf.ceph.enabled = true;
        };
      }; # Compute service
      neutron = { }; # Networking service

      # Horizon
      horizon = { }; # Dashboard
    };
```

## Troubleshooting and realising my mistake

After applying the configuration, I checked the pods that were deployed to the cluster.

```
[lyuk98@xps13:~]$ sudo kubectl get pods --namespace openstack
NAME                                  READY   STATUS    RESTARTS   AGE
cinder-api-f69f7b5cd-r82bj            0/1     Pending   0          2m44s
cinder-backup-f894f7fbd-5blr7         0/1     Pending   0          2m44s
cinder-db-init-rmwlp                  0/1     Pending   0          2m44s
cinder-scheduler-748d88959-8q6g7      0/1     Pending   0          2m44s
cinder-volume-658dfdd644-7vg6j        0/1     Pending   0          2m44s
glance-api-5b669d9488-wq7v4           0/1     Pending   0          2m47s
glance-db-init-qf978                  0/1     Pending   0          2m46s
heat-api-5dc6df8988-lkjmh             0/1     Pending   0          2m45s
heat-cfn-6fb6648b76-sqjgg             0/1     Pending   0          2m45s
heat-db-init-zptcg                    0/1     Pending   0          2m44s
heat-engine-79bdc57644-2gj97          0/1     Pending   0          2m45s
horizon-97bbb4b44-nfq4s               0/1     Pending   0          2m50s
horizon-db-init-z4tp8                 0/1     Pending   0          2m49s
keystone-api-76979f4467-p428r         0/1     Pending   0          2m46s
keystone-credential-setup-4x42w       0/1     Pending   0          2m45s
mariadb-controller-6fbff6dc68-m4wdt   0/1     Pending   0          2m49s
mariadb-server-0                      0/1     Pending   0          2m49s
memcached-memcached-0                 0/1     Pending   0          2m56s
neutron-db-init-g9kjc                 0/1     Pending   0          2m43s
neutron-rpc-server-79d6c95445-f2lbd   0/1     Pending   0          2m44s
neutron-server-5b9c476b6c-vvnxx       0/1     Pending   0          2m44s
nova-api-metadata-5c7644b9d8-nxvwq    0/1     Pending   0          2m43s
nova-api-osapi-7c79f646fb-nfxlg       0/1     Pending   0          2m43s
nova-bootstrap-d8cng                  0/1     Pending   0          2m42s
nova-cell-setup-6vtrk                 0/1     Pending   0          2m42s
nova-conductor-c554bfb4b-gzptg        0/1     Pending   0          2m43s
nova-novncproxy-5dd449bf9b-msf4f      0/1     Pending   0          2m43s
nova-scheduler-79947f684b-s9p9c       0/1     Pending   0          2m43s
nova-storage-init-kw7v4               0/1     Pending   0          2m42s
placement-api-7b468676b8-kh7r4        0/1     Pending   0          2m50s
placement-db-init-ggccl               0/1     Pending   0          2m49s
rabbitmq-cluster-wait-qd28n           0/1     Pending   0          2m50s
rabbitmq-rabbitmq-0                   0/1     Pending   0          2m51s
```

Everything seemed to be present, but none was actually running. To see what was going on, I checked RabbitMQ, which was the first component mentioned in [this guide](https://docs.openstack.org/openstack-helm/latest/install/openstack.html "Deploy OpenStack — openstack-helm 2025.2.1.dev21 documentation").

```
[lyuk98@xps13:~]$ kubectl describe pod rabbitmq-rabbitmq-0 --namespace openstack
Name:             rabbitmq-rabbitmq-0
Namespace:        openstack
Priority:         0
Service Account:  rabbitmq-rabbitmq
Node:             <none>
Labels:           app.kubernetes.io/component=server
                  app.kubernetes.io/instance=rabbitmq
                  app.kubernetes.io/name=rabbitmq
                  application=rabbitmq
                  apps.kubernetes.io/pod-index=0
                  component=server
                  controller-revision-hash=rabbitmq-rabbitmq-cdc9bc4d6
                  release_group=rabbitmq
                  statefulset.kubernetes.io/pod-name=rabbitmq-rabbitmq-0
Annotations:      configmap-bin-hash: 945fe1a250212a5fa9d938bd8aed06a2210becfc09c71fbab3728846aae9402b
                  configmap-etc-hash: d3af926b9739d4f54f00475d1cfdd365b894471d06f4d012c94778ed57c750ad
                  openstackhelm.openstack.org/release_uuid: 
                  secret-erlang-cookie-hash: f9c2a35397a22b98d875eec1818b3f87bbd58e7d4976a0c5fc1087700958862d
                  secret-rabbit-admin-hash: b819675eb8c344ecf626d029dc9807a95bfc03171182227a79fdd41718c81f07
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    StatefulSet/rabbitmq-rabbitmq
Init Containers:
  init:
    Image:      quay.io/airshipit/kubernetes-entrypoint:latest-ubuntu_focal
    Port:       <none>
    Host Port:  <none>
    Command:
      kubernetes-entrypoint
    Environment:
      POD_NAME:                    rabbitmq-rabbitmq-0 (v1:metadata.name)
      NAMESPACE:                   openstack (v1:metadata.namespace)
      INTERFACE_NAME:              eth0
      PATH:                        /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/
      DEPENDENCY_SERVICE:          
      DEPENDENCY_DAEMONSET:        
      DEPENDENCY_CONTAINER:        
      DEPENDENCY_POD_JSON:         
      DEPENDENCY_CUSTOM_RESOURCE:  
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2ng2n (ro)
  rabbitmq-password:
    Image:      docker.io/openstackhelm/heat:2025.1-ubuntu_noble
    Port:       <none>
    Host Port:  <none>
    Command:
      /tmp/rabbitmq-password-hash.py
    Environment:
      RABBITMQ_ADMIN_USERNAME:   <set to the key 'RABBITMQ_ADMIN_USERNAME' in secret 'rabbitmq-admin-user'>  Optional: false
      RABBITMQ_ADMIN_PASSWORD:   <set to the key 'RABBITMQ_ADMIN_PASSWORD' in secret 'rabbitmq-admin-user'>  Optional: false
      RABBITMQ_GUEST_PASSWORD:   <set to the key 'RABBITMQ_GUEST_PASSWORD' in secret 'rabbitmq-admin-user'>  Optional: false
      RABBITMQ_DEFINITION_FILE:  /var/lib/rabbitmq/definitions.json
    Mounts:
      /tmp from pod-tmp (rw)
      /tmp/rabbitmq-password-hash.py from rabbitmq-bin (ro,path="rabbitmq-password-hash.py")
      /var/lib/rabbitmq from rabbitmq-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2ng2n (ro)
  rabbitmq-cookie:
    Image:      docker.io/library/rabbitmq:3.13.0
    Port:       <none>
    Host Port:  <none>
    Command:
      /tmp/rabbitmq-cookie.sh
    Environment:  <none>
    Mounts:
      /tmp from pod-tmp (rw)
      /tmp/rabbitmq-cookie.sh from rabbitmq-bin (ro,path="rabbitmq-cookie.sh")
      /var/lib/rabbitmq from rabbitmq-data (rw)
      /var/run/lib/rabbitmq/.erlang.cookie from rabbitmq-erlang-cookie (ro,path="erlang_cookie")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2ng2n (ro)
  rabbitmq-perms:
    Image:      docker.io/library/rabbitmq:3.13.0
    Port:       <none>
    Host Port:  <none>
    Command:
      chown
      -R
      999
      /var/lib/rabbitmq
    Environment:  <none>
    Mounts:
      /tmp from pod-tmp (rw)
      /var/lib/rabbitmq from rabbitmq-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2ng2n (ro)
Containers:
  rabbitmq:
    Image:       docker.io/library/rabbitmq:3.13.0
    Ports:       15672/TCP (http), 5672/TCP (amqp), 25672/TCP (clustering), 15692/TCP (metrics)
    Host Ports:  0/TCP (http), 0/TCP (amqp), 0/TCP (clustering), 0/TCP (metrics)
    Command:
      /tmp/rabbitmq-start.sh
    Liveness:   exec [/tmp/rabbitmq-liveness.sh] delay=60s timeout=10s period=10s #success=1 #failure=5
    Readiness:  exec [/tmp/rabbitmq-readiness.sh] delay=10s timeout=10s period=10s #success=1 #failure=3
    Environment:
      MY_POD_NAME:             rabbitmq-rabbitmq-0 (v1:metadata.name)
      MY_POD_IP:                (v1:status.podIP)
      RABBITMQ_USE_LONGNAME:   true
      RABBITMQ_NODENAME:       rabbit@$(MY_POD_NAME).rabbitmq.openstack.svc.cluster.local
      K8S_SERVICE_NAME:        rabbitmq
      K8S_HOSTNAME_SUFFIX:     .rabbitmq.openstack.svc.cluster.local
      RABBITMQ_ERLANG_COOKIE:  openstack-cookie
      PORT_HTTP:               15672
      PORT_AMPQ:               5672
      PORT_CLUSTERING:         25672
    Mounts:
      /etc/rabbitmq/conf.d/management_agent.disable_metrics_collector.conf from rabbitmq-etc (ro,path="management_agent.disable_metrics_collector.conf")
      /etc/rabbitmq/enabled_plugins from rabbitmq-etc (ro,path="enabled_plugins")
      /etc/rabbitmq/rabbitmq-env.conf from rabbitmq-etc (ro,path="rabbitmq-env.conf")
      /etc/rabbitmq/rabbitmq.conf from rabbitmq-etc (ro,path="rabbitmq.conf")
      /tmp from pod-tmp (rw)
      /tmp/rabbitmq-liveness.sh from rabbitmq-bin (ro,path="rabbitmq-liveness.sh")
      /tmp/rabbitmq-readiness.sh from rabbitmq-bin (ro,path="rabbitmq-readiness.sh")
      /tmp/rabbitmq-start.sh from rabbitmq-bin (ro,path="rabbitmq-start.sh")
      /var/lib/rabbitmq from rabbitmq-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2ng2n (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  rabbitmq-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rabbitmq-data-rabbitmq-rabbitmq-0
    ReadOnly:   false
  pod-tmp:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  rabbitmq-bin:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      rabbitmq-rabbitmq-bin
    Optional:  false
  rabbitmq-etc:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      rabbitmq-rabbitmq-etc
    Optional:  false
  rabbitmq-erlang-cookie:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  rabbitmq-erlang-cookie
    Optional:    false
  kube-api-access-2ng2n:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              openstack-control-plane=enabled
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  53s   default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
```

Something was apparently not right with the [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/ "Persistent Volumes | Kubernetes"). I checked what PersistentVolumeClaims were present:

```
[lyuk98@xps13:~]$ sudo kubectl get pvc --namespace openstack
NAME                                STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-data-mariadb-server-0         Pending                                      general        <unset>                 168m
rabbitmq-data-rabbitmq-rabbitmq-0   Pending                                      general        <unset>                 168m
```

`rabbitmq-data-rabbitmq-rabbitmq-0` seemed to be the related one, so I went to see what went wrong with it.

```
[lyuk98@xps13:~]$ sudo kubectl describe pvc rabbitmq-data-rabbitmq-rabbitmq-0 --namespace openstack
Name:          rabbitmq-data-rabbitmq-rabbitmq-0
Namespace:     openstack
StorageClass:  general
Status:        Pending
Volume:        
Labels:        app.kubernetes.io/component=server
               app.kubernetes.io/instance=rabbitmq
               app.kubernetes.io/name=rabbitmq
               application=rabbitmq
               component=server
               release_group=rabbitmq
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       rabbitmq-rabbitmq-0
Events:
  Type     Reason              Age                     From                         Message
  ----     ------              ----                    ----                         -------
  Warning  ProvisioningFailed  4m18s (x662 over 169m)  persistentvolume-controller  storageclass.storage.k8s.io "general" not found
```

A [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/ "Storage Classes | Kubernetes") named `general` was missing. As I ran `kubectl get sc`, I could see that it was not there.

```
[lyuk98@xps13:~]$ sudo kubectl get sc --namespace openstack
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  172m
```

It was at that point when I realised my mistake: I did not go through OpenStack-Helm's guide on [its prerequisites](https://docs.openstack.org/openstack-helm/latest/install/prerequisites.html "Kubernetes prerequisites — openstack-helm 2025.2.1.dev21 documentation"). One of the requirements was a storage system, and it was clear that I missed an important part.

> [...] By default OpenStack-Helm stateful sets expect to find a storage class named **general**.

Naturally, my next step was to deploy those prerequisites: an ingress controller, MetalLB, and a storage system.

Unlike components from OpenStack-Helm, I could now use [nixhelm](https://github.com/nix-community/nixhelm "nix-community/nixhelm: This is a collection of helm charts in a nix-digestable format. [maintainers=@farcaller, @e1senh0rn]") that would download up-to-date Helm charts without sacrificing reproducibility. [It was added](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/flake.nix#L87-L88 "nixos-config/flake.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") to `flake.nix` before starting with deployments.

```nix
{
  inputs = {
    # previous inputs

    # Functions for generating Kubernetes YAMLs
    nix-kube-generators.url = "github:farcaller/nix-kube-generators";

    # Helm charts collection
    nixhelm.url = "github:nix-community/nixhelm";

    # Helm charts for OpenStack
    openstack-helm = {
      url = "git+https://opendev.org/openstack/openstack-helm";
      flake = false;
    };
  };
}
```

## Deploying Tailscale Kubernetes Operator

Prior to deploying an ingress controller, I had a look into the [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator "Kubernetes operator · Tailscale Docs").

A tag for the operator [was created](https://login.tailscale.com/admin/acls/visual/tags/add "Create tag - Tailscale") at the Tailscale admin console. Because Services it will manage were to be given the tag `tag:k3s`, I added `tag:k3s-operator` as one of its owners.

```jsonc
// Define the tags which can be applied to devices and by which users.
"tagOwners": {
    // existing tags

    "tag:k3s":          ["tag:k3s-server", "tag:k3s-operator"],
    "tag:k3s-server":   ["autogroup:admin"],
    "tag:k3s-operator": ["autogroup:admin"],
    "tag:openstack":    ["tag:k3s-operator"],
},
```

An OAuth credential, with the tag `tag:k3s-operator` and scopes `devices:core` and `auth_keys`, was then created.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/be5466a1-d6ac-4dc9-8ad5-22df4916a203.avif">
  <img src="https://images.lyuk98.com/0f65d169-09d6-48cc-918b-e73a2d7cfbf7.avif" alt="A dialog for adding a new credential at Tailscale. Checkbox &quot;Write&quot; is checked for the scopes &quot;Core&quot; and &quot;Auth Keys&quot;. Each scope has a tag &quot;tag:k3s-operator&quot; specified." title="Adding a new credential">
</picture>

The credential was added to `secrets.yaml` afterwards.

```
[lyuk98@framework:~/nixos-config]$ sops edit hosts/xps13/secrets.yaml
```

```yaml
machine-id: 7e7f3bdd40f715ee577580a868f47504
tailscale-auth-key: tskey-client-kvVL1oNXyt11CNTRL-n9uTX7M71e2xV9DCTvSmd2UYXzrGbGY9P
operator-oauth:
  client-id: kU35kotXFj11CNTRL
  client-secret: tskey-client-kU35kotXFj11CNTRL-BDzjYjzSMqAP2J9cnij8qADa9napq8sTM
```

The secrets were present, and I subsequently [wrote a Secret template](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L81-L95 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") where the placeholders will be substituted with decrypted values. The operator [tries to read the Secret](https://github.com/tailscale/tailscale/blob/v1.90.8/cmd/k8s-operator/deploy/chart/values.yaml#L18 "tailscale/cmd/k8s-operator/deploy/chart/values.yaml at v1.90.8 · tailscale/tailscale") named `operator-oauth` by default, so it naturally became the name of the manifest.

```nix
sops = {
  secrets = {
    # Tailscale Kubernetes Operator's credential
    "operator-oauth/client-id".sopsFile = ./secrets.yaml;
    "operator-oauth/client-secret".sopsFile = ./secrets.yaml;
  };

  templates = {
    # Template for file to pass as --vpn-auth-file
    "k3s/vpn-auth".content = builtins.concatStringsSep "," [
      "name=tailscale"
      "joinKey=${config.sops.placeholder.tailscale-auth-key}"
      "extraArgs=${builtins.concatStringsSep " " config.services.tailscale.extraUpFlags}"
    ];

    # Template for secret operator-oauth
    # YAML is a superset of JSON, so this can be used to write a manifest file
    "operator-oauth.yaml".content = builtins.toJSON {
      apiVersion = "v1";
      kind = "Secret";
      metadata = {
        name = "operator-oauth";
        namespace = "tailscale";
      };
      stringData = {
        client_id = config.sops.placeholder."operator-oauth/client-id";
        client_secret = config.sops.placeholder."operator-oauth/client-secret";
      };
      immutable = true;
    };
  };
};
```

It was now time to deploy the chart. At first, I thought I could use nixhelm this way:

```nix
services.k3s.autoDeployCharts = {
  # Tailscale Kubernetes Operator
  tailscale-operator =
    let
      chart = inputs.nixhelm.chartsMetadata.tailscale.tailscale-operator;
    in
    {
      repo = chart.repo;
      name = chart.chart;
      hash = chart.chartHash;
      version = chart.version;
    };
};
```

However, the `hash` was apparently not what the K3s module expected it to be.

```
[lyuk98@framework:~/nixos-config]$ nixos-rebuild build --flake .#xps13
building the system configuration...
error: hash mismatch in fixed-output derivation '/nix/store/nv2zcnh2hxw91bgh3m0lnjj38w7xf19g-pkgs.tailscale.com-helmcharts--tailscale-operator-1.90.8.tgz.drv':
         specified: sha256-orJdAcLRUKrxBKbG3JZr7L390+A1tCgAchDzdUlyT+o=
            got:    sha256-89MSeInAckiBsCK0ag2hrflVBWGgVvl22uy8xk9HU2g=
error: Cannot build '/nix/store/7355kcfxx83s39w12sbbb7sdly7n81wl-10-k3s.conf.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/khf4wsm6d6f7xwzy7y0ps6pg1asqb0lm-10-k3s.conf
error: Cannot build '/nix/store/30qspswgn2v56gkp1vz3vfwvgqgywxsj-tmpfiles.d.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/4scv2jdjf56byk921c2aqbzaap9g24rs-tmpfiles.d
error: Cannot build '/nix/store/piizalya7acffqwa7mfr8z8dyx29qmhx-etc.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/h8vvnq05mh5j3ga4lqh89c91z1hfs95c-etc
error: Cannot build '/nix/store/784qbaghgj60q8qk04jqwi7m30gz47mj-nixos-system-xps13-25.11.20251116.50a96ed.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/ljrrq7iysgb1kcnb8fm6aprzqfs206kv-nixos-system-xps13-25.11.20251116.50a96ed
Command 'nix --extra-experimental-features 'nix-command flakes' build --print-out-paths '.#nixosConfigurations."xps13".config.system.build.toplevel'' returned non-zero exit status 1.
```

To work around the problem, I decided to package charts again. First, I made sure that the derivations from nixhelm are what `helm package` can work with:

```
[lyuk98@framework:~]$ nix build github:nix-community/nixhelm#chartsDerivations.x86_64-linux.tailscale.tailscale-operator
[lyuk98@framework:~]$ ls -l result/
total 12
-r--r--r-- 2 root root  381 Jan  1  1970 Chart.yaml
dr-xr-xr-x 1 root root  372 Jan  1  1970 templates
-r--r--r-- 6 root root 5774 Jan  1  1970 values.yaml
```

Based on the previous one for OpenStack-Helm, I [wrote another small derivation](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L20-L38 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") that would package the chart into a `.tgz` file. I did not have to worry about copying dependencies here, since they were already included in chart derivations.

```nix
# Create a chart archive from nixhelm derivation
packageHelmChart =
  chart:
  pkgs.stdenv.mkDerivation {
    name = "${chart.name}.tgz";

    phases = [
      "unpackPhase"
      "buildPhase"
    ];

    src = "${chart}";

    buildPhase = ''
      cd ../
      ${pkgs.kubernetes-helm}/bin/helm package ${chart.name}
      cp --verbose *.tgz $out
    '';
  };
```

From an existing set of options, I [added another one](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L56-L62 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") for other modules to easily reference functions I just wrote.

```nix
options.services.k3s = {
  # existing options

  lib = lib.mkOption {
    default = {
      inherit packageHelmChart;
    };
    description = "Common functions for K3s modules";
    type = lib.types.attrs;
  };
};
```

The operator [was now set](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L172-L185 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") to be deployed. Because I used a different tag (`tag:k3s-operator`) for it, I updated the `values` for the chart to use the correct one.

```nix
services.k3s.autoDeployCharts =
  let
    k3slib = config.services.k3s.lib;
    charts = inputs.nixhelm.charts { inherit pkgs; };
  in
  {
    # Tailscale Kubernetes Operator
    tailscale-operator = {
      name = "tailscale-operator";
      package = k3slib.packageHelmChart charts.tailscale.tailscale-operator;
      targetNamespace = "tailscale";

      values = {
        # Use custom tag and hostname for the operator
        operatorConfig = {
          defaultTags = [ "tag:k3s-operator" ];
          hostname = "k3s-operator";
        };
      };
    };
  };
```

I then added three manifests:

1. A [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/ "Namespaces | Kubernetes") [was defined](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L140-L149 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config"); it had to be defined here, since other declarative manifests were not able to refer to it if it was [done within chart declarations](https://nixos.org/manual/nixos/stable/options#opt-services.k3s.autoDeployCharts._name_.createNamespace "Appendix A. Configuration Options") at `services.k3s.autoDeployCharts`.
2. A Secret [was defined](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L151-L154 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") based on [the template](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L81-L95 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") (`sops.templates."operator-oauth.yaml"`) I have written earlier.
3. A [ProxyGroup](https://github.com/tailscale/tailscale/blob/v1.90.8/k8s-operator/api.md#proxygroup "tailscale/k8s-operator/api.md at v1.90.8 · tailscale/tailscale") [was defined](https://github.com/lyuk98/nixos-config/blob/80cfe61d3170f4d68822e8bb9814a9efcef60b8a/hosts/xps13/k3s.nix#L156-L168 "nixos-config/hosts/xps13/k3s.nix at 80cfe61d3170f4d68822e8bb9814a9efcef60b8a · lyuk98/nixos-config") by following [an optional guide](https://tailscale.com/kb/1236/kubernetes-operator#optional-pre-creating-a-proxygroup "Kubernetes operator · Tailscale Docs").

```nix
services.k3s.manifests = {
  # Namespace for Tailscale
  # Setting autoDeployCharts.tailscale-operator.createNamespace to true does not create one
  # early enough for manifests
  namespace-tailscale.content = {
    apiVersion = "v1";
    kind = "Namespace";
    metadata = {
      name = "tailscale";
    };
  };

  # Secret for Tailscale Kubernetes Operator
  operator-oauth = {
    source = config.sops.templates."operator-oauth.yaml".path;
  };

  # Pre-creation of multi-replica ProxyGroup
  ts-proxies.content = {
    apiVersion = "tailscale.com/v1alpha1";
    kind = "ProxyGroup";
    metadata = {
      name = "ts-proxies";
    };
    spec = {
      type = "egress";
      tags = [ "tag:k3s" ];
      replicas = 3;
    };
  };
};
```

The change was applied, and I could now see the pods running.

```
[lyuk98@xps13:~]$ sudo kubectl get pods --namespace tailscale
NAME                      READY   STATUS    RESTARTS   AGE
operator-f547d778-79tmv   1/1     Running   0          93s
ts-proxies-0              1/1     Running   0          68s
ts-proxies-1              1/1     Running   0          56s
ts-proxies-2              1/1     Running   0          53s
```

The Tailscale Kubernetes Operator was ready, but there was a problem: the devices that are registered to the tailnet are not ephemeral. Based on what I have read from [an issue](https://github.com/tailscale/tailscale/issues/10166 "FR: Support K8s Operator creating ephemeral auth keys · Issue #10166 · tailscale/tailscale") and [a pull request](https://github.com/tailscale/tailscale/pull/15508 "cmd/k8s-operator: Add ephemeral node support to k8s-operator by rajsinghtech · Pull Request #15508 · tailscale/tailscale") that was ultimately not merged, creating ephemeral nodes with the operator apparently requires careful considerations. In the end, though, because rebooting the device registers separate nodes to the tailnet, I had to remove unused ones, either manually or in an automated way.

I decided not to work on the solution for now; it was up to my future self, who might find this problem more annoying than how much it did previously.

---

# Reconsidering the project

At this point, I have run into several issues that would not have happened if I chose to follow others' setup. That in itself was not a problem to me, but as I learned more about what I tried to deploy, it was clear to me that deploying OpenStack is best done with [pretty large computing resources](https://www.reddit.com/r/homelab/comments/tsxumz/my_modest_homelab_with_openstack/ "My modest homelab with Openstack : r/homelab"), which I simply do not have.

Another thing was that I could not find a personal use case for the cloud computing infrastructure outside... running Kubernetes. To me, who is not really running a production service, it looked like an additional complexity to maintain.

As a result, I decided to focus on other things for now, likely starting with Kubernetes. If I ever happen to learn OpenStack, either by personal choice or out of desperation, I may continue working on this again.
