# Intel I219-LM NIC link drops

A two-node Proxmox cluster where **every VM lost network simultaneously** and the host
became unresponsive. The physical layer was fine — the NIC itself was dropping its link.

## Symptom

- All VMs lose network at the same instant
- Red Ethernet light on the NIC
- No connectivity anywhere in the cluster
- A full power cycle restores service, sometimes for days

The giveaway is that the *whole cluster* goes down at once. A single VM losing the network
is a guest problem; every guest losing it simultaneously points at the host or the switch.

## Cause

The I219-LM (Tiger Lake, 11th gen) has firmware bugs around power management. Three
separate features trigger link instability:

- **EEE** (Energy Efficient Ethernet)
- **ASPM** (Active State Power Management)
- **Wake-on-LAN**

There is also a distinct failure underneath them: the e1000e driver reports
*"Detected Hardware Unit Hang"* — a TX unit stall caused by **packet offloads**. This is
why fixing power settings alone is not enough.

## Fix

Deployed via `/etc/rc.local`:

```bash
#!/bin/bash
# Disable ASPM for the I219-LM (Intel 11th gen Tiger Lake)
setpci -s 00:1f.6 0x80.B=0x00
ethtool --set-eee nic0 eee off
ethtool -s nic0 wol d
echo disabled > /sys/class/net/nic0/device/power/wakeup
# Disable offloads — fixes the e1000e "Detected Hardware Unit Hang"
ethtool -K nic0 tso off gso off gro off
exit 0
```

> **The interface here is `nic0`, not `eno1`.** Verify with `ip -br link` before
> assuming a predictable name — Proxmox names interfaces by PCI topology.

## What did not work

**Post-up hooks in `/etc/network/interfaces`.** The natural place for persistent NIC
settings, and it silently did nothing:

```
# this stanza was never applied
auto nic0
iface nic0 inet manual
    post-up /sbin/ethtool --set-eee nic0 eee off
```

The lines sat outside any stanza, so ifupdown ignored them entirely — no error, no
warning, settings simply never applied. Removed once discovered.

## Why this is documented in detail

Because **the first fix did not hold.** EEE, ASPM and WoL were all disabled, and the
host still hung twice more — at 14:50 and again at 00:41 the following day. Disabling
TSO/GSO/GRO is what actually resolved it.

A write-up that stops at the first configuration that appears to work leaves the next
person — or the same person, later — to rediscover that it wasn't enough.

## Recovery while the host is down

```bash
# physical: unplug and replug the Ethernet cable — proven to work
modprobe -r e1000e && modprobe e1000e     # or reload the driver
# if neither works, full power cycle
```

## Verify

```bash
ip -br link                       # interface present?
ethtool nic0                      # confirm 'Energy Efficient Ethernet: off'
cat /sys/class/net/nic0/device/power/wakeup   # should print 'disabled'
ethtool -k nic0 | grep -E 'tx|gso|gro'        # should show off
```

## Related

See also [i915 crash during VFIO handoff](vfio-i915-hang.md) — a different fault that
presents almost identically from outside.
