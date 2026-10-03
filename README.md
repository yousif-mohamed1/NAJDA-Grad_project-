<div align="center">

# NAJDA — AI Track (Graduation Project)

**Network for AI-powered Joint Dispatch and Assistance**

AI/ML features for the [NAJDA](https://github.com/youssefry01/najda) emergency dispatch platform.

</div>

---

## What this repo covers

This is the **AI track** of the NAJDA graduation project — a standalone Python service that powers all machine learning and AI features for the main NAJDA platform. It plugs into the existing Spring Boot backend via REST and WebSocket.

### Core Features

| Feature | Phase | Status |
|---------|-------|--------|
| 🤖 **Emergency Chatbot (Intake)** — guided citizen reporting via conversational AI | Phase 3 | ⬜ Not started |
| 🤖 **Emergency Chatbot (First-Aid)** — step-by-step first-aid guidance while waiting | Phase 4 | ⬜ Not started |
| 🎯 **Unit Recommendation** — rank available units by ETA, type, load | Phase 2 | ⬜ Not started |
| ⏱️ **ETA Prediction** — correct OSRM estimates with learned features | Phase 2 | ⬜ Not started |
| 🏥 **Hospital Recommendation** — match by facility type, distance, load | Phase 2 | ⬜ Not started |
| 🩺 **Vitals Early-Warning** — NEWS2 + ML trends for patient monitoring | Phase 4 | ⬜ Not started |
| 📋 **SBAR Handoff Summary** — LLM-generated handoff from timeline + vitals | Phase 4 | ⬜ Not started |
| 📸 **Photo/Video Severity** — vision model for incident severity scoring | Phase 3 | ⬜ Not started |
| 🗺️ **Demand Forecasting Heatmap** — spatial-temporal incident prediction | Phase 4 | ⬜ Not started |

See [`docs/AI_TIMELINE.md`](docs/AI_TIMELINE.md) for the full plan, architecture, and cut-line priorities.

## Architecture

```
┌─────────────────┐         ┌──────────────────────┐
│  NAJDA Backend   │◄──REST──►│  AI Service (this)    │
│  (Spring Boot)   │   + WS   │  (Python / FastAPI)   │
└─────────────────┘          └──────┬───────────────┘
                                    │
                        ┌───────────┼───────────────┐
                        │           │               │
                   OpenRouter   sentence-       Whisper
                   (LLM via     transformers    (speech)
                    locked      (retrieval)
                    prompt)
```

### Emergency Chatbot — How it works

The chatbot is **NOT a general-purpose LLM**. It uses Retrieval-Augmented Generation (RAG):

1. **Knowledge Base** — curated emergency intake flows + first-aid protocols (Markdown/JSON)
2. **Retrieval** — `sentence-transformers` finds the relevant protocol
3. **Generation** — OpenRouter LLM with a locked system prompt that can ONLY use KB content
4. **Guardrails** — hard refusal for anything outside the emergency domain

## Tech Stack

- **Python 3.11+** / **FastAPI**
- **sentence-transformers** — embedding & retrieval
- **OpenRouter** — LLM generation (same provider as the main backend)
- **scikit-learn / XGBoost** — ETA prediction, ranking models
- **Whisper** — Arabic speech-to-text (optional)
- **PyTorch / torchvision** — vision severity classification

## Getting Started

> 🚧 Setup instructions will be added as the service is built.

```bash
# Coming soon
pip install -r requirements.txt
uvicorn app.main:app --reload
```

## Related

- [Main NAJDA Repository](https://github.com/youssefry01/najda) — the full platform (backend, web, mobile)
- [`docs/AI_TIMELINE.md`](docs/AI_TIMELINE.md) — detailed AI features timeline and plan

---

*Graduation Project · October 2025 – June 2026*
