# Networking

The whole install was physically relocated: old Dutch home (192.168.10.x) → new Boise
home on a UniFi network (2026-11). This doc captures the current topology and the
IOT-side devices. The HA-side cleanup that came with the move (Boise location, purged
history, pruned old-home entries) is covered in [setup.md](setup.md).

## Topology

```
UniFi gateway  "unifi.aperture"  (10.0.0.1; DHCP + DNS for all segments)
├── LAN VLAN   10.0.0.0/24   — phones, laptops, TV/AppleTV, … (this dev Mac now sits here too)
└── IOT VLAN   10.0.1.0/24   — IoT devices, and glados itself (10.0.1.2)
```

- **glados** is on the **IOT** VLAN: wired `en0`, static lease **10.0.1.2**
  (UniFi DHCP reservation on its MAC), gateway 10.0.1.1, DNS 10.0.1.1.
  It was deliberately moved there (2026-11-17) — see *Why glados lives on IOT* below.
- Everything is routed through the UniFi gateway, so all segments can reach each
  other at L3; only *multicast* (mDNS, Govee discovery) is segment-local.

## Name matrix

| Name | Resolves via | From where it works |
|------|--------------|---------------------|
| `glados.aperture` | UniFi DNS (from the static lease hostname) | anywhere on the UniFi network (LAN or IOT) — **the one to use** |
| `glados.local` | Bonjour/mDNS only | same-L2 Apple devices only; *not* served by UniFi DNS, never crosses VLANs |
| `10.0.1.2` | — | anywhere routable |

HA is configured with `internal_url = http://glados.aperture:8123` (name-based, so it
follows the lease automatically) and no external URL.

## Why glados lives on the IOT segment

The IOT devices don't announce anything mDNS-based, so the UniFi mDNS relay
(**Auto** on all segments) can't help them. Instead:

- **Govee** lights speak their own UDP protocol: discovery is a multicast scan to
  `239.255.255.250:4001`, replies arrive unicast on `:4002`. Multicast is L2-only —
  from the LAN VLAN the scans never reached the lights. With glados *on* the IOT
  segment, the `govee_light_local` coordinator's 60 s discovery works normally.
- **Levoit** (vPure) has no local protocol at all — it's vSync **cloud** polling, so
  the VLAN is a non-issue for it.

## IOT devices

| Device | Count | How it's in HA | State |
|--------|-------|----------------|-------|
| Govee lights (`H6010` ×4, `H16B0` ×1) | 5 | `govee_light_local` config entry (discovery; needs **LAN control on** in the Govee app) | ✅ all 5 as `light.*` entities |
| Levoit air purifier | 1 | `vesync` (VeSync cloud, `levoit` is a virtual alias for it) | ⏳ **pending** — needs the Levoit Home account login in the HA UI (Add Integration → search **vSync** — "Levoit" itself doesn't appear because it's a virtual integration with no config flow) |

Notes:

- The `H6010` model is **power-only** in the `govee-local-api` library (brightness/
  color not exposed — upstream library gap, logged as a warning each discovery
  cycle). The `H16B0` gets full light control.
- Lights that are powered off simply don't respond to discovery; the coordinator
  re-discovers every 60 s, so they reappear automatically after power-on.
- If a purifier is ever re-registered in the app under a *new* account, the vSync
  entry will need re-auth (HA will prompt via the normal reauth flow).

## UniFi settings this setup depends on

- Static DHCP leases: **glados → 10.0.1.2** (hostname `glados` → DNS `glados.aperture`);
  the IOT devices also have reservations (10.0.1.x).
- **mDNS proxy: Auto** on all segments — relays HomeKit/AirPlay/Matter-style mDNS
  across VLANs (not needed for the Govee/Levoit devices, which don't use mDNS, but
  needed for Apple/Chromecast-style discovery).
- DNS server enabled (UniFi is the DNS for all clients).

## UniFi gateway flakiness

Between 2026-11-14 and 11-18 the gateway dropped the **entire** 10.0.x network for
minutes at a time (5 occurrences; glados went dark, came back without rebooting —
it was the UniFi side, not the Mac). Each event breaks HA's network-dependent bits
(websocket clients, cloud integrations) until they reconnect. If HA appears "dead"
after one of these, first check the gateway — and if it recurs, it's worth looking
at UniFi controller logs / gateway health for whatever is causing the flaps.

## This dev Mac

The Mac used to drive these changes has moved around: it was on a separate
192.168.2.x network (why `glados.local` never worked for it), and now sits on the
UniFi **LAN** VLAN (10.0.0.x). From there, `glados.aperture` is the working name;
`glados.local` won't resolve (different VLAN, mDNS is L2-only).