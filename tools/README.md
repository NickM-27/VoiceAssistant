# tools

Custom Home Assistant script tools exposed to the LLM. These are defined as HA scripts and surfaced to the model alongside the built-in intents, giving the assistant capabilities Assist doesn't provide out of the box.

## GetCurrentLocation.yaml

Returns the area — and floor, when known — the user is currently speaking from.

### What it does

When you talk to a voice satellite, that satellite briefly enters the `processing` state while it handles the request. The script scans every `assist_satellite` entity, finds the one that's processing, and resolves its area and floor:

- Exactly one satellite processing → returns that satellite's `area` and `floor`.
- More than one → returns `Unknown` (it can't tell which one you meant).
- None → returns `none`.

### Why it's helpful

By default, Home Assistant bakes the current speaker's area directly into the system prompt. That works, but the area changes depending on which satellite you're talking to — so the prompt prefix is no longer stable. That breaks prompt caching: you either need a dedicated cache slot per speaker, or the prompt has to be reprocessed every time you move to a different room.

Making location a tool the model calls on demand removes that problem. The system prompt stays identical across every satellite, so the cache stays warm, and the model looks up the room only when a request actually needs it.

It pairs with the area-disambiguation rules in [../prompts/llm-context-prompt.jinja](../prompts/llm-context-prompt.jinja): the prompt tells the model that generic, area-less commands target the current area, and this tool is how it finds out what that area is. The model is told (via the script's `description`) to call it when a request names a generic device without specifying one.

### Installing

Add it as a script in Home Assistant (paste into the scripts editor in YAML mode, or drop it into your scripts file) and expose it to your conversation agent so the model can call it.
