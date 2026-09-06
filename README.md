<div align="center">

# Sanad · سند

### An AI support companion for autistic users — built to help, never to diagnose

*Sanad (Arabic: **سند** — "support", the person you lean on)*

[![n8n](https://img.shields.io/badge/n8n-Workflow%20Orchestration-EA4B71)](https://n8n.io)
[![LangChain](https://img.shields.io/badge/LangChain-AI%20Chains-1C3C3C)](https://www.langchain.com/)
[![OpenRouter](https://img.shields.io/badge/OpenRouter-Qwen%20LLM%20%2B%20VLM-6467F2)](https://openrouter.ai)
[![ElevenLabs](https://img.shields.io/badge/ElevenLabs-STT%20%2B%20TTS-000000)](https://elevenlabs.io)
[![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E)](https://supabase.com)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](LICENSE)

</div>

---

## The problem

Autistic users — especially Arabic-speaking ones — are underserved by mainstream mental-health and support apps. The tools that do exist are usually English-only, visually overstimulating, and framed clinically: they talk *about* the user as a diagnosis rather than *to* them as a person.

Sanad takes the opposite approach. It is a calm, bilingual (Egyptian Arabic / English) companion that a user can talk to by **text, voice, or photo**, get a grounded reply in their own dialect, run a short self-check, receive tailored calming and learning activities, and reach a caregiver in one tap when things get overwhelming.

Two rules shaped every design decision in this project:

> **1. It is a support tool, not a diagnostic tool.**
> **2. The assistant never labels the user.**

The second rule is enforced in the system prompts themselves — the model is explicitly forbidden from using the words *autism, disorder, diagnosis, anxiety, depression, therapy, medication, treatment*. The user gets warmth and practical help, not a clinical vocabulary applied to them.

---

## The app

<div align="center">

| Onboarding (EN) | Onboarding (AR — full RTL) |
|:---:|:---:|
| <img src="docs/app/01-onboarding-en.png" width="330"> | <img src="docs/app/02-onboarding-ar.png" width="330"> |

| Home — AI recommendations | Home (AR) |
|:---:|:---:|
| <img src="docs/app/03-home-en.png" width="330"> | <img src="docs/app/04-home-ar.png" width="330"> |

| Self-check assessment | AI companion (text + voice) |
|:---:|:---:|
| <img src="docs/app/05-assessment.png" width="330"> | <img src="docs/app/06-ai-chat.png" width="330"> |

| Calm mode | Reach a caregiver |
|:---:|:---:|
| <img src="docs/app/07-calm-mode.png" width="330"> | <img src="docs/app/08-contact-caregiver.png" width="330"> |

</div>

**Design choices that matter for this audience:** a low-stimulation warm palette instead of saturated brand colours; one clear action per card; large touch targets; predictable bottom navigation that never reorders; every AI-generated surface explicitly badged *AI Assisted*; a persistent "support tool, not a diagnostic tool" disclaimer; and a complete right-to-left Arabic mirror of every screen — not a translation layer bolted on, but the same experience in both directions.

<details>
<summary>More screens — activities, progress, settings</summary>

<div align="center">

| All activities | Progress |
|:---:|:---:|
| <img src="docs/app/09-activities.png" width="330"> | <img src="docs/app/10-progress.png" width="330"> |

| Settings (EN) | Settings (AR) |
|:---:|:---:|
| <img src="docs/app/11-settings-en.png" width="330"> | <img src="docs/app/12-settings-ar.png" width="330"> |

</div>

Every AI capability — image description, speech-to-text, text-to-speech, contextual help — is an individual toggle the user controls, and the TTS voice type and speed are user-configurable. Nothing AI-powered is forced on someone who finds it overwhelming.

</details>

---

## System architecture

The AI layer is a single **38-node n8n workflow** that acts as the brain behind the app: one webhook in, three routed pipelines, multimodal input handling, stateful memory, and a voice response path.

```mermaid
flowchart TD
    A[Client app] -->|POST /sanad-omni| B[Webhook]
    B --> C[Master Router]
    C --> D{Main Route Switch}

    D -->|answers present| E[Assessment]
    D -->|profile only| F[Recommendations]
    D -->|message / audio / image| G[Chat]

    E --> E1[Calc Social]
    E --> E2[Calc Sensory]
    E --> E3[Calc Support]
    E1 & E2 & E3 --> E4[Build Profile JSON]
    E4 --> E5[(Supabase · user_profiles)]
    E5 --> E6[Respond]

    F --> F1[Analyze Needs]
    F1 --> F2[Match activities<br/>+ fallback set]
    F2 --> F3[Respond]

    G --> G1{Auto-Detect Input}
    G1 -->|audio| H1[ElevenLabs STT]
    G1 -->|image| H2[Qwen2.5-VL-72B<br/>vision chain]
    G1 -->|text| H3[Format context]
    H1 & H3 --> I[(Supabase · chat_history)]
    I --> J[Intent Classifier]
    J --> K{Intent Switch}
    K -->|recommendations| F1
    K -->|distress| L1[Immediate support]
    K -->|conversation| L2[Qwen3.6-Plus<br/>chat chain]
    H2 & L1 & L2 --> M[Merge outputs]
    M --> N{TTS requested?}
    N -->|yes| O[ElevenLabs TTS]
    N & O --> P[Respond to client]
    M --> Q[(Supabase · chat_history)]
```

<div align="center">
<img src="docs/architecture/workflow-canvas.png" width="900">
<br><em>The full workflow on the n8n canvas</em>
</div>

### How the three routes work

**1 · Assessment — deterministic, not generative.**
The self-check answers are scored by plain JavaScript (`Calc Social`, `Calc Sensory`, `Calc Support`), not by an LLM. Each dimension sums its question scores and maps to a level, then the three combine into a structured profile written to Supabase. This is deliberate: a user's profile must be **reproducible** — the same answers must always yield the same profile — and language models are the wrong tool for that. LLMs are used where they add value (conversation, vision), and kept away from where consistency matters more than fluency.

**2 · Recommendations — profile-aware, with a guaranteed floor.**
`Analyze Needs` reads the stored profile and selects activities matching the user's needs. If the database or an upstream API fails, the node falls back to an embedded activity set instead of erroring — a user in distress opening the app must never be met with a blank screen. Degraded output beats no output.

**3 · Chat — multimodal in, multimodal out.**
`Auto-Detect Input` inspects the payload and its binary attachments to classify the request as text, audio, or image, then routes accordingly: audio through ElevenLabs speech-to-text, images through a Qwen2.5-VL-72B vision chain, text straight through. All paths converge, load prior conversation from Supabase so the assistant has memory across turns, and pass through an intent classifier that separates ordinary conversation from a request for activities and from signs of distress — the last of which is routed to an immediate-support response rather than open-ended chat. Replies can be spoken back through ElevenLabs TTS when the user has voice enabled.

### Responsible-AI design

This is the part of the project I care about most, because the user group is vulnerable:

- **No diagnostic or medical claims.** The system prompts forbid the model from discussing diagnosis, conditions, therapy or medication, and forbid it from giving medical advice.
- **No labelling.** The assistant never tells the user anything about their condition; it simply adapts how it responds.
- **Crisis escalation is scripted, not improvised.** If a user mentions self-harm, the model is instructed to direct them to a trusted adult or emergency services — it does not attempt to counsel them. The app pairs this with a one-tap *Contact Caregiver* path.
- **Distress bypasses the chatbot.** Detected distress routes to a dedicated support response, so the most sensitive moments do not depend on free-form generation.
- **Tone follows the user, not a script.** The assistant only shifts into a comforting register when the user actually expresses difficulty; casual conversation stays casual, which avoids the patronising feel that makes many support apps unusable.
- **User-controlled AI.** Every AI feature is a toggle the user owns.

### Cultural and linguistic engineering

The assistant is prompted to *think* in Egyptian Arabic rather than translate into it — the prompt explicitly forbids carrying English idioms across, and specifies natural conversational structure and length limits. Getting a model to sound like a warm Egyptian friend rather than a translated chatbot took considerably more prompt iteration than the routing logic did, and it is the difference between an app that feels local and one that feels imported.

Two models are used for two jobs: `qwen2.5-vl-72b-instruct` for image understanding (short, tightly bounded responses), and `qwen3.6-plus` for conversation (longer, warmer). ElevenLabs `eleven_multilingual_v2` handles both directions of voice.

---

## My contribution

I designed and built the **entire AI orchestration layer** — everything in this repository:

- The 38-node n8n workflow: routing architecture, all branch logic and the intent classification layer
- All custom JavaScript nodes: input detection, deterministic profile scoring, context formatting, response shaping, fallback handling
- Prompt engineering for the conversational and vision chains, including the safety constraints and the Egyptian Arabic persona
- Multimodal pipeline integration: ElevenLabs STT/TTS, OpenRouter vision and chat models via LangChain nodes
- Supabase schema and data flow for `user_profiles` and `chat_history`, plus the stateful memory design
- Self-hosting the n8n instance on a VPS (Docker, Nginx)

**Built by the team, not by me:** the Flutter mobile application and the Django REST backend were built by my teammates. Their code is not included in this repository — what you see here is my own work. The app screenshots appear only to show the system this AI layer was built for, and full credit for the app and backend belongs to them.

Sanad began as our graduation project at South Valley National University.

---

## Repository contents

```
.
├── workflow/
│   └── sanad_omni_workflow.json     # the complete n8n workflow — importable
├── docs/
│   ├── architecture/                # workflow canvas + node detail captures
│   └── app/                         # application UI screenshots
├── LICENSE
└── README.md
```

## Running the workflow

**Prerequisites** — an n8n instance (cloud or self-hosted), plus credentials for:

| Service | Used for |
|---|---|
| Supabase | `user_profiles` and `chat_history` tables |
| OpenRouter | Qwen chat + vision models |
| ElevenLabs | speech-to-text and text-to-speech |

**Import**

1. In n8n: **Workflows → Import from File** → select `workflow/sanad_omni_workflow.json`
2. Open each credential-bearing node and connect your own Supabase, OpenRouter and ElevenLabs credentials
3. Create the two Supabase tables the workflow expects (`user_profiles`, `chat_history`)
4. Activate the workflow and take the production webhook URL from the **Sanad Omni Webhook** node

**Calling it** — `POST` to the webhook. The Master Router picks the pipeline from the payload shape:

```jsonc
// Chat
{ "session_id": "...", "profile_id": "...", "message": "..." }

// Chat with voice or image — send audio_data / image_data as binary
{ "session_id": "...", "profile_id": "...", "audio": true }

// Assessment
{ "profile_id": "...", "answers": { "q1_social_interaction": 3, "q2_eye_contact": 2, ... } }

// Recommendations
{ "profile_id": "...", "type": "recommendation", "user_profile": { ... } }
```

> **Note:** the workflow ships with webhook authentication set to `None` for local testing. Enable authentication on the webhook node before exposing an instance publicly.

## Project status

Complete and functional as a graduation project — workflow, mobile app and backend all working together, and self-hosted on a VPS during development. There is no public live demo: the endpoint depends on private API credentials, so the workflow is published here to be read and imported rather than called.

## License

[MIT](LICENSE) © Omar Mohsen
