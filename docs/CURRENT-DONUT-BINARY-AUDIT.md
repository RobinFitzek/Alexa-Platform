# Binary-Audit: aktuelles donut-Image NS6575/8349 (Fire OS 6.5.7.5) + BLE

Eigene Extraktion aus der offiziellen OTA `update-kindle-donut-NS6575-8349` (CloudFront, FTVDB
2026-10-27): payload.bin → system.img (ext4, 1 GB) → debugfs. Build: Android **7.1.2, API 25**,
`security_patch=2026-03-01`, `ro.product.device=donut_puffin`.

## Gemessene Binary-Versionen (sha256 verifiziert)
| Datei | Befund | Konsequenz |
|---|---|---|
| `/system/lib/libcivetweb.so` | **75.832 B, sha256 `fb5e92cb…`, Version-String `"1.6"` @ offset 59631** – byte-identisch donut/crumpet/doebrite 2023→2026; `mg_version()`=1.6 (Subagent via Disasm) | **CVE-2025-55763 (1.14–1.16) NOT applicable**; CVE-2020-27304 (1.8–1.14) N/A; civetweb 1.6 von 2014, aber cgarrot-Fuzzing robust, whad-Routen 404, mTLS zu |
| `libthrift.so` | 370.256 B, **0.9.0** | CVE-2019-0205 offen – nur Setup-AP :8080 (oobed) |
| `dnsmasq` | **2.51** | CVE-2017-14491ff. nur Setup-AP, praktisch unreachable |
| `avahi-daemon` | **0.6.31** | bootet im Normalmodus nicht (rlimit) |
| `dhcpcd` | 6.8.2 | CVE-2016-1503 in AOSP-Version gefixt |
| `wpa_supplicant` | v2.6- (AOSP) + Backports bis patch 2026-03 | KRACK/FragAttacks-supplicant/Dragonblood gepatcht |
| `/system/bin/socks` | 34.380 B, sha256 `01285a14…`; REA elf-notes: elf32 ARM; strings: `/dev/socket/socks_sup`, „whitelist is disabled…“, ARP-MAC | :1080 latent-only (live bestätigt: 0x09) |
| `oobed` | 521.948 B | Setup :8080 |
| `/system/lib/modules/wlan_mt76x8_sdio.ko` | 2,4 MB, **vermagic 4.4.22+ aarch64**, Marker „Support maximum PMKID Cnt“ **vorhanden** in NS6575/8349 | 2020er RSN/PMKID-Clamps kamen per OTA aufs Gerät – nur wer nie updatet, fährt den ungepatchten 2019er Treiber |

## BLE (Subagent ble-whisperjoin-ava)
- WhisperJoin (GATT FE00, 17 Kommandos): durch **CBL-Cloud-Token** gedeckt (status 5), kein Bypass.
- AvA **CVE-2022-25809**: in NS65741/NS6575 gefixt (Amazon patched 2022-04; Fire OS 6 bekommt Amazon-seitige Updates unabhängig vom AOSP-Patchlevel – FTVDB zeigt OTAs bis Okt 2026).
- BrakTooth CVE-2022-20021/22/23 (MT7658 Controller): Controller-FW 2019 ungepatcht (kein Evidence für Amazon-Update) → BT-DoS in Funkreichweite möglich.

## Live-Messung am echten Gerät
`docs/measurements/2026-11-07-donut-192.168.1.111.json` – Port-Set matcht doebrite-Referenz exakt;
Abweichung: :4070 TCP offen, HTTP-GET hängt (kein getInfo-Leak live sichtbar auf diesem Build).

## Nachtrag: NS65741/8138 (kamakiri-Pin-Build) entpackt
Mit `cyxx/extract_android_ota_payload` (payload.bin von OTA 8138, sha256 ac22b78c…) extrahiert:
system.img (1 GB ext4), boot.img (Android bootimg, cmdline `bootopt=64S3,32N2,64N2 …`),
lk.img, tee1/tee2, preloader.img. Ergebnis: **libcivetweb.so sha256 `fb5e92cb…` = identisch mit
NS6575/8349** (civetweb 1.6), libthrift 0.9.0, build.prop patch 2026-03-01. → Alle donut-Builds
tragen dieselben alten Third-Party-Libs; die CVE-Bewertung oben gilt für 8138 und 8349 gleichermaßen.

## Boot-Kette aus NS65741/8138 (extrahierte Images, RE per strings)
- `preloader.img` (151.552 B): Magic `EMMC_BOO`, USBDL-Strings (`usbdl_send_da`, „DA validation
  disabled“, download.c) = der MediaTek-USB-Download-Mode, den **kamakiri-donut** ausnutzt.
- `lk.img` (212.992 B): Magic `0x58881688` (MTK-LK), enthält `amzn_verify_unlock`,
  „Failed to get unlock key/code“, temp-unlock via idme, `pl_build_desc`/`lk_build_desc`-Keys
  (die getvar-Werte aus echo-dot-assist #112), `[FASTBOOT] mode`, „Force brom download recovery success“.
- `tee1/tee2.img`: Magic `0x58881688` (TrustZone).
- `boot.img`: `ANDROID!`, Kernel dekomprimiert: **Linux 4.4.22+ (gcc 4.9, build 2026-08-31)**,
  cmdline `bootopt=64S3,32N2,64N2 … veritykeyid=id:f3530e18f64d11fc25eb2dd762979f078de990bf`
  → dm-verity-Schlüssel-ID direkt in den Bootargs.

## PuffinApp (8138) + Live-4070 Nachtrag
- `/system/bin/PuffinApp` (6,7 MB, elf32 ARM; REA inspect-elf-notes OK): baked AVS-Endpunkte
  `https://avs-alexa-14-na.amazon.com`, `api.amazonalexa.com`/`api.eu.amazonalexa.com` (+integ/gamma),
  alexaClientSDK-Symbole (avsCommon logger/metrics) → PuffinApp = AVS-Device-SDK-Client, Suche/ASR/NLU
  komplett cloud-seitig (schließt die Kreis zur EVIDENCE.md).
- Live :4070 auf 192.168.1.111: `getInfo` (HTTP/1.0) → 200 mit deviceID/publicKey/version 2.9.0;
  `addUser` (minimal + voller Param-Satz) → **404** auf diesem donut-Build (doebrite: 500) → nur
  getInfo-Leak exponiert. Measurement: `measurements/2026-11-07-donut-4070-adduser.json`.

## wlan-Treiber 8138 vs 8349 (sha256-Vergleich)
- `wlan_mt76x8_sdio.ko`: 8138 = sha256 `a0cf8069…` (2.377.064 B), 8349 = `1749fad4…` (2.378.536 B),
  beide vermagic `4.4.22+ aarch64`, beide mit Clamp-Marker „Support maximum PMKID Cnt“.
- → Schon die kamakiri-Pin-Version NS65741/8138 trägt die 2020er RSN/PMKID-Clamps; der ungepatchte
  „2019er Factory-Treiber“ betrifft nur nie-geupdatete Geräte. FragAttacks/RSN-OOB auf deinem
  (auto-geupdateten) Dot praktisch erledigt.

## Image-Diff NS65741/8138 vs NS6575/8349
- /system/bin: 728 Binaries in beiden; 8349 +`LocalAutomationExecutionD` (Local-Automation-Engine),
  8138-only `smarthome_utility_puffin`.
- libcivetweb.so + libthrift.so: **sha256-identisch** in beiden Builds (fb5e92cb… / 6908506a…)
  → Third-Party-Libs werden nie angefasst, egal wie neu der Build.

## OTA-Signatur kryptografisch verifiziert (8138)
- `META-INF/MANIFEST.MF` (SignApk): `SHA-256-Digest: 28tsCZ6Su3ftrkQG1WrirIsFuffv8z+jOwddiBGTUb4=`
  = base64(sha256(payload.bin)) = `dbcb6c099e92bb77edae4406d56ae2ac8b05b9f7eff33fa33b075d88119351be` –
  **matcht exakt** mit unserer berechneten payload.bin-sha256 → ZIP-Signatur deckt payload.bin voll ab.
- Signatur-Zertifikat (META-INF/CERT.RSA, PKCS#7): `CN=Amazon, OU=Amazon, kindle-cs-support@amazon.com`.
  → OTA-Manipulation ohne Amazons privaten Schlüssel unmöglich (Basis für „kein remote OTA-Angriff“).

## TEE + Gesamt-Check
- `tee1/tee2.img` (2.363.392 B, Magic 0x58881688): strings liefern kaum Klartext (TEE meist
  opaque/komprimiert) – für tiefes TEE-RE braucht es Ghidra/IDA auf dem ARM32-Image.
- Verifikation der Messungen und Live-Probes: `docs/measurements/` + `STATE-OF-THE-UNION.md`.
