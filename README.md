# Stammy 🗣️

**An AI speech-practice companion built on [Rasa Mantle](https://rasa.com/docs/mantle/overview).**

Stammy is a voice-and-chat agent that acts as a patient speech therapy coach. It guides people through fluency-shaping practice (easy onset, light contact, slow rate, phrasing and pausing), keeps the pace gentle, remembers how each exercise went, and wraps up every session with a short summary.

> ⚠️ **Important:** Stammy is a practice companion, **not** a medical device and **not** a replacement for a licensed speech-language pathologist (SLP). It works from speech-to-text transcripts, so it cannot reliably hear or diagnose disfluencies. Please work with a qualified SLP for assessment and treatment.

---

## Why Mantle?

Mantle runs one LLM in a loop, but the *framework* controls the loop. You describe behavior as **Skills** in plain language, then add hard guarantees only where they matter. For a therapy-style agent that means:

| Need | Mantle feature used |
|---|---|
| Gentle, plain-language coaching | `skill.md` instructions |
| Never log an attempt before an exercise was given | `tool_constraints` with `requires:` |
| Exact, reassuring wording (disclaimer, encouragement) | `responses.yml` |
| Remember technique, exercise and self-rating | `memory.yml` |
| Same agent on chat and voice | Voice channel in the Inspector |
| Slow, patient speech | `orchestrator.voice_rules` in `agent.yml` |

---

## Prerequisites

- **Python 3.10 – 3.13**
- A **free Rasa Developer Edition license** (up to 1,000 conversations/month): get one from <https://rasa.com/docs/mantle/getting-started>
- An **LLM API key** (the wizard defaults to OpenAI: `OPENAI_API_KEY`)
- For voice: a **Deepgram API key** (`DEEPGRAM_API_KEY`), the Inspector's default speech provider
- An IDE with a coding agent (Cursor or Claude Code) is recommended for the build loop
- A microphone and a modern browser for voice testing

---

## Step-by-step: install Mantle and run Stammy

### 1. Create a folder and a virtual environment

```bash
mkdir stammy && cd stammy
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

### 2. Install Mantle

Mantle ships inside the `rasa-pro` package. It is currently **pre-release**, so pin the exact version rather than letting pip pick the latest stable one:

```bash
pip install rasa-pro==3.20.0.dev6
```

> The version above is the one in the Getting Started guide at the time of writing. Check <https://rasa.com/docs/mantle/getting-started> and the [changelog](https://rasa.com/docs/mantle/changelog) for the current pin.

Verify the install:

```bash
rasa --version
```

### 3. Create the project with the setup wizard

```bash
rasa init --engine mantle
```

The wizard will:

1. Ask where to create your project (choose the current folder or a subfolder such as `stammy/`)
2. Ask for your **Rasa license key** and **LLM API key**, and write them to `.env`
3. Install Cursor and Claude Code compatible skills
4. Validate and train the starter agent
5. Open the **Inspector** so you can chat with the starter agent right away

You now have this scaffold:

```
stammy/
├── agent.yml            # identity, persona, global rules
├── integrations.yml     # LLM provider, channels, optional tracing
├── AGENTS.md            # context for your coding agent
├── .env                 # secrets written by the wizard
└── skills/
    └── view_transactions/   # sample skill, which we will replace
        ├── skill.md
        ├── memory.yml
        └── tools.py
```

### 4. Check your environment variables

Open `.env` and confirm the keys are present:

```bash
RASA_LICENSE="your-rasa-developer-edition-key"
OPENAI_API_KEY="your-llm-key"
DEEPGRAM_API_KEY="your-deepgram-key"   # add this for voice
```

To use a different LLM provider, edit the `llm:` block in `integrations.yml`. Never commit `.env` to source control.

### 5. Connect your coding agent (the Rasa Copilot)

`rasa init` already installed the skills for Claude Code and Cursor. Open the project folder in either tool. For a project **not** created with `rasa init`, install them manually:

```bash
rasa tools skills install mantle
```

### 6. Replace the sample skill with Stammy's skills

Delete `skills/view_transactions/` and create the folders below. You can write the files by hand (see [Project layout](#project-layout)) or ask the Copilot:

```text
Build a speech practice agent called Stammy. It acts as a supportive speech
therapy coach for people who stutter. Skills: a greeting with a clear
"I'm a practice companion, not a clinician" notice; a practice_exercise skill
that asks which technique to work on (easy onset, light contact, slow rate,
pausing), gives one short exercise at a time, waits patiently, gives warm
feedback, asks the user to rate how it felt, and logs the attempt; and a
wrap_up skill that summarises the session. Never diagnose or give medical
advice. Keep every reply short and encouraging.
```

Review the diff the Copilot proposes before accepting it.

### 7. Train (this is the validation gate)

```bash
rasa train
```

`rasa train` checks the whole project for undeclared memory writes, unknown response names and malformed conditions. Fix any errors it reports before debugging conversation behavior.

### 8. Chat with Stammy

```bash
rasa inspect
```

A browser window opens. Type a message such as *"Hi, I want to practise easy onset"* and step through each turn in the Inspector to see which skill was active, what memory was set and which tools ran.

### 9. Talk to Stammy (voice)

1. Make sure `DEEPGRAM_API_KEY` is set in `.env` (Deepgram is the Inspector's default for speech recognition and synthesis)
2. Restart the Inspector: `rasa inspect`
3. Leave the message box empty and click **Start voice conversation** (the waveform button)
4. Allow microphone access when your browser asks
5. Click **End conversation** to hang up

If you see **"Voice isn't set up yet"**, an ASR or TTS key is missing, or another voice startup error occurred. Set the key in `.env`, restart `rasa inspect` and check the server log.

### 10. Evaluate with simulations, then loop

Ask the Copilot to simulate a user and score the agent:

```text
Generate and run a simulation for the practice_exercise skill where the user
picks "easy onset", attempts two exercises and rates them. Assert that each
attempt is logged and the agent never gives medical advice.
```

You get a pass/fail summary with a link to the full transcript, plus recommended fixes (a missing constraint, an ambiguous instruction, a tool that should be gated). Tighten the instructions, re-run `rasa train`, re-run the simulations, inspect again and repeat until Stammy behaves the way you want.

---

## Project layout

```
stammy/
├── agent.yml
├── integrations.yml
├── .env
└── skills/
    ├── default_session_start/
    │   ├── skill.md
    │   └── responses.yml
    ├── practice_exercise/
    │   ├── skill.md
    │   ├── memory.yml
    │   ├── tools.py
    │   └── responses.yml
    └── wrap_up/
        ├── skill.md
        └── responses.yml
```

### `skills/default_session_start/skill.md`

Voice channels send `/session_start` when a call connects, so this skill lets Stammy speak first. It replaces the bundled silent skill.

```markdown
---
name: Session Start
description: Greet the user at the start of a session.
routing:
  engine_managed: true
---

:::ordered_block id=main
steps:
  - id: notice
    action: utter_practice_notice
  - id: greet
    action: utter_greet
:::
```

### `skills/default_session_start/responses.yml`

```yaml
responses:
  utter_practice_notice:
    - text: >-
        Quick note before we start. I'm a practice companion, not a clinician,
        and I can't replace a speech therapist.
  utter_greet:
    - text: Hi, I'm Stammy. There's no rush here. What would you like to practise today?
```

### `skills/practice_exercise/skill.md`

```markdown
---
name: Practice Exercise
description: Guide the user through one fluency technique exercise at a time
tool_constraints:
  - log_attempt:
      requires: >
        session.practice_exercise.current_exercise_id
        and session.practice_exercise.self_rating
---

Coach the user through speech practice. Be warm, patient and brief.
Never diagnose, never give medical advice, and never comment on how fluent
the user sounded. You only see a transcript, so you cannot judge their speech.

1. Ask which technique they want to work on: easy onset, light contact,
   slow rate, or pausing. Record their choice.
2. Call @tool.get_exercise to get one short exercise for that technique.
   Read it out and invite them to try when ready. Wait as long as they need.
3. After they try, respond with encouragement about effort, not about
   fluency. Ask how it felt: easy, okay, or hard. Record their answer.
4. Call @tool.log_attempt. Then offer another exercise or to finish for today.

If the user seems distressed, slow down, acknowledge the feeling, and
suggest speaking with their speech-language pathologist or a trusted person.
```

### `skills/practice_exercise/memory.yml`

```yaml
schema:
  public:
    technique:
      type: categorical
      enum_values: [easy_onset, light_contact, slow_rate, pausing]
      llm_settable: true
      description: The fluency technique the user chose to practise.
    self_rating:
      type: categorical
      enum_values: [easy, okay, hard]
      llm_settable: true
      description: How the user said the attempt felt.
    current_exercise_id:
      type: text
    current_exercise_text:
      type: text
    attempts_logged:
      type: text
```

### `skills/practice_exercise/tools.py`

Mock data. Swap in a real exercise bank or database as you grow the project.

```python
from rasa.mantle.tools.decorator import ToolContext, tool
from rasa.mantle.tools.result import ToolResult

_EXERCISES = {
    "easy_onset": [
        {"id": "eo_1", "text": "Take a slow breath, then say 'Hello, how are you' starting the first word very gently."},
        {"id": "eo_2", "text": "Breathe out softly and say 'Orange juice, please', letting the first sound glide in."},
    ],
    "light_contact": [
        {"id": "lc_1", "text": "Say 'Please pass the paper' touching your lips together as lightly as you can on the p sounds."},
    ],
    "slow_rate": [
        {"id": "sr_1", "text": "Say 'I would like a cup of tea' stretching each word a little longer than usual."},
    ],
    "pausing": [
        {"id": "pa_1", "text": "Say 'Tomorrow, I'm going to the market, and then home', pausing briefly at each comma."},
    ],
}


@tool(description="Get one short practice exercise for the chosen technique.")
async def get_exercise(technique: str, context: ToolContext = None) -> ToolResult:
    options = _EXERCISES.get(technique.strip())
    if not options:
        return ToolResult(llm_response={"found": False})
    exercise = options[0]
    context.memory.set("current_exercise_id", exercise["id"])
    context.memory.set("current_exercise_text", exercise["text"])
    return ToolResult(llm_response={"found": True, "exercise": exercise["text"]})


@tool(description="Log a completed practice attempt with the user's self-rating.")
async def log_attempt(exercise_id: str, self_rating: str, context: ToolContext = None) -> ToolResult:
    # Replace with a database write when you want real progress tracking.
    return ToolResult(llm_response={"logged": True, "exercise_id": exercise_id, "rating": self_rating})
```

### `skills/wrap_up/skill.md`

```markdown
---
name: Wrap Up
description: Close the session with a short summary when the user is done
---

When the user says they are finished, give a brief, kind summary of what they
practised today, remind them that regular short sessions help, and say goodbye
once. Do not ask another question after the goodbye.
```

### `agent.yml` additions

Tune how Stammy sounds. Note that setting `voice_rules` **replaces** Mantle's built-in voice rules, so keep the parts you still want. Check the [`agent.yml` reference](https://rasa.com/docs/mantle/reference/agent-yml) for the full schema and keep the identity and persona fields the scaffold generated.

```yaml
orchestrator:
  voice_rules: |
    You are on a voice call with someone practising their speech.
    Speak slowly and calmly in short sentences.
    Ask one question at a time, then wait patiently for the answer.
    Never rush the user and never finish their sentences.
    Do not use lists, markdown or any structured layout.
    Say numbers and dates the way a person would say them aloud.

fillers:
  examples:
    - user: I want to practise easy onset
      ack: Great choice. Let me find a good one for you.

rules:
  - Never diagnose a speech or language condition and never give medical advice.
  - Praise effort and persistence, never fluency.
  - When the customer says they are done, give exactly one brief goodbye and stop.
```

---

## Tune the voice for people who stutter

Defaults built for fast call-centre turn-taking can cut a speaker off during a block or a long pause. Before testing with real users, review:

- **Speech recognition end-of-speech timing.** For Deepgram, `endpointing` is the milliseconds of silence before a transcript is final (default `400`). For Deepgram Flux models, see `eot_threshold` and `eot_timeout_ms`. Longer values give the speaker more time. See [Speech providers](https://rasa.com/docs/mantle/reference/speech-providers).
- **Silence timeout.** After the silence timeout the agent checks in, and after three check-ins it says goodbye and hangs up. Reword or lengthen it for a patient pace: [Silence timeout](https://rasa.com/docs/mantle/reference/default-skills#silence-timeout).
- **Interruptions and backchannels.** Decide whether the agent should stop when the user talks over it: [Interruptions](https://rasa.com/docs/mantle/reference/integrations-yml#interruptions).

Change one setting at a time and re-test out loud.

---

## Troubleshooting

| Symptom | What to check |
|---|---|
| `rasa: command not found` | Activate the virtual environment (`source .venv/bin/activate`) |
| Wrong or unexpected Rasa version | Reinstall the pinned version: `pip install rasa-pro==3.20.0.dev6` |
| `rasa train` fails with `undeclared_memory_write` | Add the key to the skill's `memory.yml` |
| A tool never appears | Check the full namespaced condition, e.g. `session.practice_exercise.self_rating`; the Inspector tools panel shows each tool's gate result |
| "Voice isn't set up yet" | Set `DEEPGRAM_API_KEY` in `.env`, restart `rasa inspect`, check server logs |
| Chat works but you are not heard | Check microphone permission, then ASR connection errors |
| Agent replies in text but no speech | Check the TTS connection, model/voice identifier and provider access |
| Changes to skills not showing | Run `rasa train` again; channel settings only need an Inspector restart |

---

## Roadmap

- [ ] Real progress tracking (database-backed `log_attempt`, streaks, weekly summaries)
- [ ] A `references/` folder with SLP-approved technique explanations for grounded answers
- [ ] Confirmation (`requires_confirmation`) before sharing progress reports
- [ ] Per-language voices via `language_map`
- [ ] Phone line (telephony) channel
- [ ] Observability with OpenTelemetry traces and metrics

---

## Learn more

- Rasa docs home: <https://rasa.com/docs>
- Getting Started: <https://rasa.com/docs/mantle/getting-started>
- Skills: <https://rasa.com/docs/mantle/skills>
- Tool constraints: <https://rasa.com/docs/mantle/build-guide/tool-constraints>
- Voice agents: <https://rasa.com/docs/mantle/voice/overview>
- Designing for voice: <https://rasa.com/docs/mantle/voice/designing-for-voice>
- Testing voice: <https://rasa.com/docs/mantle/voice/testing>
- Speech providers: <https://rasa.com/docs/mantle/reference/speech-providers>

## License & disclaimer

Stammy is an educational project. It is not medical advice and not a substitute for professional speech-language pathology care. Use of Rasa Mantle is subject to the [Rasa Developer Edition terms](https://rasa.com/developer-terms).
