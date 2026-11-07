# Eigene Alexa auf dem Dot 3 – abgestimmte Open-Source-Empfehlung (4 Agenten, Stand 2026-11-07)

Kein eigener Harness (entfernt). Recherche mit 4 Agenten-Perspektiven (präzise / breit / HA / OVOS),
abgeglichen. Einstimmiges Fazit:

## Empfehlung (alle Agenten gleich)
**Server = Home Assistant (Core 2026.11.0) + Assist-Pipeline + Wyoming-Dienste;
Dot 3 = Satellite via echo-dot-assist (hassmic, ESPHome- oder Wyoming-Modus) nach kamakiri-Unlock;
LLM = eigene API (Ollama / LiteLLM / llama.cpp / eigenes OpenAI-kompatibles Gateway);
Amazon = on-device-Firewall (hassmic) + Router/DNS-Block (MAC 3C-5C-C4-B7-17-DE).**

```
Echo Dot 3 (unlocked, echo-dot-assist/hassmic)
  on-device: Wake-Word (Amazon-Pryon ODER microWakeWord/openWakeWord), Mic-AEC, Speaker, Firewall
  │ ESPHome-Native-API (default, kein YAML) oder Wyoming (PROTO=wyoming, :16700)
  ▼
Home Assistant 2026.11: Pipeline (UI) Sprache=de
  STT: wyoming-faster-whisper v3.8.1 (language: de, model turbo/small; DE-Finetune TheTobyB-ct2)
       oder Speech-to-Phrase (de) für reine Gerätesteuerung
  Agent: built-in Intents (lokal, "prefer local") ODER eigenes LLM:
       - HA ≥2026.8: native `litellm`/`llama_cpp`-Agents; ollama-Integration (qwen3:8b, Tool-Calling)
       - eigene API: HACS `ha-openai-compatible` / `extended_openai_conversation` mit base_url
         (core openai_conversation hat KEIN base_url!) – oder LiteLLM v1.108.2 als Proxy davor
  TTS: wyoming-piper v2.5.2, Voice de_DE-thorsten-* (rhasspy/piper archiviert → OHF-Voice/piper1-gpl)
  Skills/Steuerung: exponierte Entities + HA-Automationen/Blueprints + LLM-Prompt
```

## Alternativen (abgestimmt)
| Stack | Wann | Status 2026-11-07 |
|---|---|---|
| **OVOS / Neon** | eigene Skill-Plattform statt HA | sehr aktiv (ovos-core 3.7.5a1, tägliche Commits; Persona+Solver = beliebige OpenAI-kompatible LLM-API; Deutsch 97 %; HA-Kopplung via skill-homeassistant + Wyoming-Bridges). Kein nativer donut-Client → Dot hängt an HA, OVOS parallel. |
| **techo5-dot v1.0.56** | statt Fire OS komplettes Alpine-Linux auf dem Dot | donut unterstützt (1 Gerät getestet); signierte A/B-Updates; ESPHome-Satellite; max. lokal. |
| **Tater v2.0.7** | kompletter Ersatz-Stack mit eigenen Echo-Satellites | aktiv (heute released); eher实验haft. |
| Rhasspy | – | **tot** (archiviert); nur Wyoming-Protokoll lebt (OHF-Voice v1.11.0). Nicht nutzen. |
| LibreEcho | donut-Port | nur Research (#348) – nicht verfügbar. |

## Ohne Öffnen (Fallback, sofort nutzbar)
Kein eigener Code läuft auf dem Dot selbst (siehe REMOTE-ATTACK-SURFACE.md). Steuerbar bleibt er
extern: HA `alexa_media_player` (TTS/Media auf den Dot) + Bluetooth/AUX/DLNA. Amazon-Server am
Router blocken (MAC des Dot, siehe REMOTE-ATTACK-SURFACE.md).

## Hardware-Pfad (kamakiri) – Gates
Nur D9N29T, nur Fire OS 6574.1 (NS65741, probe.sh + sha256), nativer Linux-PC, Testpads TM3/4/5
(VCC offen), nie online vor dem Unlock (OTA schließt Exploit), Backups aller Boot-Partitionen,
beide A/B-Slots. firebreak v1.0.0 = Dot 2; donut = kamakiri-donut v1.0.0 (XDA 4801400).
Risiko: echo-dot-assist #112 (frisch geupdatete Geräte → locked, nicht heilbar).
→ Details: `docs/SAFETY-BRICK-AVOIDANCE.md`.

## Quellen (Auswahl, alle 2026-11-07 geprüft)
home-assistant.io/voice_control + /integrations/{wyoming,ollama,litellm,llama_cpp} ·
github.com/OHF-Voice/{wyoming,wyoming-faster-whisper v3.8.1,wyoming-piper v2.5.2,linux-voice-assistant,piper1-gpl} ·
github.com/Gamer92000/echo-dot-assist (daily betas) · github.com/OpenVoiceOS (ovos-core, persona, skills-store) ·
github.com/BerriAI/litellm v1.108.2 · github.com/adamjs83/ha-openai-compatible ·
github.com/HuskerMinion/techo5-dot · xdaforums.com/t/4801400 · github.com/bboe/firebreak v1.0.0
