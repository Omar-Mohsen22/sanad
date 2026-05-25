# Sanad Omni: Multi-Modal AI Support Assistant Orchestrator

Sanad Omni is an advanced, production-grade omnichannel backend orchestrator built entirely on **n8n**. It acts as a highly adaptive, multi-modal AI assistant designed to deliver context-aware support. Utilizing a complex routing pipeline, it dynamically processes text, audio, and visual inputs, maintains stateful conversation history, and handles user profiling with seamless database synchronization.

---

## 🛠️ System Architecture

The workflow leverages a modular architecture that cleanly separates intent classification, heavy analytical processing, external LLM orchestration, and multi-media file handling.

[Sanad Omni Architecture Blueprint] <img width="1587" height="647" alt="Screenshot 2026-05-24 182041" src="https://github.com/user-attachments/assets/a2ba4c57-c51b-4d0d-b2ae-d34d21fa0760" />


### Core Workflow Engines:
1. **Dynamic Input Router:** Automatically intercepts incoming payloads via webhooks, identifying binary objects (images/audio) vs. standard prose.
2. **Algorithmic Profiling & Assessment Node:** Computes behavioral, social, and sensory metrics dynamically before storing structured JSON profiles.
3. **Intent Classifier & Switch Layer:** Evaluates whether requests require general chatbot conversation, fallback emergency support, or specific item/activity recommendations.
4. **Stateful Chat History Engine:** Syncs and sorts asynchronous interactions to feed context back into LLM memory context windows.

---

## ✨ Key Features

* **Multi-Modal Input Detection:** Processes multi-media payloads. It automatically bridges audio streams via ElevenLabs Speech-to-Text and routes image data to Vision-Language models.
* **Context-Aware Egyptian Persona ("Sanad"):** Powered by LangChain and OpenRouter (`qwen/qwen3.6-plus` / `qwen/qwen2.5-vl-72b-instruct`), engineered to dynamically adjust tone according to user sentiment.
* **Granular Profile Scoring:** Features automated JavaScript modules (`Calc Social`, `Calc Sensory`, `Calc Support`) to score, categorize, and build data structures on the fly.
* **Database Syncing (Supabase RPC):** Real-time read/write syncing utilizing relational mapping for structured chat logs (`chat_history`) and assessment metrics (`user_profiles`).
* **Robust Fail-safes & Fallbacks:** If a database transaction or API boundary breaches, the architecture defaults gracefully to embedded fallback arrays ensuring 100% service availability.

---

## 🎛️ Technology Stack

* **Orchestration:** n8n (Advanced Workflow Automation)
* **AI Framework:** LangChain Integration (n8n native blocks)
* **LLM Providers:** OpenRouter (Qwen Multimodal & Large Language Systems)
* **Voice Intelligence:** ElevenLabs (TTS & STT Pipelines)
* **Database & BaaS:** Supabase (PostgreSQL with Remote Procedure Calls)

---

## 🚀 Deployment & Installation

### Prerequisites
Ensure you have an active **n8n** instance (Cloud or self-hosted Docker container) and API access tokens for:
* Supabase (Database URL, Service/Anon Keys)
* OpenRouter API
* ElevenLabs API

