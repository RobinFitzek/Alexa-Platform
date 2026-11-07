# Debloat: wie viel Amazon-Software/Tracker fliegt vom Dot 3 (D9N29T)?

Gemessen aus dem extrahierten System-Image NS65741/8138 (debugfs, 2026-11-07):
- /system (bin/lib/lib64/app/priv-app/etc/vendor): 1346 Dateien, 181 MB; **Amazon-named: 43 Dateien (3,2 %), 27,4 MB (15,1 %)**
- Größte Alexa-Client-Komponenten in /system/bin: `AlexaHybridExecutionControllerLitespeed` 8,7 MB, `PuffinApp` 6,5 MB, `puffinmrmd`, `smarthome_utility_puffin`, `davsFileCheck.sh`
- priv-app: com.amazon.credentialmanager.neo, diode.neo, factory_reset.neo (+ OOBE/csm-APKs)
- dazu: doppler/avs-Libs, metrics/telemetry (metrics-collector, device metrics), AVS-Endpunkte in PuffinApp (avs-alexa-14-na.amazon.com …)

## Was pro Weg runterfliegt
| Weg | Was entfernt wird | Anteil |
|---|---|---|
| **echo-dot-assist (Fire OS bleibt)** | PuffinApp + gesamter Alexa-Client/Doppler-Stack, Metrics/Telemetry, OOBE; Firewall blockt Amazon-Cloud komplett; Mic/Wake-Word (mixer/libasp/libpryon) bleiben (Hardware nötig) | der komplette Alexa-Cloud-Client + Tracking ≈ die ~15 % Amazon-Anteil am /system-Volumen (27 MB) + 100 % Cloud-Traffic |
| **techo5-dot (Alpine statt Fire OS)** | komplettes Android/Fire-OS-Userspace inkl. ALLER Amazon-Pakete/Daemons | **~100 % des Userspace**; bleibt nur die signierte Boot-Kette (BootROM, preloader, lk, tee) – die ist ohne Amazons RSA-Key nicht änderbar (veritykeyid in bootargs, RPMB) |
| **no-open (kein Unlock)** | nur Traffic-Block (Router/DNS) + eigener Skill | 0 % Software entfernt, aber 100 % Amazon-Cloud-Traffic blockierbar |

Fazit: mit techo5 ≈ 100 % der Software, mit echo-dot-assist der gesamte Alexa-Client+Tracker (~15 % des System-Volumens, 100 % des Cloud-Traffics); die Boot-Kette (lk/tee/preloader) bleibt immer – die ist kryptografisch Amazons.
