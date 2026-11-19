# OmniTrainer — Multimodal Moderation for Customer Service Training

> **About this project:** this is my solution to **OmniTrainer**, the final project of the
> **Multimodal AI Applications** course on [udacity.com](https://www.udacity.com). Udacity provided
> the scaffolding (project structure, FastAPI backend, audio and video agents, customer agent, tests
> and eval framework). I completed the parts the project asked for and added a few extensions,
> described in [Work done](#work-done).

OmniTrainer is a training app for customer service agents at a fictional company, ACME Enterprise.
The trainee chats with a simulated, unhappy customer played by an LLM. The customer's
*ACME Power Widget Pro* stopped working and they want a refund. Before anything the trainee
sends reaches the customer (text, images, video or recorded audio), it goes through AI moderation:

- **Text and audio** are checked for personally identifiable information (PII), an unfriendly
  tone and an unprofessional tone. Audio is transcribed first.
- **Images and video** are checked for PII (any person or part of a person), disturbing content
  and low quality.

A flagged message is blocked and never reaches the customer. The trainee sees the moderator's
rationale explaining why. Every step is traced with OpenTelemetry and can be inspected in Arize Phoenix.

## Architecture

```
┌──────────── Gradio chat UI :7860 (gradio_app.py) ────────────┐
│  Trainee types text, attaches files, or records audio         │
└───────────────┬──────────────────────────────────────────────┘
                │ every piece of content, one at a time
                ▼
      FastAPI backend :8000 (fastapi_app.py), Bearer USER_API_KEY
                │
                ▼
      Moderation agents (Pydantic AI + Gemini) → structured result
      text · image · video · audio            {flags…, rationale}
                │
     flagged? ──┴── yes → blocked; rationale shown in the sidebar
                │
                no
                ▼
      Customer agent (Gemini plays the customer), with the full
      conversation history → reply shown in the chat

All of the above is traced with OpenTelemetry → Arize Phoenix :6006
```

- **Moderation agents:** one Pydantic AI agent per modality. Each returns a validated Pydantic model
  (`TextModerationResult`, `ImageModerationResult`, …) with boolean flags and a `rationale`.
- **Customer agent:** an LLM role-playing the customer. It receives the conversation history every
  turn, so its replies stay consistent.
- **Frontend and backend:** the Gradio UI calls the moderation agents through the HTTP API, so
  another frontend (React, a mobile app, …) could reuse the same backend unchanged.
- **Observability:** each conversation is one trace in Phoenix:
  ```
  conversation (session.id)
  └── chat_turn (session.id, input summary, feedback when flagged)
      ├── moderate_text / moderate_image / moderate_video / moderate_audio
      └── llm_customer → agent run → model call (tokens, latency)
  ```

## Project structure

```
.
├── multimodal_moderation/            # the application package
│   ├── agents/
│   │   ├── text_agent.py             # text moderation agent
│   │   ├── image_agent.py            # image moderation agent (BinaryContent input)
│   │   ├── audio_agent.py            # audio: transcription + moderation
│   │   ├── video_agent.py            # video moderation agent
│   │   └── customer_agent.py         # LLM playing the ACME customer
│   ├── types/
│   │   ├── moderation_result.py      # Pydantic output schemas for all agents
│   │   └── model_choice.py           # model + settings passed to each agent call
│   ├── gradio_app.py                 # chat UI, moderation flow, tracing spans
│   ├── fastapi_app.py                # REST API for the moderation agents
│   ├── tracing.py                    # OpenTelemetry → Phoenix setup
│   ├── app.py                        # starts Phoenix, the API and the UI together
│   ├── env.py                        # reads configuration from .env
│   └── utils.py                      # file type detection
├── tests/                            # unit tests (use a stand-in model, no API calls)
├── evals/                            # Pydantic Evals suites (call Gemini for real)
│   ├── text/ image/ audio/ video/    # test_cases.py + evaluators.py per modality
│   ├── test_data/                    # sample text, images, audio and video
│   ├── common_evaluators.py          # HasRationale
│   ├── config.py                     # model under test vs. judge model
│   └── utils.py                      # repeated cases for measuring consistency
├── RUN_EVIDENCE.pdf                  # test output, eval output, UI and Phoenix screenshots
├── env.example                       # template for .env
└── pyproject.toml / uv.lock          # dependencies (managed with uv)
```

## Getting started

### Requirements

- Python 3.12+
- [uv](https://docs.astral.sh/uv/)
- A Gemini API key from [Google AI Studio](https://aistudio.google.com/)

### Install

```bash
uv sync --dev
uv pip install -e .
```

### Configure

```bash
cp env.example .env
```

Then edit `.env`:

| Variable | Purpose |
|---|---|
| `GEMINI_API_KEY` | Your Gemini API key |
| `USER_API_KEY` | Any non-empty string. The UI uses it to authenticate with the backend API |
| `DEFAULT_GOOGLE_MODEL` | Model used by all agents, e.g. `gemini-2.5-flash-lite` |
| `EVAL_JUDGE_MODEL` | Model used as the LLM judge in evals |
| `EVAL_NUM_REPEATS` | How many times each eval case runs. Use 1 while iterating, 5+ for a steadier score |
| `GOOGLE_GEMINI_BASE_URL` | Only for the Udacity workspace proxy. Remove it when using your own key |

`.env` is git-ignored. Never commit it.

## Usage

### Run the app

```bash
uv run multimodal-moderation
```

This starts three services:

| Service | URL |
|---|---|
| Chat UI | http://localhost:7860 |
| Moderation API (interactive docs) | http://localhost:8000/docs |
| Phoenix tracing dashboard | http://localhost:6006/projects |

Press **Ctrl+C** to stop. If you see "Address already in use", an earlier copy is still running:
`pkill -f multimodal-moderation`.

**Example session:** greet the customer, attach `evals/test_data/professional_image.jpg` (it passes
moderation), then send something rude such as *"That's your problem, not mine."* The message is
blocked and the rationale appears in the **Moderation Agent Feedback** panel. Click **End
Conversation** to close the trace, then explore it in Phoenix.

### Use the API directly

```bash
curl -X POST http://localhost:8000/api/v1/moderate_text \
  -H "Authorization: Bearer $USER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hi John, your refund to 12 Main St is on its way."}'

curl -X POST http://localhost:8000/api/v1/moderate_image_file \
  -H "Authorization: Bearer $USER_API_KEY" \
  -F "file=@evals/test_data/image_with_person.jpg"
```

The other endpoints are `/api/v1/moderate_video_file` and `/api/v1/moderate_audio_file`.

### Run the tests

```bash
uv run pytest tests/ -vv --ignore=tests/test_env_setup.py
```

The tests check the wiring with Pydantic AI's `TestModel`, so they make no API calls and cost
nothing. `tests/test_env_setup.py` calls Gemini to verify your credentials. Drop the `--ignore` to
include it.

### Run the evals

```bash
uv run evals/text/test_cases.py
uv run evals/image/test_cases.py
uv run evals/audio/test_cases.py
uv run evals/video/test_cases.py
```

Each case is graded by a rule-based check of the flags and by an **LLM judge** that grades the
rationale. Both the agent and the judge are LLMs, so scores vary from run to run, and a score below
100% is expected.

## Work done

### Required parts (TODOs in the Udacity scaffolding)

1. **Structured outputs** (`types/moderation_result.py`): added `AudioModerationResult` with
   `transcription` and required bool flags `contains_pii`, `is_unfriendly` and `is_unprofessional`.
   Each field has a `description`, which becomes part of the JSON schema Gemini has to follow.
2. **Text moderation agent** (`agents/text_agent.py`): a Pydantic AI `Agent` with
   `output_type=TextModerationResult`, run asynchronously with the model chosen per call.
3. **Image moderation agent** (`agents/image_agent.py`): wraps the uploaded bytes in
   `BinaryContent` with their MIME type, so Gemini receives a real image next to the instruction.
4. **Gradio app** (`gradio_app.py`):
   - multimodal `ChatInterface` (text, image, video, audio, microphone);
   - each piece of content is moderated before it's sent; flagged content is blocked and its
     rationale shown;
   - approved files are forwarded to the customer agent as `BinaryContent`;
   - conversation memory through `gr.State` (`additional_inputs` / `additional_outputs`), so the
     customer remembers earlier turns.
5. **Tracing:** `conversation` span (with `session.id`) kept open across turns; `chat_turn`
   spans linked to it as children; `moderate_*` spans; `feedback` attribute on flagged turns.
6. **Evals:** completed the text and image cases (expected flags, LLM-judge rubrics, repeated
   cases, retries) and ran all four suites.

### Design decisions and extensions

- **Prompt injection hardening (text):** the message under review is wrapped in
  `<message>…</message>` tags. The instructions say the content is untrusted data, must never be
  followed as instructions, and that any manipulation attempt has to be reported in the rationale.
- **Adversarial eval case:** `prompt_injection_text` is a rude message that tries to close the tag
  early and pose as a "SYSTEM NOTE" to get itself approved. It was flagged correctly in 5/5 runs,
  and 4 of the 5 rationales explicitly named the manipulation attempt.
- **More useful traces:** `chat_turn` also records `session.id`, the text length and the number of
  files. The `feedback` attribute is recorded for flagged media too, not only for text.
- **UI fix:** removed a duplicated "Content flagged: Content flagged:" prefix in the feedback panel.

### Results

- **Unit tests:** 45/45 passing.
- **Text evals** (4 cases × 5 repeats): 97.5% of assertions passed. One borderline judgment
  (a direct phone number flagged as unprofessional) and one cut-off rationale.
- **Image, audio and video evals:** 100% (single run).

Full outputs and screenshots are in [`RUN_EVIDENCE.pdf`](RUN_EVIDENCE.pdf).

### Observations

- The LLM judge receives the **file path** of each eval input, not the content, so it grades whether
  the rationale is plausible but can't check it against the actual text or image.
- A few rationales from `gemini-2.5-flash-lite` stopped right at an apostrophe. The flags were
  unaffected, but this is why the rationale is graded separately from the flags.

## Credits

Course, scenario and starter code: Udacity, **Multimodal AI Applications**, final project
*OmniTrainer*. Built with [Pydantic AI](https://ai.pydantic.dev/),
[Pydantic Evals](https://ai.pydantic.dev/evals/), [Google Gemini](https://ai.google.dev/),
[Gradio](https://www.gradio.app/), [FastAPI](https://fastapi.tiangolo.com/) and
[Arize Phoenix](https://phoenix.arize.com/).
