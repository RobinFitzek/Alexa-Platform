# Sicherheit: den Echo Dot 3 (D9N29T) nicht unbrauchbar machen

Stand: 2026-11-06. Diese Seite fasst Quellen zusammen und ersetzt sie nicht: **Maßgeblich ist
[`devices/donut/README.md`](https://github.com/Gamer92000/echo-dot-assist/blob/main/devices/donut/README.md)
von echo-dot-assist.** Weicht sie von dieser Seite ab, gilt sie. Für den Unlock selbst nimmt man am
besten `scripts/setup.sh donut`, das vor dem Unlock nachfragt.

> **Warnung (Upstream, issue #112, offen):** Der Unlock kann das Gerät unbrauchbar machen.
> **Kürzlich aktualisierte Echos können betroffen sein, das Ausmaß ist unbekannt.** Ein Dot, der bis
> vor Kurzem online war, gehört also zur Risikogruppe. Unklar ist auch, ob irgendein Unlock derzeit
> sicher ist. Upstream rät: „do not unlock an Echo you want to keep“.

Viele öffentliche Berichte stammen vom Dot 2 (`biscuit`), dessen Unlock anders läuft. Die Spalte
„Gilt für“ zeigt deshalb, was wirklich am Dot 3 (`donut`) beobachtet wurde.

## Die gefährlichsten Fehler

| Fehler | Folge | Gilt für | Quelle |
|---|---|---|---|
| Gerät geht **online zu Amazon**, bevor der Unlock fertig ist | Es aktualisiert sich selbst; ein Update kann die genutzte Lücke schließen | Dot 3 | echo-dot-assist `devices/donut/README.md` |
| kamakiri läuft fehlerfrei, das Gerät kommt aber **locked** zurück (grüner Ring, `unlock_status false`, `restricted on locked hw`) | Bisher kein Weg gefunden, den Unlock zu beenden oder rückgängig zu machen; 2 Geräte betroffen, Umfang unbekannt. Im ersten Fall fehlte `kaeru` beim Zurücklesen in `expdb`; dieses Gerät war schon vorher defekt | Dot 3 | echo-dot-assist #112 |
| Falsches Modell (C78MP8 Refresh `crumpet`, 36EBT3 mit Uhr `doebrite`) | Nicht unterstützt | Dot 3 | echo-dot-assist README, `devices/donut/README.md` |
| Falsche Firmware-Datei | Das Setup nimmt nur die exakte Datei; eine umbenannte andere Version wird abgelehnt | Dot 3 | echo-dot-assist #62 |
| Zweite Installation im selben Slot, weil der Slot **von Hand** gewechselt wurde | Der andere Slot behält Amazons neuere Firmware | Dot 3 | `devices/donut/README.md` |
| USB über VM, WSL oder macOS | Timing des Bootrom zu knapp: Abbruch oder Antwort zu spät | Dot 3 | `devices/donut/README.md` |
| Nach dem Flash ins Netz ohne Lockdown | Amazon-OTA überschreibt das Gerät, es startet ins Amazon-Setup | Dot 3 | techo5-dot #37 (offen) |
| LK (Bootloader) neuer, als der Exploit kennt | Der Unlock wird abgelehnt, nichts wird geschrieben. Ob das so auch für kamakiri-donut gilt, ist offen | **Dot 2** | firebreak #114 |
| Bootloop nach dem Flash | Drehender blauer oder türkiser Ring | **Dot 2** (#8, #19), Gerät in #1 nicht genannt | techo5-dot #1, #8, #19 |

**Offene Frage: Ist ein Brick beim Dot 3 endgültig?** Es gibt den Bericht, der Dot 3 habe kein
Bootrom-USBDL, ein Brick wäre dann permanent. Firebreak hält das so nicht für zutreffend, weil
kamakiri-donut den Bootrom über `0e8d:0003` erreicht. Die Sicherung wurde an einem donut aber nie
ausgelesen (`docs/unlocking.md`). Man sollte also nicht mit einer Rettung rechnen.

## Checkliste

Die Reihenfolge ist wichtig. Die Befehle stehen in `devices/donut/README.md`.

### A. Vor dem Öffnen

- [ ] Restrisiko akzeptiert (Warnung oben), das Gerät darf verloren gehen
- [ ] Modell auf der Unterseite ist **D9N29T** (nicht C78MP8, nicht 36EBT3)
- [ ] Gerät stromlos lassen, **nicht** per Alexa-App einrichten, Internet am Router sperren
- [ ] Linux direkt auf dem PC (Live-USB-Stick reicht), kein macOS, keine VM, kein WSL
- [ ] Dateien liegen in `firmware/donut/` und stimmen per `sha256sum`:
  - `update-kindle-donut_puffin-NS65741_user_8138_0013222529668.bin`:
    `ac22b78cf94c2ebacfa90447b770b803d9f72f35858d5218975d97fea95a0245`
  - `kamakiri-donut-v1.0.0.zip`:
    `4d2bb52eaf661f6616aa4268584d44cf9155c1328c34bfcba01a6558f942b738`
  - `boot-root.zip`:
    `de49cc88b27a8e77cf97cf0156bee50e4ddc0e116c41aaede06b494e38397be0`

Fire OS 6574.1 ist die Datei, die **geflasht** wird, keine Voraussetzung am Gerät: Sie ersetzt beim
Unlock, was vorher lief, auch eine neuere Version. Ein gesperrter Dot lässt sich vorab ohnehin nicht
per `adb` prüfen.

### B. Verkabeln und Unlock

- [ ] USB-Kabel mit **USB-A**-Stecker am PC (bei USB-C fehlt der 5,1-kΩ-Widerstand am CC-Draht, der PC sieht dann nichts)
- [ ] Löten: GND an TM3, D− an TM4, D+ an TM5, **VCC nicht anschließen und isolieren**
- [ ] udev-Regel `scripts/51-echo-unlock.rules` installiert
- [ ] Reihenfolge: USB-Kabel in den PC, Punkt-Taste halten, Netzteil einstecken (ohne Netzteil kein Strom, kein Bootrom)
- [ ] `bootrom-step.sh` im Terminal ausführen (fragt nach Enter), dann bei **drehendem Regenbogen** `fastboot-step.sh`
- [ ] **Grüner Ring statt Regenbogen: stoppen**, `fastboot getvar all` notieren, von Wi-Fi fernhalten, in #112 nachsehen.
      Vergleichswerte dort: `pl_build_desc 8f5a1d5-20180802_014127`, `lk_build_desc 6f7225e-20230407_013816`

### C. Stock-Firmware in beide Slots, Root

- [ ] Erste `twrp install`, dann `adb reboot recovery`, weißen Ring abwarten
- [ ] Zweite `twrp install` muss den **anderen** Slot nennen
- [ ] **`bcbtool set_active` nicht benutzen:** TWRP wechselt den Slot selbst. Die Zeile im XDA-Thread ist hier falsch; `setup.sh donut` wechselt nie von Hand
- [ ] `boot-root.zip` installieren (patcht beide Slots), neu starten
- [ ] Ergebnis: Stock Fire OS, unregistriert, kein Wi-Fi, oranger Ring, `adb shell` ist root

### D. Danach

- [ ] `scripts/probe.sh`: darf keine abweichende Bibliothek melden, sonst Schritt 1 wiederholen
- [ ] Sperre **vor** dem WLAN: `deploy.sh`, `lockdown.sh`, dann `wifi-join.sh`
- [ ] **Vor Schritt 5 nicht neu starten:** Das WLAN-Profil überlebt einen Neustart, die Sperre nicht
- [ ] `scripts/install-system.sh`
- [ ] `secrets/update.key` sichern. Sie entsteht erst beim Installer, signiert die Updates und öffnet den Zugang per adb über WLAN
- [ ] Wer sein Wake-Word wechseln will: Der Abschnitt dazu erfordert einmal Amazon-Kontakt, die Update-Sperre laut Upstream vorher einrichten

**Partitions-Backups:** Upstream beschreibt dafür keinen eigenen Schritt für den Dot 3. Die Liste
`preloader, lk_a, lk_b, tee1, tee2, kb, dkb, expdb, misc, persist, boot_a, boot_b, recovery` stammt
aus techo5-dot #19, also vom Dot 2, und ist für den donut nicht geprüft; laut techo5 hat er zum
Beispiel keine `recovery`-Partition. Lesen geht laut #112 mit `./bootrom-step.sh read <partition> <datei>`.

## Reverse-Engineering-Ergebnis: Boot-Kette des donut

```
BROM (MediaTek, USB VID 0e8d:0003; kamakiri-donut startet die Payload über USB-Control-Transfers,
     stage1 @ 0x100A00, boot0-Header wird zuerst gelöscht, damit ein hängender Run den Bootrom
     ohne Kurzschluss erreicht)
 → preloader (151.552 B in der OTA)
 → lk / little kernel (212.992 B; fastboot, unlock_status)
 → tee1/tee2 (TrustZone, je 2.363.392 B)
 → boot (9.330.688 B; Android-Boot-Image)
 → system (1.073.741.824 B; ext4, donut_puffin, A/B)
```

Am Image geprüfte Details und der Weg dorthin stehen in [EVIDENCE.md](EVIDENCE.md). Kurz:

- `build.prop`: `ro.build.version.release=7.1.2` (der Fingerprint nennt 6.0),
  `ro.board.platform=mt8167`, `ro.build.ab_update=true`. Upstream nennt als SoC MT8516.
- Die Payload ist **nicht verschlüsselt**, sondern komprimiert: 523 Operationen (133 `REPLACE_XZ`,
  385 `REPLACE_BZ`, 5 `REPLACE`), alle Prüfsummen stimmen. Statische Analyse der OTA ist also möglich.
  Nur `tee1`/`tee2` bleiben auch dekomprimiert bei Entropie 8,0.
- Der Cloud-Client ist `PuffinApp` (AVS Device SDK über HTTP/2), nicht `alexad`.

## Ring-Diagnose

| Ring | Bedeutung | Vorgehen | Beobachtet an |
|---|---|---|---|
| dauerhaft blau | Tot, evtl. mitten im Update gestorben | Unklar: In #112 brachte der Unlock ein solches Gerät nicht zurück | Dot 3 (#112) |
| **grün**, dauerhaft | Locked fastboot nach kamakiri | Stoppen; **keine Rettung bekannt** (#112) | Dot 3 |
| drehend blau | Stock-Kette versucht ein halb geschriebenes Update zu booten | Vom WLAN fernhalten, #112 lesen | Dot 3 (#112) |
| drehend blau/türkis | Bootloop nach dem Flash | TWRP, Backups zurückspielen | Dot 2 (techo5 #8, #19) |
| drehender Regenbogen | Entsperrtes fastboot | Normal: `fastboot-step.sh` | Dot 3 |
| weiß | TWRP | Normal | Dot 3 |
| orange | Stock, unregistriert, kein WLAN | Normal nach Schritt 1 | Dot 3 |

## Quellen

- echo-dot-assist: `README.md`, `devices/donut/README.md`, `docs/FINDINGS.md`, issues #112, #62
- techo5-dot: issue #37 (Dot 3); #8, #19 (Dot 2); #1 (Gerät nicht genannt)
- bboe/firebreak: `docs/unlocking.md`, PR #114 (Dot 2)
- XDA: [UNLOCK/ROOT/TWRP Echo Dot 3rd Gen 2018 donut](https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-dot-3rd-g-2018-donut.4801400/)
  (für diese Prüfung nicht gelesen; die `bcbtool`-Zeile dort widerspricht Upstream)
- Eigene OTA-Analyse: [EVIDENCE.md](EVIDENCE.md) (Payload-Prüfung per stdlib-Inline-Skript, siehe dort)
