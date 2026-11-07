# Setup-Modus des Echo Dot 3 – Angriffsfläche (nur nach Factory Reset / Setup-Taste)

Quelle: cgarrot/echo-dot-3-doebrite-research (doebrite NS65741/8149, 2026 – gleiche puffin-Familie wie
donut), insb. `evidence/THRIFT_IDL.md`, `evidence/OOBE_SPEC.md`, `evidence/PROVEN.md`,
`evidence/SYNTHESE.md`, `docs/03-oobe-protocol.md`, `disclosure/DISCLOSURE_AMAZON.md`.

## Wie man in den Setup-Modus kommt
Factory Reset (Action-Taste ~20 s halten) oder Netzwerk-Trennung → Dot spannt offenen AP
`Amazon-XXXX`, Gateway **10.201.126.241/28**. Gemessen: in diesem Zustand ist **nur TCP 8080 offen**
(1080/4070/5541/8888 sind zu). dnsmasq hört auf UDP/53 (nur Setup), TCP per Firewall geblockt.

## oobed :8080 – Apache Thrift 0.9.0 (TJSON), unauthenticated
- HTTP `POST http://10.201.126.241:8080/OOBE`, `Content-Type: application/x-thrift`, Body = TJSON array
  `[1,"method",1,seq,{args}]`. Stack: civetweb → libdoppleroobe.so → **libthrift 0.9.0 (2013)**
  (auch im 2026-Build noch drin → Amazon hebt Deps nicht an).
- 15 Methoden (exakte IDL aus jadx von base.apk): `ping, getDeviceDetails, getScanList, getKnownList,
  connectToAP, connectToAPEx, disconnectFromAP, forgetAP, getRegistrationState,
  getAlexaConnectionState, getLinkCode, getOTADetails, evaluateCaptivePortal, setLocaleAndEndpoints,
  setupComplete`.
- **CVE-2019-0205 (pre-auth DoS)**: TJSONProtocol rekursiert ohne Tiefenlimit. PoC (5 Zeilen):
  `body = b'[1,"ping",1,1,' + b'{"0":{"rec":'*20000 + b'{}' + b'}}'*20000 + b']'` (~280 KB) →
  Verbindung dropped, `oobed` ist `oneshot` und antwortet **nie wieder** bis Power-Cycle.
- **SSRF in `getLinkCode`**: Device baut TLS-Verbindung zu `persist.oobe.reg.endpoint:port` – Host
  über DNS auflösbar (Umleitung api.amazon.com→127.0.0.1 honoured), Port client-gesteuert via
  `setLocaleAndEndpoints` → Loopback-Port-Oracle (eigenes :8080 erreichbar). Volles MITM scheitert
  an Zertifikatsprüfung (CN=api.amazon.com, TLS 1.1 only).
- **Info-Leak `getDeviceDetails`**: DSN, MAC, productId, deviceType + **ephemeral setup cert**
  (CN=setup,O=Lab126, pro Session neu).
- **Provisioning-Takeover `connectToAP`**: WPA2-PSK als PKCS#7/CMS (AES-256-CBC, RSA key transport)
  an das frische Device-Cert verschlüsselt (`openssl smime -encrypt -aes256 -outform PEM`).
  Offen: `smethod=1` ohne Passwort. → Angreifer-AP = volle Traffic-Kontrolle über den Dot.
  **Warnung: `setupComplete` ohne Registration ist destruktiv** (Gerät in unerreichbarem
  Zwischenzustand, nur Factory Reset hilft).

## dnsmasq 2.51 (nur Setup-AP)
Versionsbedingt angreifbar (CVE-2017-14491/92/93 RCE-Klasse, P0-PoCs public), gemessen aber
„non atteignable": TCP/53 per Firewall zu, `-DNO_SCRIPT`, kein Upstream im Setup → praktisch DoS-Leer.

## Relevanz fürs eigene Gerät
Setup-Modus ist der einzige Zustand mit echter RCE-Klasse – aber nur erreichbar, wenn der Dot
factory-resettet ist, und selbst dann kam kein Code-Exec zustande (cgarrot: „no RCE, no root“).
Für deinen D9N29T heißt das: Setup-Fenster meiden (nie resetten, wenn du ihn behalten willst) –
und im eigenen Netz ist der Setup-AP eine Einladung; am Router sieht man die MAC 3C-5C-C4-B7-17-DE.
