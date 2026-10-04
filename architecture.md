# Architecture

Two physical machines, virtualised with Proxmox, running a Docker host and a NAS.

## Topology

```
                         ┌─────────────────────────────┐
                         │  Router  192.168.100.1      │
                         │  DHCP · DNS · Pi-hole        │
                         └──────────────┬──────────────┘
                                        │ 192.168.100.0/24
                    ┌───────────────────┴───────────────────┐
                    │                                       │
         ┌──────────┴──────────┐              ┌─────────────┴─────────┐
         │  pve                │              │  pve2                 │
         │  OptiPlex 5090      │              │  old laptop           │
         │  i5-11500 6c/12t    │              │  Ryzen 7, 8GB         │
         │  I219-LM 1Gbps      │              │  10Gbps                │
         │  Intel UHD 750      │              └─────────────┬─────────┘
         └───┬────────┬────┬───┘                            │
             │        │    │                                │
       ┌─────┘        │    └──────────┐         ┌───────────┴────────┐
       │              │               │         │  LXC  agent        │
  ┌────┴────┐   ┌─────┴─────┐   ┌─────┴────┐    │  automation + LLM   │
  │ VM 100  │   │ VM 102    │   │ LXC 101  │    └────────────────────┘
  │ Docker  │   │ NAS       │   │ VPN mesh │
  │ 4c/7GB  │   │ OMV       │   │ (off)    │
  └────┬────┘   └─────┬─────┘   └──────────┘
       │              │
       │        ┌─────┴──────────────────────────┐
       │        │  NFS / SMB                    │
       │        │  media · vault · backups      │
  ┌────┴─────────────────┐                      │
  │ Docker bridge        │──────────────────────┘
  │ homelab_default      │   NFS: the Docker host mounts
  │ service DNS resolves │   the NAS directly for media
  └──────────────────────┘

  Tailscale overlay ──► remote access without exposing ports
```

## Hosts

| Host | Machine | Specs | Role |
|---|---|---|---|
| `pve` | Dell OptiPlex 5090 | i5-11500 (Tiger Lake, 6c/12t), 256GB SSD + 1TB HDD, I219-LM 1Gbps, UHD 750 | Primary node — runs all the VMs |
| `pve2` | Retired laptop | Ryzen 7 3000 series, 8GB, 512GB SSD, 10Gbps | Second node, HA-capable |

One host does all the work. The second exists so the cluster can survive a failure of
either node — which, given the hardware faults documented in
[troubleshooting/](troubleshooting/), was not a hypothetical concern.

## Virtual machines

| ID | Name | Purpose |
|---|---|---|
| 100 | Docker | All containerised services |
| 102 | NAS | OpenMediaVault, direct disk passthrough |
| 103 | Ubuntu | Scratch VM, normally off |

## Container

| ID | Name | Purpose |
|---|---|---|
| 101 | VPN | Tailscale mesh |
| 104 | agent | Automation and assistant workloads |

## Networking

- Single `/24`, flat. Segmentation is at the container network level, not the VLAN
  level — the flat LAN is the main thing I would change given a real brief.
- Pi-hole for network-wide DNS filtering
- Containers share a Docker bridge network, so services resolve by name internally
- Tailscale provides remote access without opening inbound ports

## Design decisions

**Disk passthrough for the NAS.** The NAS VM gets the physical disk directly rather than
a virtual disk on an image. Rebuilds and upgrades can't corrupt it, and the filesystem
has no layer between it and the hardware.

**OptiPlex as the primary node.** It is not fast, but it is reliable hardware with
replaceable parts — which turned out to matter more than throughput.

**Media on the NAS, config on local disk.** Media is large and backed up to the NAS;
service configuration stays on local SSD where reads are fast.

**No GPU passthrough.** Transcoding uses VAAPI on the host iGPU. This started as a crash
fix and turned out to be the right call generally — see
[troubleshooting/vfio-i915-hang.md](troubleshooting/vfio-i915-hang.md).

## Known weaknesses

- **The primary host has two separate faults that both present as "the cluster is
  down."** Both are documented and fixed, but the symptom overlap makes diagnosis slower
  than it should be.
- **Flat network.** Fine for a homelab, wrong for anything with a compliance obligation.
- **The second node runs a laptop.** Sufficient for HA, not for anything load-bearing.
- **Backups are on the same host as the data they protect.** A single disk failure loses
  both.
