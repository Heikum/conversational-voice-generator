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
| `Failed to fetch` / `Netwerkfout richting OpenRouter` | Opened as `file://` (see above), no internet, or a network/CSP that blocks `openrouter.ai`. Serve over `http://localhost`. |
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
