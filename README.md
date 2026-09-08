# Conversational Voice Generator

Internal tool for **Essent** customer support: generate realistic and edge-case
simulated phone conversations between a customer and our voice bot, to test the
bot's robustness before it goes live.

The bot side is **deterministic** (a scripted state machine that lives in the
code). The customer side is played by an **LLM** whose style is shaped by three
persona sliders. Every generated conversation is in Dutch.

## Run it

No build step, but it **must be served over HTTP** — opening `index.html` as a
`file://` URL makes the browser send `Origin: null`, which OpenRouter's CORS
rejects, so every turn fails with *"Failed to fetch"*. Serve the folder:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/
```

(The page detects `file://` and shows a warning if you forget.)

### Troubleshooting

| Error | Cause / fix |
|---|---|
| `OpenRouter onbereikbaar` / `Failed to fetch` | Opened as `file://` (see above), or the network blocks the API. The error now also shows a no-auth probe of `openrouter.ai`: if that is "óók onbereikbaar", the whole domain is blocked here — common on mobile hotspots, VPNs, corporate proxies, and privacy extensions / ad-blockers (uBlock, Brave Shields). Try a normal Wi-Fi connection, disable shields/extensions for `localhost`, or use a different browser. The request already sends only `Authorization` + `Content-Type` (no extra headers) to keep the CORS preflight minimal. |
| `AbortSignal object could not be cloned` | Old version in an embedded browser — fixed; pull latest. |
| `OpenRouter 401` | Bad or expired API key. |
| `OpenRouter 402` | Out of OpenRouter credits. |
| `Time-out …` | Provider was slow; click **Beurt opnieuw proberen**. |

## Configure

| Field | Notes |
|---|---|
| **OpenRouter API key** | Entered in the UI, stored in `localStorage`. **Never commit a key.** See security note below. |
| **Klantintelligentie** | 0 = struggles with simple instructions, swaps digits, needs everything repeated · 100 = sharp and efficient |
| **Tech-vaardigheid** | 0 = doesn't know what "IBAN" means, fumbles the phone · 100 = knows exactly what's asked |
| **Stemming** | 0 = calm and friendly · 100 = angry, threatens to escalate |
| **🎲 Randomize** | Jumps all three sliders to random values — quick way to batch edge cases. |
| **Forceer edge case** | Adds one curveball instruction to the prompt each turn (wrong digit, changing their mind, background noise, resistance to a question). |
| **🎭 Genereer klantbeeld** | Optional. Sends a prompt built from the sliders + the generated customer record (approx. age, name) to an image model (`IMAGE_MODEL`, default `google/gemini-2.5-flash-image`, ~$0.04/image, ~10s) and shows an AI impression of the caller. Deliberately over-the-top — a comedy-series / reality-TV still, on-camera flash, caught mid-gesture — with a random scene + prop each time, so it exaggerates the sliders (fuming rage, phone held upside down, papers flying, smug thumbs-up…). Still an illustration, not the real customer. Cleared on New session / Start. If a portrait was generated it is embedded (large base64) in the JSON export; **Bewaar afbeelding** saves it separately. Tune the tone in `buildPersonaPrompt()` / `PERSONA_SCENES` / `PERSONA_PROPS`. |

The transcript is paced at roughly one line every ~2 seconds (`TURN_GAP_MS` /
`BOT_READ_MS` in `index.html`) so a viewer can follow the call unfold even when
the model responds instantly; when the model is slow its latency dominates.

**Klantstem (optional TTS).** Only the *customer's* lines, never the bot's. Tick
**Klantstem** to have each customer reply spoken aloud as it appears, or click
▶ on any customer bubble to (re)play just that line. `openai/gpt-audio-mini`
via OpenRouter, streamed as pcm16 and muxed to WAV in the browser; emotion is
taken from the Stemming slider. After a session: **🔊 Speel klant af** plays all
customer lines back to back, **⬇︎ Klantaudio (.wav)** downloads them as one file
(clips generated on demand, ~1–2s each, ~$0.002/line with the mini model).
Model/voice are `VOICE_MODEL` / `VOICE_NAME` in `index.html`. Auto-play mid-call
can be blocked by the browser's autoplay policy — the ▶ buttons always work.

The customer model is fixed to **`z-ai/glm-5.3-flash`** (`DEFAULT_MODEL` in
`index.html`, one line to change). GLM 5.3 Flash accepts `response_format` but
does not enforce a JSON schema, so the app also sends a strict "JSON only"
prompt and parses tolerantly (code fences, reasoning preambles). It is a
reasoning model that cannot have reasoning disabled, so the app requests
`reasoning: { effort: "low" }`, allows a generous `max_tokens`, and applies a
per-request timeout so a slow provider surfaces a retry button instead of
hanging. Each run still generates an internal random seed (ground-truth record +
Socratic subset) that is recorded in the exported JSON for traceability.

## Bot state machine

1. **Greeting** — fixed line.
2. **Authentication** — asks `date_of_birth`, `postcode`, `iban_last3` one at a
   time. Each field: match → advance; unclear/mismatch → one fixed re-ask;
   second miss → fixed "verificatie mislukt" hand-off and the session ends
   (this failure path is a deliberate, exportable test case).
3. **Confirm old amount** — states the ground-truth monthly amount.
4. **Ask new amount** — parses a number from the reply.
5. **Explicit confirmation** — repeats the number, requires a spoken yes/no.
   Only "yes" marks it `saved` with a timestamp; "no" loops back to step 4.
6. **Socratic follow-up** — 3–5 questions from the scenario's question bank,
   asked one at a time, answers captured verbatim.
7. **Closing** — fixed line; session `complete`, export unlocked.

The engine dispatches purely on `step.type` and reads everything from a scenario
config object (`SCENARIOS` in `index.html`). Only `budget_bill_change`
(*Maandbedrag wijzigen*) exists today; a second topic is just another config
object — no engine changes.

## OpenRouter integration

One chat-completion call per **customer** turn. The system prompt is built from
the scenario, the ground-truth record, the slider values + their descriptors,
the current bot question, and the full history so far. The model must return:

```json
{
  "spoken_reply": "string — natural spoken Dutch",
  "provided_value": "string | null — normalized value for the field being asked",
  "wants_repeat": "boolean",
  "off_topic": "boolean",
  "wants_to_end_call": "boolean"
}
```

`spoken_reply` is rendered; the other fields drive the state machine.

## Export

- **Kopieer transcript** — plain text for pasting into a ticket / test doc.
- **Download transcript (.json)** — full turn-by-turn transcript, slider values +
  descriptors, ground-truth record, verification pass/fail, saved amount +
  timestamp, timestamps, engine log.
- **Nieuwe sessie** — fresh ground-truth record, sliders/transcript reset.

## Security note

This tool calls OpenRouter **directly from the browser**, so the API key is
exposed client-side and travels in network requests. That is acceptable for an
**internal test tool only** — do not deploy this publicly, and do not commit an
API key to this repo. Use a key with a low spending cap and rotate it freely.
