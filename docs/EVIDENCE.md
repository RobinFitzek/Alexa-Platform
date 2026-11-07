# Evidenz: Wie die Suche beim Echo Dot 3 funktioniert

Stand: 2026-11-06. Jede Aussage ist nach ihrer Belegstärke einsortiert: direkt an der Firmware
geprüft, von Dritten am selben Gerät berichtet, oder nur aus Sekundärquellen übernommen.

## Untersuchte Firmware

- Datei: `update-kindle-donut_puffin-NS65741_user_8138_0013222529668.bin`, Fire OS 6574.1 (NS65741/8138)
- Bezug: [FTVDB](https://ftvdb.com/echo/firmware/com.amazon.donut.android.os/f7cd5f3b6d6bfa59e5ba6a6779285c34-13222529668-fire-os-6574-1-ns65741-8138-2026-08-31/)
- sha256 (von echo-dot-assist gepinnt):
  `ac22b78cf94c2ebacfa90447b770b803d9f72f35858d5218975d97fea95a0245`
- `payload.bin`: 112.635.490 B, `FILE_HASH=28tsCZ6Su3ftrkQG1WrirIsFuffv8z+jOwddiBGTUb4=`
  (SHA-256 in Base64, nachgerechnet, stimmt mit `payload_properties.txt` überein)

Die Firmware liegt bewusst nicht im Projekt (Amazon-Code, 112 MB). Der sha256 der ganzen `.bin` wurde
bei der ersten Analyse gegen den gepinnten Wert geprüft. Bei der Nachprüfung lag nur noch der
entpackte Inhalt vor, geprüft ist dort `payload.bin` per `FILE_HASH`.

## Nachprüfen

```bash
mkdir ota && cd ota && unzip ../update-kindle-donut_puffin-NS65741_user_8138_0013222529668.bin
# payload.bin gegen FILE_HASH aus payload_properties.txt prüfen (stdlib only):
python3 - <<'PY'
import hashlib, base64, re
data = open('payload.bin','rb').read()
want = re.search(r'FILE_HASH=(\S+)', open('payload_properties.txt').read()).group(1)
print('payload hash ok:', base64.b64encode(hashlib.sha256(data).digest()).decode() == want)
PY
debugfs -R 'ls -p /system/bin' extract/system.img
debugfs -R 'dump /system/bin/PuffinApp PuffinApp' extract/system.img
readelf -d PuffinApp | grep NEEDED
strings PuffinApp | grep -E 'HTTP2Transport|audio/x-l16|OPUS'
```

## 1. Direkt an der Firmware geprüft

**Payload.** `CrAU` Version 2, Blockgröße 4096. 523 Operationen: 133 `REPLACE_XZ`, 385 `REPLACE_BZ`,
5 `REPLACE`. Alle 523 Daten-Hashes stimmen. Die hohe Entropie der Blöcke kommt von der Kompression;
dekomprimiert hat ein `system`-Block Entropie 1,0. Nur `tee1`/`tee2` bleiben bei 8,0.

| Partition | Größe in Byte |
|---|---|
| system (ext4) | 1.073.741.824 |
| boot | 9.330.688 |
| tee1, tee2 | je 2.363.392 |
| lk | 212.992 |
| preloader | 151.552 |

**`build.prop`:** `ro.product.device=donut_puffin`, `ro.product.model=AEODN`,
`ro.build.version.release=7.1.2`, `ro.board.platform=mt8167`, `ro.build.ab_update=true`.
Der Fingerprint in `ota.prop` nennt `donut_puffin:6.0`.

**Cloud-Client.** `/system/bin/PuffinApp` nutzt das AVS Device SDK (Namensraum `alexaClientSDK`).
Gelinkt sind unter anderem `libACL.so`, `libAVSCommon.so`, `libcurl_nghttp2.so`, `libpryon.so`,
`libPryonDetector.so`, `libmixerAPI.so`. Der Transport ist HTTP/2 (`HTTP2TransportFactory`,
`LibcurlHTTP2ConnectionFactory`). Die Fähigkeiten `SpeechRecognizer`/`Recognize` und
`SpeechSynthesizer`/`Speak` sind als Strings vorhanden.

**Audio-Uplink.** Strings nennen `audio/x-l16preamble; rate=16000` und `OPUSAudioEncoding`; dazu
`libacsdkOpusAudioEncoder.so`, `libopus.so`. Welches Format im Betrieb gewählt wird, zeigt die
statische Analyse nicht.

**Nicht vorhanden** in `/system/bin` und `/system/lib`: `alexad`, `libalexad.so`, `alexaspeechplayer`,
`audioencoderd`, SPDY-Bibliotheken und GStreamer-Bibliotheken (`libgst*`).

**Lokale Ausführung.** `AlexaHybridExecutionControllerLitespeed` und `/system/lib/alexahybrid/` sind
vorhanden. Die lokalen Skills dort heißen `alerts`, `chrono`, `stop_fallback`, `volume_control`.
Ein Such-Skill ist nicht darunter. Das stützt „keine lokale Suche“, ist aber kein vollständiger
Beweis, denn das Binary wurde nicht im Detail analysiert. Sicher ist: Ganz so dünn ist der Client
nicht, Wecker und Lautstärke lassen sich lokal bedienen.

## 2. Von Dritten am selben Gerät berichtet

echo-dot-assist (`docs/FINDINGS.md`, `devices/donut/README.md`, Build D9N29T/NS65741):

- `PuffinApp` hält `micAsr` und führt das Wake-Word über `libpryon.so` aus
- Audio-Frontend `mixer`/`libasp.so`, `micAsr` als verarbeiteter Mikrofonstrom

## 3. Nur aus Sekundärquellen, nicht an der Firmware belegt

- Cloud-Kette ASR → NLU → Fulfillment → TTS: AVS-Doku; passt zu Abschnitt 1, läuft aber auf
  Amazons Servern und ist am Gerät nicht sichtbar
- „Web-Suche über Bing“: smarthomeinsider.co.uk; aktueller Stand unbekannt
- „Faktenfragen über Knowledge Graph/Wikipedia“: Sekundärquelle
- `AMAZON.SearchQuery`: Amazon-Developer-Blog 2018; betrifft Skills, nicht die eingebaute Suche
- Netzwerkverhalten: arXiv:2105.13500 „An Analysis of Amazon Echo's Network Behavior“

## 4. Nur zum Vergleich: Dot 2

[kilworks/echo-dot-rea](https://github.com/kilworks/echo-dot-rea) analysiert den **Dot 2**
(`biscuit`, Modell `AEOBC`). `alexad`, `libalexad.so`, SPDY, `url.alexa`, `/?operation=auth` und
`alexaspeechplayer` stammen von dort und gelten **nicht** für den Dot 3. Der Vergleich zeigt aber
den Umbau: Amazon ist vom eigenen Doppler-Stack (SPDY) zum AVS Device SDK (HTTP/2) gewechselt.
