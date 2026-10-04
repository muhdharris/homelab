# i915 crash during VFIO passthrough

A Proxmox host that hung whenever a GPU-passthrough VM started or stopped. Superficially
identical to [NIC link drops](nic-link-drops.md) — every VM loses network, the host stops
responding — but a completely different cause and fix.

## Symptom

- `drm_mode_config_cleanup` WARNING appears in `dmesg`
- Host hangs when a VM with `hostpci0` passthrough starts **or stops**
- Nothing happens until a physical power cycle

## Cause

The Intel UHD 750's `i915` driver does not handle device handoff to VFIO cleanly. The
display driver still holds the device when passthrough claims it.

## Fix

### Step 1 — bind the GPU to `vfio-pci` at boot

`/etc/modprobe.d/vfio.conf`:

```
softdep i915 pre: vfio-pci
options vfio-pci ids=8086:4c8a
```

The `softdep` line is the part that matters. It forces `vfio-pci` to load **before**
`i915`, so `i915` never gets first claim on the device. Without it the ordering is
non-deterministic and the crash returns intermittently.

`8086:4c8a` is the UHD 750's PCI ID — confirm yours with `lspci -nn | grep VGA`.

### Step 2 — kernel parameters

`/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet i915.enable_guc=2 i915.enable_psr=0 i915.enable_dc=0"
```

Then `update-grub && reboot`.

## The better answer: don't do GPU passthrough at all

Media transcoding does not need it. The host iGPU's VAAPI capability is available to
containers directly, which removes the whole class of problem — no handoff, no kernel
ordering to get right, no crash surface.

```yaml
services:
  jellyfin:
    devices:
      - /dev/dri:/dev/dri      # VAAPI, host iGPU, no passthrough
```

If the goal is transcoding rather than gaming or 3D workloads, this is the configuration
to use. The passthrough steps above remain documented because the crash is worth
recognising when it appears on other hardware.

## Diagnosing

```bash
dmesg | grep -iE 'i915|drm|vfio'      # look for drm_mode_config_cleanup
lspci -nnk | grep -A2 VGA             # confirm which driver has claimed the GPU
cat /etc/modprobe.d/vfio.conf          # verify softdep is present
```

## Related

[NIC link drops](nic-link-drops.md) — same outward symptom, different fault.
