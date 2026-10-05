# homelab

A two-node Proxmox cluster running a Docker media stack and an OpenMediaVault NAS,
built to learn infrastructure properly rather than to look impressive on a dashboard.

The interesting parts of this repo are not the diagram — they are the two hardware
faults in [troubleshooting/](troubleshooting/), both of which took the entire cluster
offline and both of which looked identical from outside.

---

## What runs here

| Layer | What |
|---|---|
| Compute | Proxmox VE, 2 nodes, quorate |
| Storage | OpenMediaVault VM with direct disk passthrough, shared over NFS/SMB |
| Services | 16 Docker containers — media automation, download client, request management, dashboards |
| Networking | Flat `/24`, Pi-hole for DNS, Tailscale for remote access |

Full detail: [architecture.md](architecture.md)

## The two faults worth reading about

**Both present identically from the outside:** every VM loses network, the host stops
responding, only a power cycle fixes it. Different causes, different fixes.

### [Intel I219-LM NIC link drops](troubleshooting/nic-link-drops.md)

The whole cluster going down turned out to be a firmware quirk in the onboard NIC.
Disabling EEE, ASPM and Wake-on-LAN was the obvious fix — **and it did not hold.** The
host hung twice more with all three applied. The actual cause was packet offloads
triggering an e1000e hardware unit hang.

That detail is the reason this write-up exists. A version that stopped at the first
configuration which appeared to work would have left the problem half-solved and
undocumented.

### [i915 crash during VFIO passthrough](troubleshooting/vfio-i915-hang.md)

The same symptom, a completely different cause — a driver ordering problem during GPU
handoff. Fixed by forcing `vfio-pci` to load before `i915`, then superseded by not doing
GPU passthrough at all and using the host iGPU for VAAPI transcoding instead.

## Compose files

`compose/` holds sanitised versions of the running files: no host addresses, no
credentials, no tailnet names. Every secret comes from a `.env` file which is gitignored
and `chmod 600`.

They were originally not published, and that was the right call at the time — they held
credentials in plaintext `environment:` blocks. The fix was to externalise those into
`.env` rather than to keep hiding the files.

Each service lives in its own directory with its own `docker-compose.yml` and
`.env.example`:

```
compose/
  immich/    docker-compose.yml  .env.example
  firefly/   docker-compose.yml  .env.example
  pihole/    docker-compose.yml  .env.example
```

The naming is deliberate. The compose file is called `docker-compose.yml` so Compose
finds it without an explicit `-f`, and each service keeps its own `.env` rather than
sharing one — a shared `.env` gets overwritten when you set up the next service, and
the variables collide with the wrong values. To bring one up:

```bash
cd compose/immich
cp .env.example .env      # then fill it in
docker compose up -d
```

The `.env` files are already covered by the `.gitignore` in this directory.

`architecture.md` describes the shape of the stack.

## What I would do differently

- **Segment the network.** Flat `/24` is fine at home and wrong anywhere with a
  compliance obligation. VLANs were never needed here; that was a shortcut.
- **Back up somewhere else.** Media and backups share a host. One disk failure loses
  both, which makes the backup theatre.
- **Document faults as they happen.** Both write-ups were reconstructed after the fact.
  A timestamped note at the moment of failure would have saved an hour of guessing.

## Licence

MIT
