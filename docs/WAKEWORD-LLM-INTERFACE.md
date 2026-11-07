# Wake-Word → LLM: die Schnittstelle auf dem Dot 3 (D9N29T)

Ziel: Wake-Word bleibt on-device, danach übernimmt das eigene LLM (über HA/OVOS-Harness, nicht selbstgebaut).

## 1. On-device: die Wake-Word-Schnittstelle (libpryon / Pryon-API)
Quelle: echo-dot-assist `docs/re-pryon.md` (statisch + qemu + on-device bestätigt), Firmware NS65741/8138.
- `libpryon.so` (19,8 MB): processweiter Singleton; Modelle als Model-Sets (`pryon.manifest`), Decoder per
  `PryonDecoder_NewSpotterAudioDecoder(...)`, Audio-Format **16 kHz / 16 bit / mono** (`PryonDecoder_PushAudioEventSamples`),
  Ergebnis-Callback `PryonApi_SetEnumeratedResultCallback`, Client-Events `PryonDecoder_PushClientEvents`
  (z. B. `AlarmState`, `AudioPlayerState`, `audio_playback` – hassmic setzt sie, damit Wake-Word während
  Antworten/Timern leiser erkannt wird).
- Stock-Modelle: `en-US/ALEXA`, `echo-en-US`, `echo-de-DE` (DAVS) mit unterschiedlichen Thresholds.
- **Eigenes Wake Word:** echo-dot-assist akzeptiert beliebige **microWakeWord / openWakeWord**-Modelle
  (README-Tabelle + GUIDE#wake-word) – trainierbar z. B. auf openwakeword.com / HA-Wakewords-Collection.

## 2. Audio-Schnittstelle zum Server
- `mixer`/`libasp.so` stellt den **micAsr-Stream: 16 kHz mono s16, post-AEC** bereit (single consumer).
- echo-dot-assist `hassmic` streamt das + Wake-Events an den Server:
  - **ESPHome-Mode (default):** Gerät erscheint in HA als ESPHome-Device (`assist_satellite`, `media_player`,
    `light` Ring, `event` Tasten, Sensoren/Switches) – kein YAML nötig; verschlüsselte native API, Key setzt HA.
  - **Wyoming-Mode** (`PROTO=wyoming`, Port 16700): reiner Audio-Satellite; Wake-Word server-seitig
    (wyoming-openwakeword / microWakeWord als Wyoming-Service).
- Settings/Log auf dem Dot: `http://<echo-ip>:28931/`.

## 3. Server: Wake-Event → LLM-Pipeline (HA 2026.11, UI-konfiguriert)
Pipeline-Stages (developers.home-assistant.io/docs/voice/pipelines): `wake_word → stt → intent/conversation agent → tts`.
```yaml
# Conversation-Agent = eigene LLM-API (HACS ha-openai-compatible, da core openai_conversation kein base_url hat)
openai_compatible:
  - name: "Meine LLM"
    api_key: !secret llm_key
    base_url: "http://llm-server:4000/v1"   # LiteLLM/vLLM/eigene API
    model: "mein-modell"
    prompt: "Du steuerst Home Assistant auf Deutsch. Kurz antworten."
# native Alternativen seit HA 2026.8: litellm / llama_cpp Agenten; ollama-Integration (qwen3:8b, Tool-Calling, <25 exposte Entities)
# STT: Whisper-App (language: de, model: turbo)  |  TTS: Piper-App (de_DE-thorsten-*)
# Pipeline im UI anlegen (Sprache de) und dem Satellite zuweisen; Entities exposen; custom_sentences/de/*.yaml für eigene Sätze.
```
- OVOS-Alternative: OVOS-Core + Persona/Solver (beliebige OpenAI-kompatible API), HA-Kopplung via
  `skill-homeassistant` (OVOS→HA) und Wyoming-Bridges; Satellite am HiveMind-Hub.

## 4. Einbindung
Empfohlener Stack inkl. LLM-Anbindung: `OPEN-SOURCE-STACK.md` (HA Assist + Wyoming + eigene
LLM-API; OVOS als Alternative).
