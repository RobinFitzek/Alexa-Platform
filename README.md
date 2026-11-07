# Echo Dot 3 (2018) — `donut` — Software-Only Security Research

A complete, reproducible, evidence-based security assessment of the Amazon Echo Dot 3rd Gen
(model **D9N29T**, codename `donut` / `donut_puffin`), performed over the air — LAN, Wi-Fi,
BLE and setup-mode HTTP only. No hardware modification, no opening, no UART, no test pads, no USB.

The repository is **research only**: it contains measurements, analysis and documentation.
It deliberately contains no exploit code, no firmware dumps, no keys and nothing that could
interfere with a device you do not own.

## TL;DR

| | |
|---|---|
| **Device** | Amazon Echo Dot 3rd Gen, model D9N29T |
| **Codename** | `donut` (`donut_puffin`), MediaTek MT8516/MT8167 family, WiFi/BT MT7658 (mt76x8) |
| **OS** | Fire OS 6 = Android 7.1.2 (API 25), AOSP security patch 2026-03-01, kernel 4.4.22 |
| **Firmware tested** | `update-kindle-donut_puffin-NS65741_user_8138_…bin` and NS6575/8349, sha256 pinned |
| **LAN surface (live measured)** | TCP 1080, 4070, 5541, 8888, 55442, 55443, 55445 |
| **Outcome** | 1 pre-auth DoS (setup mode, CVE-2019-0205), 1 SSRF, 1 info leak, 1 latent root-proxy — **no remote RCE, no root, no firmware install over the air** |

**Headline result:** the boot chain is cryptographic end to end — RSA-PSS in `lk`, dm-verity on
`/system`, RPMB anti-rollback, per-model signed OTAs over pinned TLS. Even a root shell would not
allow replacing the OS. The only known unlock is the physical kamakiri bootrom exploit (USB test
pads TM3/TM4/TM5, requires opening the case) and only on the pinned firmware NS65741. Without
opening the device there is **no known remote or software-only root** — verified against CVE
databases, MediaTek bulletins and the public research repos as of 2026-11-07, and cross-confirmed
by an independent audit of the sibling device `doebrite` (cgarrot/echo-dot-3-doebrite-research).

## Findings

| # | Finding | Pre-auth | Evidence |
|---|---|---|---|
| 1 | Pre-auth DoS in `oobed` (setup AP, :8080) — Apache Thrift 0.9.0 JSON recursion, CVE-2019-0205 | yes (setup mode) | `docs/SETUP-MODE-ATTACKS.md` |
| 2 | SSRF in OOBE `getLinkCode` — attacker-controlled host (DNS) and port; proven as loopback oracle | yes (setup mode) | `docs/SETUP-MODE-ATTACKS.md` |
| 3 | Spotify Connect `getInfo` on :4070 leaks `deviceID`, keys and owner alias, unauthenticated | yes (LAN) | `docs/measurements/2026-11-07-donut-4070-adduser.json` |
| 4 | `/system/bin/socks` on :1080 — root SOCKS5 that builds `iptables` command lines via `system()`, authorized by spoofable ARP-MAC + deviceID whitelist; every CONNECT refused with `0x09` while whitelist is empty | latent | `docs/REMOTE-ATTACK-SURFACE.md`, measurements |
| 5 | `whad` :55442 answers 404 on all routes; :55443/:55445 require mTLS (WHALEXA self-signed cert) | closed | measurements |
| 6 | Bundled libraries ancient and never bumped: civetweb 1.6, thrift 0.9.0, dnsmasq 2.51, avahi 0.6.31, dhcpcd 6.8.2, wpa_supplicant 2.6- | — | `docs/CURRENT-DONUT-BINARY-AUDIT.md` (sha256 pinned) |
| 7 | WiFi driver mt76x8: 2026 image (8138/8349) contains the 2020 PMKID/RSN clamps; never-updated 2019 units likely vulnerable to FragAttacks-class issues | depends on firmware | `docs/WIFI-AND-BINARY-VERSIONS.md` |
| 8 | Boot/OTA chain not attackable remotely: no downgrade, no OTA MITM, signature chain verified against payload hash | — | `docs/EVIDENCE.md` |

Full CVE mapping against the real reachable surface: `docs/CVE-MATRIX-ANDROID72.md`.

## Repository layout

```
docs/
  STATE-OF-THE-UNION.md            index: what is proven, what is open, verification log
  REMOTE-ATTACK-SURFACE.md         LAN / setup / BLE / WiFi surface, live results
  CVE-MATRIX-ANDROID72.md          known Android 7.1.2 exploits vs. this device
  CURRENT-DONUT-BINARY-AUDIT.md    binary versions from extracted OTA images, sha256 pinned
  WIFI-AND-BINARY-VERSIONS.md      mt76x8 / wpa_supplicant / FragAttacks assessment
  SETUP-MODE-ATTACKS.md            oobed :8080, CVE-2019-0205, SSRF, dnsmasq 2.51
  EVIDENCE.md                      OTA payload/signature verification, reproducible commands
  DEBLOAT.md                       measured size share of Amazon code in /system
  SAFETY-BRICK-AVOIDANCE.md        kamakiri path, brick risks (echo-dot-assist #112), gates
  OPEN-SOURCE-STACK.md             consensus recommendation for repurposing after unlock
  WAKEWORD-LLM-INTERFACE.md        on-device wake word → own LLM interface
  measurements/                    raw JSON of the live probes against 192.168.1.111 (2026-11-07)
```

## Ethics & disclosure

- All testing was performed on **hardware owned by the researcher**, on the researcher's own network.
- No third-party device, account or network was touched; no device was damaged.
- Findings about other vendors' products (e.g. CVE-2019-0205 in Apache Thrift) belong to the
  respective upstream projects; this repo only documents impact on this device.
- Amazon firmware and libraries are **not redistributed** here; OTA images are obtained from
  Amazon's public CDN and verified by pinned sha256 (`docs/EVIDENCE.md`).

## License

Documentation in this repository: CC BY 4.0. Research is provided for devices you own.
