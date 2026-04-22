# prompts

The prompts that shape voice-assistant behavior. Three files, each feeding a different stage of the pipeline.

| File | Stage | Consumer |
| --- | --- | --- |
| [stt-prompt.md](stt-prompt.md) | Speech-to-text | Initial/bias prompt handed to the STT model |
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

A Jinja template rendered per LLM turn. It overrides the default Assist context-injection path via the [LLM Intents](https://github.com/skye-harris/llm_intents) integration, which is what makes a custom template possible in the first place and also what's used to hide unused built-in tools from the model.

The goal is tighter, more deliberate context than Assist gives you by default:

- Render the exposed entities as **static context** (names, areas, types — fine to answer from directly) and lean lightly on `GetLiveContext` rather than pushing the model to call it for every question. Stock Assist over-emphasizes live-state calls; this template trusts the rendered list for "do I have X" questions and only asks for live state when the answer actually depends on it.
- Map `HassTurnOn`/`HassTurnOff` onto lock/unlock so lock control flows through the same tool surface.
- When the satellite's area is known, tell the model that generic commands ("turn on the lights") target that area. When only a device type is given with no area, ask which area — unless there is only one device of that type in the home.
- If nothing is exposed, tell the model to instruct the user to expose entities rather than inventing devices.

Edit this when:

- Changing how devices are described to the model (e.g., exposing extra attributes).
- Adjusting the area-disambiguation rules.
- Adding new capability flags that should gate behavior.

Tool exposure (which built-in intents the model sees) is configured separately in LLM Intents, not in this template.

## primary-prompt.md

The system prompt. This is the file that most shapes perceived quality, and the one most worth iterating on. Read it directly for the specifics — it's short and already grouped by topic.

### Why the prompt is this prescriptive

Out of the box a general-purpose chat model doesn't know what "good" looks like for voice. The prompt is where you encode your preferences across three axes:

- **Brevity.** Models default to conversational padding — "Sure, I can help with that. Here's what I found…" — and to exhaustive answers when a short one was wanted. For voice, every extra word is latency the user hears. The prompt sets explicit ceilings (multi-day forecasts capped at two or three sentences, general answers limited to what was asked) because without them the model fills the space.
- **Response shape.** Separate from length, this is *how* an answer is structured. Without guidance, the model will narrate a device action before performing it ("Okay, turning the kitchen light on now…") or answer "is the pharmacy open?" by reading off hours and letting the user do the math. The prompt pins down the shapes that matter: execute device commands first and report the final state, sequence weather as current temperature → conditions and precipitation arc → highs and lows, answer open/closed questions with the explicit "open right now and closes at …" / "currently closed and opens at …" templates. Consistent shape is what makes the assistant feel like a product instead of a chatbot.
- **Tool-calling effectiveness.** Models over- and under-call tools in equally annoying ways. They answer home-specific questions from guesses instead of querying memory, pass extra arguments (`domain`, `device_class`) that misroute Home Assistant service calls, or stop after one tool call when the question needs two. Per-tool rules — when to call, what to pass, how to incorporate the result — are what turn a capable model into a reliable one.

Tune these to your own preferences. Want chattier replies? Loosen brevity. Prefer to hear the action announced before it happens? Change the response-shape rules. The point isn't the specific choices in this prompt, it's that the choices are written down — swap the model and the behavior stays close to what you wanted.

When editing, preserve the imperative voice and short, rule-shaped lines. Models follow "Do X. Never Y." far better than paragraphs of guidance.

### Editing checklist

Before shipping a prompt change:

1. Re-test the five canonical flows: a device command, an ambiguous device command, a weather question, a places query with multiple results, and a general-knowledge question.
2. Confirm responses follow their templates: device actions report final state without narrating the request first, weather sequences correctly, and open/closed answers use the exact "open right now" / "currently closed" phrasing.
3. If you added a new tool section, give it the same shape as the existing ones (when to call, what args to pass, how to format the response).
