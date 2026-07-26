# Voice Assistant

A reliable, fully local voice assistant built on Home Assistant. This repo collects the configs and prompts that back the setup. The full narrative — hardware choices, failure modes, and tradeoffs — lives in the Home Assistant community write-up:

[My journey to a reliable and enjoyable locally-hosted voice assistant](https://community.home-assistant.io/t/my-journey-to-a-reliable-and-enjoyable-locally-hosted-voice-assistant/944860)

## Architecture

```
Wake word  ->  STT  ->  LLM (tool calls)  ->  TTS proxy  ->  TTS
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

- Two Satellite1 small squircle enclosures
- Two Facebook Portals running View Assist as a hub-style satellite
- Home Assistant Voice Preview Edition satellite

### Voice server

- Beelink MiniPC with USB4
- USB4 eGPU enclosure
- GPU sized to the chosen LLM — a 24 GB card (RTX 3090 / RX 7900 XTX class) comfortably runs the 20B–30B MoE / ~9B dense models this setup targets. Smaller cards work with smaller models at the cost of response time.

### Wake word

- Custom "Hey Robot" model trained with the microWakeWord trainer [AppleSilicon](https://github.com/TaterTotterson/microWakeWord-Trainer-AppleSilicon) | [Nvidia GPU or General CPU (slower)](https://github.com/TaterTotterson/microWakeWord-Trainer-Nvidia-Docker)

### STT Model

- Qwen3-ASR 1.7B, running in llama.cpp — see [configs/config.ini](configs/config.ini)

### Agent Model

- Gemma4 26B-A4B, chat and tool calling, running in llama.cpp — see [configs/config.ini](configs/config.ini)

### TTS

- Kokoro TTS — handles currency, phone numbers, and addresses well, and supports voice mixing
- [TTS Proxy](https://github.com/Thyraz/tts-proxy) — a HACS integration that wraps the real TTS entity and rewrites the model's text before it is spoken: strips markdown and emoji, and spells out numbers, dates, times, and units. Home Assistant points at the proxy entity, and the proxy forwards to Kokoro.

Speech formatting belongs here rather than in the system prompt. The model keeps writing normally — digits, dates, units — so responses stay readable in the text chat view, and the speech-only cleanup happens at the layer that actually feeds the speech.

### Home Assistant integrations

Several of these are third-party HACS integrations, not part of a stock Home Assistant install. `llm_intents` in particular is load-bearing: it's what lets the custom [prompts/llm-context-prompt.jinja](prompts/llm-context-prompt.jinja) override the default Assist context, and it's how unused built-in tools are hidden from the model.

- [LLM Conversation](https://github.com/skye-harris/hass_local_openai_llm) for running OpenAI compatible LLM backends with optimizations for HomeAssistant
- [llm_intents](https://github.com/skye-harris/llm_intents) — web search, places, weather forecast; also overrides the default Assist context template and controls which tools are exposed to the model
- [Model Context Protocol](https://www.home-assistant.io/integrations/mcp/) — connects the assistant to the long-term memory service so it can recall home-specific facts. See [memory/README.md](memory/README.md).
- [TTS Proxy](https://github.com/Thyraz/tts-proxy) — post-processes responses for speech; see [TTS](#tts) above

## Repo layout

- [configs/](configs/) — llama.cpp server config for the LLM and STT model. See [configs/README.md](configs/README.md).
- [prompts/](prompts/) — the system prompts that shape behavior: identity and response rules, the Home Assistant context template, and the STT transcription prompt. See [prompts/README.md](prompts/README.md). Build one interactively with the [Prompt Builder](https://nickm-27.github.io/VoiceAssistant/) ([docs/index.html](docs/index.html)).
- [tools/](tools/) — custom Home Assistant script tools exposed to the LLM, such as resolving the area the user is speaking from. See [tools/README.md](tools/README.md).
- [memory/](memory/) — the Docker setup for the long-term memory service (MCP memory server + filtering proxy) the assistant searches for home-specific facts. See [memory/README.md](memory/README.md).

## Known issues worth knowing up front

- Ollama's defaults make it easier to get started but in doing so make a lot of decisions that add latency and reduce efficiency. Prefer llama.cpp with an explicit, higher-quality quant.
- Too many exposed entities will blow the context window and tank tool-calling accuracy. Group devices in Home Assistant and expose the groups. The reference setup exposes fewer entities.
