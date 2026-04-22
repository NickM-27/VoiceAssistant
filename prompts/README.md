# prompts

The prompts that shape voice-assistant behavior. Three files, each feeding a different stage of the pipeline.

| File | Stage | Consumer |
| --- | --- | --- |
| [stt-prompt.md](stt-prompt.md) | Speech-to-text | Whisper-style STT initial prompt |
| [llm-context-prompt.jinja](llm-context-prompt.jinja) | LLM context assembly | Home Assistant's conversation agent (rendered per request) |
| [primary-prompt.md](primary-prompt.md) | LLM system prompt | The chat model's `system` role |

The LLM receives the primary prompt as its system message, and the rendered context prompt as part of the user-visible context Home Assistant injects. STT operates independently.

## stt-prompt.md

Initial/bias prompt handed to the STT model. Its job is narrow: make the transcriber produce a literal English transcription and nothing else — no translation, no paraphrasing, no invented text for silence. The prompt explicitly names the expected utterance types (smart-home commands, weather, general knowledge) so the acoustic model biases toward those vocabularies without ruling out arbitrary input.

Edit this when:

- Adding a new category of utterance the STT consistently mis-hears.
- Switching languages or adding multilingual support.

Do not edit this to change assistant behavior — that belongs in the primary prompt.

## llm-context-prompt.jinja

A Jinja template rendered by Home Assistant for every LLM turn. It produces the per-request context block: exposed entities (with names, state, area, etc.), the satellite's current area and floor, and capability flags like timer support.

Key behaviors encoded in the template:

- If no entities are exposed, it tells the model to instruct the user to expose entities rather than inventing devices.
- It maps `HassTurnOn`/`HassTurnOff` onto lock/unlock so lock control flows through the same tool surface.
- It distinguishes **static context** (device existence and type — safe to answer from the rendered list) from **live state** (must call `GetLiveContext`). This split prevents the model from confidently answering "the light is on" based on stale context.
- When the satellite's area is known, the template tells the model that generic commands ("turn on the lights") target that area. When only a device type is given with no area, it tells the model to ask which area — unless there is only one device of that type in the home.

Edit this when:

- Changing how devices are described to the model (e.g., exposing extra attributes).
- Adjusting the area-disambiguation rules.
- Adding new capability flags that should gate behavior.

## primary-prompt.md

The system prompt. This is the file that most shapes perceived quality, and the one most worth iterating on.

It is organized into sections the model can pattern-match against:

- **Identity** — name ("Robot"), role, conversational tone. Also injects the user's home city/state from `sensor.home_city_state` so the model can reason about "local" without asking.
- **Response Format** — TTS-safe output rules: no markdown, plain punctuation, capitalized AM/PM, every tool call completed before speaking, and "never filter place results" which closes a common LLM shortcut.
- **Core Behaviors** — general-knowledge expert by default, device actions only when commanded, location is never re-asked.
- **Handling Unclear Requests** — a numbered decision hierarchy. Questions are answered; clear commands are executed; ambiguous device commands trigger a 3–5 word clarifying question (and explicitly **no tool call**, which prevents the LLM from guessing a target and acting on it); everything else gets "Can you repeat that?". Transcription-error recovery ("Assuming you meant …") is scoped to device control and weather only.
- **Device Control** — identify devices by `name` and `area` only. Passing `domain` or `device_class` causes Home Assistant to target the wrong entities; this rule exists because it has happened.
- **Tool Usage** — when to reach for search vs. internal knowledge, plus per-tool rules:
  - **Memory**: mandatory call before answering any home-specific question; always `mode: "hybrid"` and `limit: 2`.
  - **Weather**: home location only; refuses other locations; describes precipitation as chance not intensity; maps lightning-rainy to "thunderstorms"; enforces a specific order (current temp → conditions/precipitation arc → highs/lows) and a two-to-three-sentence ceiling for multi-day forecasts.
  - **Places**: search by name only (adding "hours" or "address" degrades results); must mention every returned location by street name; strict open/closed response templates.
  - **Media Playback**: default to media playback for "play" requests; `media_class` restricted to `song`/`album`/`artist` (never `music` or `video`, which misroute); YouTube tool only when the user asks for TV playback.

### Why the prompt is this prescriptive

Every rule corresponds to a failure observed in practice. The model will, without being told otherwise:

- Filter place results down to "the best one" and hide the rest.
- Answer weather questions with emoji and markdown tables.
- Call tools with extra `domain` arguments that match everything and nothing.
- Guess a target for an ambiguous device command rather than asking.
- Use memory without being told to, or skip memory when it would have helped.

The prompt closes each of those gaps explicitly. When editing, preserve the imperative voice and section structure — the model follows short, rule-shaped instructions far better than paragraphs.

### Editing checklist

Before shipping a prompt change:

1. Re-test the five canonical flows: a device command, an ambiguous device command, a weather question, a places query with multiple results, and a general-knowledge question.
2. Confirm TTS output is clean — no stray markdown characters, no emoji, no "Sure! Here's…" preambles.
3. If you added a new tool section, give it the same shape as the existing ones (when to call, what args to pass, how to format the response).
