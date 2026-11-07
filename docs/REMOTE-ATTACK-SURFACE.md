# Remote-Angriffsfläche des Echo Dot 3 (D9N29T / donut_puffin) – Gesamtergebnis

Recherche via 4 Subagenten + REA (rea@6.8.0, `~/Documents/Git/rea`) + eigene OTA-Analyse.
Ziel: Software-only/Remote-Weg, um eigene Software auf den Dot zu bringen **ohne Öffnen**.

## Verdict (einstimmig, evidenzbasiert)

> **Es gibt derzeit KEINEN bekannten Netzwerk-/BLE-/WLAN-Root oder RCE für den D9N29T
> ohne physischen Eingriff.** Die Boot-Kette ist kryptografisch versiegelt:
> `lk` verifiziert boot/recovery per **RSA-PSS (Amazon-Key)**, `/system` läuft unter
> **dm-verity**, Anti-Rollback für preloader/lk/tee liegt in **RPMB** (firebreak: `rpmb_state 1`).
> OTA: TLS-gepinntes Manifest (kein HTTP-Fallback auf Fire OS 6), per-Modell signiertes
> SignApk-ZIP (otacerts), signiertes payload.bin-Manifest gegen on-device RSA-Key
> (server-supplied keys auf Android deaktiviert, `update_attempter_android.cc`).
> Selbst root würde das OS nicht wechseln können („Root gives the house keys, not the safe's" –
> cgarrot, docs/10). **Einziger bekannter Root-Pfad: kamakiri-donut über USB-Testpads
> (TM3/TM4/TM5) = Gerät öffnen.**

## Angriffsfläche im konfigurierten Zustand (normales WLAN)

| Port | Dienst | Auth | Befund |
|---|---|---|---|
| 4070 | Spotify Connect | **keine** | Info-Leak: `GET /spotifyConnect?action=getInfo` liefert deviceID, publicKey, **Spotify-Anzeigename des Besitzers** (cgarrot Finding #5) |
| 1080 | `/system/bin/socks` (SOCKS5, **root**, an p2p0) | ARP-MAC-Whitelist (spoofbar) | whitelist leer → alle CONNECT abgelehnt (0x09); latentes `system()`+iptables-Primitiv, nicht ausgenutzt |
| 55442 | whad HTTP (Audio-Cache) | – | 34 Routen probed, alle 404; DEF-CON-26-RCE-Kette (CVE-2018-12686 civetweb + CVE-2017-1000254 libcurl) greift nur bei 1./2. Gen ≤ 608490720 |
| 55443/55445 | whad WHALEXA HTTPS | mTLS (Amazon private CA) | foreign certs → `alert 48 unknown_ca`; keine unauth Local API |
| 5541 | Matter | PASE/CASE | gefuzzt, robust |
| 8888 | – | – | RST/closed |
| 40317, 5353, 1900, 16384–32767/udp | TPH, mDNS, UPnP, SIP/RTP | – | gefuzzt, „measured closed" |

Quellen: cgarrot/echo-dot-3-doebrite-research docs/02+07+08 (doebrite = gleiche Fire-OS-6-/puffin-Generation), DEF CON 26 "Breaking Smart Speakers", arXiv:2105.13500, jhautry/echo-dot port-scans.

## Setup-Modus (nur nach Factory Reset / langer Action-Taste)
- **8080 `oobed`** (Thrift-JSON `/OOBE`, Apache Thrift **0.9.0**): pre-auth **DoS CVE-2019-0205** (eine 280-KB-Request killt oneshot-Service bis Power-Cycle) + **SSRF** in `getLinkCode` (DNS-basierter Host, client-gesteuerter Port → Loopback-Port-Oracle) + Info-Leak `getDeviceDetails` (DSN, MAC, ephemeral setup cert) + `connectToAP` = Provisioning auf Angreifer-AP. Alles **ohne Code Execution**.
- **WhisperJoin BLE** (GATT FE00): Protokoll komplett reverset (17 Kommandos), aber alle nützlichen → `UNAUTHORIZED_PROVISIONER` (CBL-Token von Amazons Cloud). Kein Bypass bekannt.

## Funk/WiFi/BT (Radio-Reichweite)
- **CVE-2022-25809 (AvA)**: 3./4. Gen Echo Dot – arbitrary voice commands via bösem Skill (remote) oder gepaartem BT-Gerät (BLE-Reichweite, reconnect jederzeit) → Gerätesteuerung, kein Root. Gepatcht 2022-04.
- **CVE-2023-33248**: 16–22 kHz Audio → Voice Commands (DolphinAttack).
- **BrakTooth** CVE-2022-20021/22/23 (MediaTek BT Link Manager, DoS) – 2019er Firmware evtl. ungepatcht.
- **FragAttacks** (CVE-2020-26140ff.): mt76x8-Treiber von 2019 hat 2020er Fixes wahrscheinlich nicht → Plaintext-Injection bei evil-AP. KRACK: nicht betroffen (supplicant gepatcht). BlueBorne: gepatcht. SweynTooth: MediaTek nicht betroffen.
- **GhostLock CVE-2026-43499** (Kernel-UAF, lokaler Root) – braucht schon Code Exec auf dem Gerät, kein donut-Port, kein remote Trigger.

## OTA/Update-Kette (LAN-MITM / bösartiger Update-Server / Downgrade)
- Nicht viable: TLS-Pinning (Starfield G2 gepinnt, rogue CA → alert 42/46/48; SSLsplit schlägt fehl), kein HTTP-Fallback auf Fire OS 6, per-model SignApk + payload-Manifest-Signatur (on-device Key, Server-Key-Override deaktiviert), per-op SHA-256 + partition hash, lk RSA-PSS + dm-verity + RPMB Anti-Rollback.
- `max_timestamp` existiert in update_engine erst ab Android 8.1 – auf Fire OS 6 schützt nur Pinning+Signatur+Bootloader; Downgrade-Remote trotzdem nicht möglich (signierte echte Zips nur via TLS; RPMB blockt pl/lk/tee-Downgrade).
- Theoretisch: Recovery-Zip-Parser-Bug-Klasse (CVE-2017-0475, gefixt seit 2017-03) – auf NS65741 (2019) sehr wahrscheinlich gepatcht.
- Realistischer Remote-Effekt: **OTA-DoS** (Updates blocken) – genau das, was die Router-Sperre macht.

## Was das für „own your hardware" ohne Öffnen bedeutet
1. Eigene Firmware/Root ohne Öffnen: **nicht möglich** (oben belegt).
2. Maximal ohne Öffnen: kein eigener Code auf dem Dot möglich; Amazon-Verkehr am Router blocken (MAC, siehe unten) + DNS-Sinkhole; Steuerung läuft extern (z. B. Home Assistant `alexa_media_player`, Bluetooth/AUX/DLNA) – Optionen in `OPEN-SOURCE-STACK.md`.
3. Volle lokale Kontrolle (echo-dot-assist/techo5): nur mit Öffnen + kamakiri → `SAFETY-BRICK-AVOIDANCE.md`.

## CVE-Matrix Android 7.1.2
Vollständige Matrix (welche bekannten Exploits der Android-7.1.2-Ära auf den Dot zutreffen):
**[CVE-MATRIX-ANDROID72.md](CVE-MATRIX-ANDROID72.md)**. Kern: `security_patch=2026-03-01` deckt nur
AOSP ab; Amazon-Daemons + Vendor-Treiber (dnsmasq 2.51, avahi 0.6.31, Thrift 0.9.0, mt76x8,
MT7658-Controller) bleiben verwundbar. Remote-Root auf konfiguriertem Gerät: keiner bekannt;
kamakiri bleibt physischer Fallback.

## Remote-Messungen (ohne Root, vom LAN)
Live am echten Dot (192.168.1.111) gemessen und als JSON abgelegt unter `measurements/`
(2026-11-07): Ports 1080/4070/5541/8888/55442/55443/55445 offen, SOCKS :1080 antwortet `0x09`
(Policy-Refusal), Spotify-getInfo-Leak auf :4070, whad :55442 → 404, :55443/:55445 → mTLS alert 40/48,
SSDP/mDNS antworten nicht. Fuzz von :4070/:55442 (10 Klassen malformed HTTP): kein Crash.

## Router-Block (TP-Link Archer VR2100v) – MAC des Dot
Aus der Client-Liste eindeutig identifiziert (IEEE-OUI-Abgleich, 205 Amazon-OUIs):
**`3C-5C-C4-B7-17-DE`** (IP 192.168.1.111, „Unknown“) = Amazon Technologies Inc. = dein Echo Dot.
→ Advanced → Access Control → Blacklist → diese MAC (Website-Liste leer = komplettes Internet aus; LAN bleibt). DNS-Level ergänzen via AdGuard/Pi-hole: `||amazon.*^$client=3C:5C:C4:B7:17:DE`, `||cloudfront.net^…`, `||amazonaws.com^…`.

## Quellen (Auswahl)
- https://github.com/cgarrot/echo-dot-3-doebrite-research (docs/01–10, evidence/) – komplette OTA-Audit der Fire-OS-6-Generation
- https://github.com/aslater3/LibreEcho/issues/348 (donut = #348, „no-solder route?“ offen)
- https://github.com/bboe/firebreak (docs/devices.md donut, anti-rollback/RPMB; ota.py)
- https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-dot-3rd-gen-2018-donut.4801400/ (kamakiri, nur mit Öffnen)
- https://media.defcon.org/DEF%20CON%2026/...Breaking-Smart-Speakers...pdf + tencentbladeteam/Exploit-Amazon-Echo (nur 1./2. Gen)
- NVD: CVE-2019-0205, CVE-2022-25809, CVE-2023-33248, CVE-2017-0475, CVE-2022-20021/22/23, CVE-2020-26140ff, CVE-2026-43499
- AOSP update_engine android-7.1.2_r36 (Signaturpfade, key-override disabled); arXiv:2105.13500; AV-TEST 2017 (Pinning)
- Eigene Analyse: OTA NS65741 sha256-verifiziert, Manifest-Partitionen boot/lk/tee1/tee2/preloader/system, Amazon-Signaturzertifikat
