# State of the Union – Echo Dot 3 (D9N29T) Security Research

Stand: 2026-11-07. Dieses Repo ist reine Research-Dokumentation: was gemessen, bewiesen und
widerlegt wurde. Es enthält keinen ausführbaren Exploit-Code und keine Firmware.

## Bewiesen (mit Evidenz)
- Gerät: D9N29T = donut_puffin, MT8516/MT8167-Familie, Fire OS 6 = **Android 7.1.2 (API 25)**,
  `security_patch=2026-03-01` (aus OTA build.prop NS65741 und NS6575/8349). Auto-OTA bis 2026.
- Boot/OTA nicht remote angreifbar: lk RSA-PSS, dm-verity, RPMB anti-rollback, TLS-pinning,
  per-model signierte OTAs. Kein Downgrade, kein OTA-MITM (cgarrot + AOSP + eigene OTA-Analyse).
- **Live am echten Dot (192.168.1.111)**: Ports 1080/4070/5541/8888/55442/55443/55445 offen –
  identisch zur doebrite-Referenz; socks 0x09, whad 404 + mTLS alert; SSDP/mDNS antworten nicht.
  Measurement: `measurements/2026-11-07-donut-192.168.1.111.json`.
- Binary-Audit aktuelles Image: civetweb **1.6** (sha256 fb5e92cb…, CVE-2025-55763/2020-27304 N/A),
  thrift 0.9.0 (CVE-2019-0205, nur Setup), dnsmasq 2.51 (nur Setup), avahi 0.6.31 (bootet nicht),
  dhcpcd 6.8.2, wpa_supplicant 2.6- (2026 gepatcht), wlan_mt76x8 4.4.22+ **mit PMKID-Clamp im 2026er Image**.
- Setup-Modus (nur nach Reset): oobed :8080 Thrift, CVE-2019-0205 DoS-PoC, getLinkCode-SSRF,
  getDeviceDetails-Leak, connectToAP-Takeover; dnsmasq 2.51 praktisch unreachable. Kein RCE gemessen.
- Funk: FragAttacks/RSN-OOB nur gegen *alte* Treiber; 2026er Image gepatcht. QUIET-IE-DoS real.
  BLE: WhisperJoin CBL-gated, AvA gefixt, BrakTooth-Controller evtl. DoS.
- **Fazit: ohne Öffnen kein remote Root/RCE bekannt.** kamakiri (USB-Testpads) = einziger Root-Pfad.

## Offen / nächste Aktionen
1. Firmware-Version des echten Dots in der Alexa-App ablesen (Gerät → Info) – 8138/8349 haben
   den gepatchten wlan-Treiber (PMKID-Clamp bestätigt), nie-geupdatete 2019er nicht.
2. civetweb-Version: erledigt (1.6, sha256 fb5e92cb…, identisch 8138=8349).
3. Live-Messungen gegen 192.168.1.111 sind in `measurements/` dokumentiert und jederzeit
   mit einfachem Port-Scan/HTTP reproduzierbar.
4. Monitoring: monatlicher Check auf neue remote/no-solder-Exploits (CIRCL CVE-API,
   MediaTek-Bulletins, Key-Repos: LibreEcho #348, echo-dot-assist #112, firebreak, kamakiri).

## Doku-Map
docs/: REMOTE-ATTACK-SURFACE.md · CVE-MATRIX-ANDROID72.md · CURRENT-DONUT-BINARY-AUDIT.md ·
WIFI-AND-BINARY-VERSIONS.md · SETUP-MODE-ATTACKS.md · SAFETY-BRICK-AVOIDANCE.md · EVIDENCE.md ·
DEBLOAT.md · OPEN-SOURCE-STACK.md · WAKEWORD-LLM-INTERFACE.md · measurements/ (JSON-Rohmessungen)

## Freshness-Check 2026-11-07 (Subagent, GitHub API + MediaTek Bulletins + CIRCL CVE)
- **Fazit unverändert:** kein bekannter remote/software-only Root ohne Öffnen; LibreEcho #348
  („no-solder route?“) weiterhin unbeantwortet.
- Neu: **firebreak v1.0.0 (2026-11-06)** automatisiert den kamakiri-donut-Unlock (weiterhin
  physisch: Case öffnen + USB an TM3/4/5 löten) – PRs #121/#126/#131. Issue #112 aktiv: kamakiri
  greift auf kürzlich geupdateten Geräten nicht mehr („restricted on locked hw“) → physischer
  Weg wird enger, Pin bleibt NS65741.
- Keine neuen CVEs für MT8516/MT7658/MT7668/Echo in MediaTek-Bulletins Mär–Nov 2026 oder CIRCL;
  Nov-2026-Modem-RCEs betreffen Echo nicht (kein Mobilfunk).

## Verifikations-Log
- 2026-11-07: Live-Probe gegen 192.168.1.111 stabil (Ports 1080/4070/5541/8888/55442/55443/55445,
  socks 0x09, whad 404, mTLS alert 40, WHALEXA-Cert O=WHALEXA OU=0x2567d9ea) – Rohdaten in
  `measurements/2026-11-07-donut-192.168.1.111.json`. Freshness-Check: keine neuen
  donut-Exploits/CVEs (LibreEcho #348, eda #112 offen; firebreak v1.0.0 vom 2026-11-06; kamakiri-Branch
  seit 2026-08-07 unverändert). Cross-Check gegen cgarrot/echo-dot-3-doebrite-research (doebrite,
  gleiche Fire-OS-6-Generation): identische Port-Surface, gleiches Fazit „kein OTA-Firmware-Wechsel möglich“.
