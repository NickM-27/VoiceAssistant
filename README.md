# Voice Assistant

A reliable, fully local voice assistant built on Home Assistant. This repo collects the configs and prompts that back the setup. The full narrative — hardware choices, failure modes, and tradeoffs — lives in the Home Assistant community write-up:

[My journey to a reliable and enjoyable locally-hosted voice assistant](https://community.home-assistant.io/t/my-journey-to-a-reliable-and-enjoyable-locally-hosted-voice-assistant/944860)

## Architecture

```
Wake word  ->  STT  ->  LLM (tool calls)  ->  TTS
                          |
                          +-- Home Assistant (devices, intents)
                          +-- llm_intents (web search, places, weather, YouTube)
                          +-- Music Assistant
                          +-- Frigate (camera vision)
                          +-- Memory tool
```

Each stage is swappable. The LLM is the only piece that is meaningfully sensitive to hardware.

## Components

### Voice endpoints

- Home Assistant Voice Preview Edition satellite
- Two Satellite1 small squircle enclosures
- Pixel 7a running View Assist as a hub-style satellite

### Voice server

- Beelink MiniPC with USB4
- USB4 eGPU enclosure
- GPU sized to the chosen LLM — a 24 GB card (RTX 3090 / RX 7900 XTX class) comfortably runs the 20B–30B MoE / ~9B dense models this setup targets. Smaller cards work with smaller models at the cost of response time.

### Wake word

- Custom "Hey Robot" model trained with the microWakeWord trainer [AppleSilicon](https://github.com/TaterTotterson/microWakeWord-Trainer-AppleSilicon) | [Nvidia GPU or General CPU (slower)](https://github.com/TaterTotterson/microWakeWord-Trainer-Nvidia-Docker)

### STT

- Gemma4 E4B running in llama.cpp

### LLM runtime

- llama.cpp (chosen over Ollama for the tuning it exposes — see [configs/](configs/))

### Current Models

- Gemma4 26B A4B (MoE), Q4_K_XL quant from Unsloth — chat and tool calling
- Gemma4 E4B Q4_K_XL — speech-to-text

### TTS

- Kokoro TTS — handles currency, phone numbers, and addresses well, and supports voice mixing

### Home Assistant integrations

Several of these are third-party HACS integrations, not part of a stock Home Assistant install. `llm_intents` in particular is load-bearing: it's what lets the custom [prompts/llm-context-prompt.jinja](prompts/llm-context-prompt.jinja) override the default Assist context, and it's how unused built-in tools are hidden from the model.

- [LLM Conversation](https://github.com/skye-harris/hass_local_openai_llm) for running OpenAI compatible LLM backends with optimizations for HomeAssistant
- [llm_intents](https://github.com/skye-harris/llm_intents) — web search, places, weather forecast; also overrides the default Assist context template and controls which tools are exposed to the model

## Repo layout

- [configs/](configs/) — llama.cpp server config for the LLM and embedding model. See [configs/README.md](configs/README.md).
- [prompts/](prompts/) — the system prompts that shape behavior: identity and response rules, the Home Assistant context template, and the STT transcription prompt. See [prompts/README.md](prompts/README.md).

## Known issues worth knowing up front

- Home Assistant's built-in `HassGetWeather` intent returns garbage when no weather entity is exposed. Override it with an automation that routes weather questions to the `llm_intents` weather tool.
- Ollama's defaults make it easier to get started but in doing so make a lot of decisions that add latency and reduce efficiency. Prefer llama.cpp with an explicit, higher-quality quant.
- Too many exposed entities will blow the context window and tank tool-calling accuracy. Group devices in Home Assistant and expose the groups. The reference setup exposes fewer entities.
