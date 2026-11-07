# WiFi-Angriffsplan + Binary-Versionen (Echo Dot 3 Familie)

Quellen: Subagent `wifi-fragattacks-plan` (vanhoefm/fragattacks, AOSP bulletins, MediaTek), Subagent
`deepdive-socks-whad-wifi` (cgarrot-Repo), eigene Extraktion der biscuit-Block-OTA 272.5.6.4
(CloudFront 1b9718ec…, SignApk-JAR → system.new.dat → sdat2img → ext4 → debugfs; libcivetweb.so per
REA `inspect-elf-notes` geprüft: elf32 ARM, gold).

## Binary-Versionen (aus lesbarem System-Image, biscuit Fire OS 5.5.6.4 / 2016)
| Komponente | Version | CVE-Relevanz |
|---|---|---|
| dnsmasq | **2.51** (2010) | CVE-2017-14491/92/93 RCE — nur Setup-AP erreichbar |
| avahi-daemon | **0.6.31** (2011) | CVE-2017-6519; bootet im Normalmodus nicht (rlimit) |
| dhcpcd | 5.5.6 (biscuit) / **6.8.2** (doebrite 2026) | CVE-2016-1503 in AOSP 6.8.2 gefixt |
| wpa_supplicant | 2.3-5.1.1 (biscuit) / 2.6-base+backports (NS65741, patch 2026-03) | 2026-Build: supplicant-side gepatcht |
| libthrift | **0.9.0** – auch im 2026-doebrite-Build | CVE-2019-0205 (setup :8080) |
| libcivetweb.so | ~1.11 (biscuit, 153 KB) / 75 KB (doebrite 2026, Version nicht exponiert, fuzz-robust) | CVE-2018-12686 (DEF CON 26) 2018 gefixt; CVE-2025-55763 (1.14–1.16) Status offen |
| kernel | 4.4.22 (doebrite NS65741) | MTK 3.18/4.4 post-2019 LPEs = local only |
| wlan_mt76x8 | **2019er Factory-Binary** (nie upgedatet) | 2020er RSN-Clamps fehlen vermutl. |

## WiFi-Angriffsplan (Radio-Reichweite, kein physisches Öffnen)
Voraussetzung: 2× Alfa AWUS036ACH (Injection) oder ath9k_htc/hwsim; Tool: github.com/vanhoefm/fragattacks.

| Angriff | CVE | Effekt auf konfiguriertem Dot 3 | Precondition |
|---|---|---|---|
| Plaintext-Injection (client akzeptiert unverschlüsselte Frames) | CVE-2020-26140/26142/26143/26145, 24588 | **beliebige plaintext IP-Pakete an den Dot** (gefälschte LAN-Traffic an seine Dienste) – kein Key nötig | evil AP, gleicher Kanal; 2019er mt76x8 pre-fix |
| A-MSDU-als-EAPOL | 26144 | injiziertes EAPOL → supplicant-Confusion/DoS | gleicher Kanal |
| Frag-Cache über Reconnect / mixed-key | 24586/24587 | Injection/Exfil bei Rekey | Rogue-AP deauth / MC-MitM |
| MediaTek RSN-IE OOB write (pmkid_cnt/pairwise_cnt ungeclampt, rsn.c:266/337) | CVE-2020-MTK-Klasse | **pre-auth kernel OOB write → Crash/Reboot (DoS)** | Rogue-AP Beacon/Assoc beim Scannen |
| QUIET IE 802.11h | – | **DoS: Dot verstummt 67 s+** (bestätigt) | Rogue-AP Beacon |
| WPS | – | **keine WPS-Fläche** (OOBE nur NONE/WPA/WPA2) | – |
| Supplicant-CVEs (KRACK, Dragonblood, FragAttacks-supplicant, P2P CVE-2021-27803) | div. | im 2026-03-Build gepatcht (AOSP-Komponente) | – |

Repro-Kommandos (fragattacks): `./fragattack.py wlan0 --ap --inject wlan1 'ping I,P'` (26140),
`'ping I,P,P'` (26143), `'ping D,BP --bcast-ra'` (26145), `'amsdu-inject'` (24588),
`'ping I,E,R,AE' [--full-recon]` (24586), `'ping I,F,BE,AE --pn-per-qos' [--rekey-req]` (24587, MC-MitM).
`TEST COMPLETED SUCCESSFULLY` = vulnerable. Verifikation ohne Root-Shell am Dot nur über Side-Effekte
(DHCP/ICMP/ARP zur Dot-IP, Alexa-Erreichbarkeit). Binary-Vintage-Check on-device (nach Root):
`strings wlan_mt76x8 | grep -i "PMKID Cnt"` – fehlt = 2019er Binary.

**Fazit WiFi:** realistisch Funk-only: plaintext injection toward client + pre-auth RSN-OOB-DoS +
Quiet-IE-DoS. Kein Key-Recovery, kein RCE. Supplicant 2026 gepatcht, Vendor-Driver nicht.
