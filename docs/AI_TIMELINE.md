# NAJDA — AI Features Timeline

> **Network for AI-powered Joint Dispatch and Assistance**
> Graduation project · AI team plan
> Start: **Early October 2025** → Final defense: **~June 2026**

---

## How to read this document

- ✅ = done &nbsp;·&nbsp; 🔧 = in progress &nbsp;·&nbsp; ⬜ = not started
- **[Edit]** markers flag suggested changes or additions to the original plan
- The [Cut Line](#cut-line--if-behind-schedule) at the bottom tells you what to drop first if time runs short

---

## Overview

| Phase | Weeks | Period | Focus |
|-------|-------|--------|-------|
| [Phase 0](#phase-0-foundation-weeks-1-2-early-oct) | 1–2 | Early Oct | Tracks, AI service, data contracts, logging |
| [Phase 1](#phase-1-data--evaluation-weeks-3-6-oct-nov) | 3–6 | Oct–Nov | Simulator, datasets, evaluation harness |
| [Phase 2](#phase-2-dispatch-intelligence-weeks-7-12-nov-dec) | 7–12 | Nov–Dec | Unit recommendation, ETA, hospital recommendation |
| [Phase 3](#phase-3-intake-intelligence-weeks-13-18-jan-feb) | 13–18 | Jan–Feb | Arabic extraction, voice, vision, false reports, clustering |
| [Phase 4](#phase-4-medical-forecasting--citizen-weeks-19-23-feb-mar) | 19–23 | Feb–Mar | Vitals, handoff, heatmap, first-aid, translation |
| [Phase 5](#phase-5-integration--delivery-weeks-24-30-mar-may) | 24–30 | Mar–May | Testing, evaluation, thesis, demo, rehearsals |

---

## Phase 0: Foundation (weeks 1-2, early Oct)

> **Goal:** Set the ground rules so every later feature plugs in cleanly.

- [ ] **Split the AI team into 3 tracks**
  - **Dispatch ML** — unit recommendation, ETA, forecasting, hospital recommendation
  - **Language & Vision** — Arabic NLP, speech, image/video, translation, first-aid assistant
  - **Medical & Data** — vitals, handoff summaries, synthetic data, evaluation
- [ ] **Build one AI service** (Python / FastAPI) that the Spring Boot backend calls over REST
  - Keep OpenRouter behind it → one interface, one log format, one fallback
  - > **[Edit]** The backend already uses OpenRouter for AI priority + duplicate suggestion (see [`README.md`](../README.md) architecture diagram). Define clearly which calls stay in Spring Boot vs. migrate to the new Python service. Avoid two AI gateways doing overlapping work.
- [ ] **Define a data contract for every feature**
  - Input schema, output schema, confidence score, explanation text
  - > **[Edit]** Write these as OpenAPI specs from day one. The backend already has REST endpoints — aligning contracts early avoids a painful rewrite in Phase 5.
- [ ] **Add suggestion logging in the backend**
  - Events: `shown`, `accepted`, `dismissed`, `overridden`
  - All later evaluation depends on this — do not skip
  - > **[Edit]** The backend already has WebSocket push for incident/mission/unit state. Piggyback suggestion events onto the same channel so the dispatcher console can show acceptance rates live.
- [ ] **Decide which features may see decrypted content**
  - Server-side only (the backend already uses AES-256-GCM at rest for chat, descriptions, clinical notes)
  - > **[Edit]** Document this decision in a `docs/AI_DATA_ACCESS_POLICY.md` — the thesis reviewers will ask.

---

## Phase 1: Data & Evaluation (weeks 3-6, Oct-Nov)

> **Goal:** No model is useful without data to train on and metrics to judge it by. Protect these weeks.

- [ ] **Build the Cairo incident simulator**
  - Incidents by type, hour, and district
  - Shift patterns and unit positions
  - Hospital loads
  - Synthetic vitals streams
  - > **[Edit]** Align the simulator's data model with the existing `Facility` entity (unified hospitals, fire/police/ambulance stations with `facilityType`). The backend already has `registered` vs. geographic-only facilities — the simulator should respect this distinction.
- [ ] **Collect / generate labeled data**
  - Arabic / Egyptian-dialect emergency reports
  - Fire, crash, and injury images from public datasets
  - Audio samples for speech-to-text
  - > **[Edit]** Start a `data/` top-level directory with a clear `README.md` explaining provenance, licensing, and preprocessing steps for every dataset. Add it to `.gitignore` if the data is too large for Git, and document how to fetch it.
- [ ] **Build the evaluation harness**
  - Baselines (simple rules: nearest unit, average ETA)
  - Metrics per feature
  - One Jupyter notebook per feature
  - > **[Edit]** Add a `Makefile` or `justfile` in the AI service root so `make eval` runs every notebook end-to-end. Reproducibility matters for the thesis.

> **Deliverable:** One reproducible dataset + a benchmark script that every later feature plugs into.

---

## Phase 2: Dispatch Intelligence (weeks 7-12, Nov-Dec)

> **Goal:** The dispatcher console shows AI suggestions end-to-end, with accept/override logging.

### Unit Recommendation (weeks 7-9)

- [ ] Rank on-shift units by ETA, type, load, and capability
- [ ] Include a "why" explanation string
- [ ] Start rule-based, then add a learned ranker
- > **[Edit]** The backend already enforces "shifts gate everything" — a unit can't receive a mission unless someone is on shift as `LEAD`. The recommendation model must query shift state, not just unit existence. Wire this into the existing shift service rather than duplicating the logic.

### ETA Prediction (weeks 9-11)

- [ ] Correct OSRM ETAs using hour-of-day and area features
- [ ] Gradient boosting is enough — don't over-engineer
- > **[Edit]** The backend already calls OSRM for routing. Log raw OSRM ETAs alongside actual arrival times from day one (Phase 0 logging!) — this becomes your training data for free.

### Hospital Recommendation (weeks 11-12)

- [ ] Match by facility type, distance, and current load
- > **[Edit]** The `Facility` model already unifies all facility types. Hospital load data will need a new endpoint or a WebSocket channel — coordinate with the backend team early.

> **🏁 Milestone (end of December):** Dispatcher console shows suggestions end-to-end with accept/override logging.

---

## Phase 3: Intake Intelligence (weeks 13-18, Jan-Feb)

> ⚠️ **Exams likely fall in January** — keep early January light: integration and bug fixes only.

### 🤖 Emergency Chatbot — Intake Mode (weeks 13-15)

> **[New feature]** A conversational chatbot that helps citizens report emergencies step-by-step. NOT a general-purpose LLM — strictly scoped to emergency intake and first-aid guidance.

#### Architecture

| Component | Tech | Purpose |
|-----------|------|---------|
| Knowledge base | Curated emergency intake flows + first-aid protocols (Markdown/JSON in `data/kb/`) | Every response is grounded — no hallucination |
| Retrieval | `sentence-transformers` (e.g. `all-MiniLM-L6-v2`) in the Python AI service | Finds the right protocol/flow for the citizen's situation |
| Generation | OpenRouter LLM with a **locked system prompt** | Natural bilingual conversation, but can ONLY use KB content |
| Intent classifier | Rule-based first, small classifier later if needed | Routes between: intake → first-aid → out-of-scope refusal |
| Guardrails | Hard refusal + "I can only help with emergencies" | Blocks anything outside the emergency domain |

#### Intake conversation flow

- [ ] **Build the intake question tree** — structured JSON flows per emergency type:
  - Medical: What happened? → Conscious? → Breathing? → Bleeding? → Location
  - Fire: What's burning? → People trapped? → Floor/building? → Location
  - Accident: Vehicles involved? → Injuries? → Road blocked? → Location
  - Crime: Type? → Ongoing? → Weapon? → Location
- [ ] **Conversational extraction** — the chatbot asks follow-up questions naturally, extracts structured fields (type, severity, location, victim count) as the citizen talks
- [ ] **Auto-create incident** — once enough fields are extracted, the bot creates an `Incident` via the backend API with the citizen's confirmation
- [ ] **Bilingual support** — Arabic (Egyptian dialect priority) and English, auto-detected from the citizen's first message
- [ ] **Fallback to human dispatcher** — if the bot can't determine the emergency type after N turns, escalate to a live dispatcher with the partial transcript attached
- > **[Edit]** This replaces the "Arabic Report Extraction" feature — instead of extracting from a single free-text blob, the chatbot guides the citizen through a structured conversation. The extraction still happens, but interactively. The few-shot LLM approach via OpenRouter stays the same under the hood.

#### Where it lives

- [ ] **Web:** New chat widget on `/emergency` page — citizen can type or use the existing form
- [ ] **Mobile:** Chat screen in the Expo app's citizen flow
- [ ] **Backend:** New `/api/ai/chat` endpoint in the Python AI service, called from both web and mobile
- > **[Edit]** Use the existing WebSocket infrastructure for streaming chat responses. The backend already has WS push — add a `chat` channel type.

### Voice Reports (weeks 15-16)

- [ ] Speech-to-text for Arabic
- [ ] Feed output into the **chatbot** — voice becomes another input modality for the same intake flow
- > **[Edit]** Consider Whisper large-v3 — it handles Egyptian Arabic reasonably well. This is a "Nice to have" (see cut line) so keep the integration surface minimal: one endpoint, one format. The chatbot can display the transcribed text and let the citizen correct it before proceeding.

### Photo / Video Severity (weeks 16-18)

- [ ] Pretrained vision model + fine-tuned classification head
- [ ] Feed severity score into the incident priority
- [ ] **[Edit — Chatbot integration]** The citizen can send a photo/video mid-chat. The bot runs severity classification and uses it to adjust the extracted priority.
- > **[Edit]** Media is already stored in Supabase private buckets with signed URLs. The AI service will need temporary access to download media for inference — use short-lived signed URLs (the backend already generates these). Don't copy media to a second location.

### False-Report Detection & Mass-Casualty Clustering (weeks 17-18)

- [ ] Both build on the data already being logged
- [ ] False-report detection: flag anomalous patterns
  - **[Edit — Chatbot integration]** The chatbot's conversation logs are a rich signal — inconsistent answers, copy-pasted text, or impossibly fast responses can feed into the false-report model.
- [ ] Clustering: detect when multiple reports describe the same large-scale event
- > **[Edit]** The backend already has duplicate-incident suggestion. Clustering is a natural extension — but the two must not conflict. Define clearly: duplicates = same incident, clusters = related incidents in a mass-casualty event.

---

## Phase 4: Medical, Forecasting & Citizen (weeks 19-23, Feb-Mar)

### Vitals Early-Warning (weeks 19-20)

- [ ] NEWS2-style score first (rule-based)
- [ ] Then an ML model for trends
- [ ] Show alerts on both responder and hospital views
- > **[Edit]** The mobile Expo app covers Responder — vitals alerts need to work on both web and mobile. Coordinate the push notification format with the mobile team.

### SBAR Handoff Summary (weeks 20-21)

- [ ] LLM summary from timeline, vitals, and notes
- [ ] Human-edit step before it's finalized
- > **[Edit]** Clinical notes are encrypted (AES-256-GCM). The summarization endpoint must decrypt server-side, summarize, and return the summary also encrypted. Document the data flow clearly — this is a thesis-defense question.

### Demand Forecasting Heatmap (weeks 21-22)

- [ ] Spatial-temporal model for incident demand
- [ ] Admin map layer with staging suggestions
- > **[Edit]** The web console already uses MapLibre. Build the heatmap as a MapLibre layer — don't introduce a second map library. The admin role (`/admin`) is the right surface for this.

### 🤖 Emergency Chatbot — First-Aid Mode (weeks 22-23)

> This is the **second mode** of the Emergency Chatbot built in [Phase 3](#-emergency-chatbot--intake-mode-weeks-13-15). Same infrastructure, different knowledge base.

- [ ] **Curate the first-aid knowledge base** — vetted protocols only:
  - CPR / choking / bleeding control / burns / fractures / allergic reactions
  - Sourced from WHO, Red Cross/Crescent, or equivalent — cite every protocol
  - Stored as structured Markdown in `data/kb/first-aid/`
- [ ] **Activate after incident creation** — once the chatbot finishes intake and creates an incident, it switches to first-aid mode: "While you wait for help, here's what you can do"
- [ ] **Step-by-step guidance** — not a wall of text. The bot walks the citizen through one action at a time, confirming before moving to the next step
- [ ] **Strict refusal** — if the citizen asks something outside the KB (e.g. drug dosages, diagnosis), the bot says: "I can only provide basic first-aid guidance. A medical professional is on the way."
- [ ] **Disclaimer on every session** — "This is automated guidance, not medical advice. Follow the instructions of arriving responders."
- [ ] **Real-time translation** — cross-language communication between citizen and responder chat, using the same LLM pipeline
- > **[Edit]** Promoted from "Nice to have" to a core part of the chatbot. The infrastructure (RAG, OpenRouter, guardrails) is already built in Phase 3 — this phase just adds the first-aid KB and the mode-switching logic. Much cheaper to build now that the chatbot exists.

> **🛑 Feature freeze at end of week 23.** Nothing new gets started after this date.

---

## Phase 5: Integration & Delivery (weeks 24-30, Mar-May)

### Testing (weeks 24-25)

- [ ] End-to-end testing across all AI features
- [ ] Latency testing (target: < 2s for recommendations, < 5s for LLM calls)
- [ ] Fallback testing — what happens when the AI service is down?
  - > **[Edit]** The web/backend should degrade gracefully: hide suggestion UI, don't break the dispatch workflow. Test this explicitly.
- [ ] Security review of every AI endpoint
  - > **[Edit]** Particular focus: can a non-admin user hit the AI endpoints directly? The backend enforces permissions in the service layer — verify the AI service does too.

### Evaluation (weeks 25-27)

- [ ] Final experiments and numbers
  - Precision / recall for extraction and classification
  - ETA mean absolute error
  - Ranking accuracy (NDCG or MRR) for unit recommendation
  - Ablation studies
- [ ] Simulated dispatcher study: with vs. without AI suggestions
  - > **[Edit]** Even 5–10 people doing a mock dispatch session with recorded incidents is enough. Script the scenario in advance.

### Thesis & Demo (weeks 27-29)

- [ ] Thesis / report writing
- [ ] Architecture diagrams (the existing `docs/diagrams/` + PlantUML setup is perfect for this)
- [ ] Demo script
- [ ] **Record a backup demo video** — never rely solely on a live demo
  - > **[Edit]** Run the demo on recorded/simulated data you fully control. The Cairo simulator from Phase 1 is your safety net here.

### Defense Prep (weeks 29-30)

- [ ] Two full defense rehearsals
- [ ] Slack time for surprises

---

## Cut Line — If Behind Schedule

> If the team is behind at **week 12**, cut from the bottom of this list first.
> A weak version of the "Nice to have" features is worse than skipping them entirely.

| Tier | Features |
|------|----------|
| 🔴 **Must have** | Unit recommendation, ETA prediction, **emergency chatbot (intake mode)**, vitals early-warning, evaluation harness |
| 🟡 **Should have** | Hospital recommendation, photo/video severity, forecasting heatmap, handoff summary, **emergency chatbot (first-aid mode)** |
| 🟢 **Nice to have** | Voice reports, false-report detection, clustering, translation |

---

## Biggest Risks

| Risk | Mitigation |
|------|-----------|
| **Data** — everything depends on the simulator; if it slips, everything slips | Protect weeks 3–6 at all costs |
| **Arabic dialect quality** — MSA models fail on Egyptian phrasing | Budget extra time; test on real عامية early |
| **Integration with web/mobile teams** — late API disagreements are expensive | Agree on API contracts in Phase 0, not Phase 5 |
| **Demo reliability** — live demos break | Run on recorded/simulated data you control; have a backup video |

---

## Suggested Additions (not in original plan)

> [!IMPORTANT]
> These are editorial suggestions based on reviewing the existing codebase.

1. **Create a `services/ai/` directory** at the repo root for the FastAPI service. Keep it a sibling of `backend/`, `web/`, and `mobile/` — the `docker-compose.yml` at the root can orchestrate all four.

2. **Add the AI service to `docker-compose.yml`** early (Phase 0). Even a stub that returns `{"status": "ok"}` lets the web/backend teams develop against it from day one.

3. **Version the AI models.** Every model artifact should have a version string logged with every prediction. When the thesis says "model v3 improved ETA MAE by 12%," you need to prove it.

4. **Set up a shared experiment tracker** (MLflow, Weights & Biases free tier, or even a shared spreadsheet). Individual Jupyter notebooks are fine for exploration, but you need one place to compare results across the team.

5. **Write integration tests for the AI ↔ backend boundary** — not just unit tests on each side. A broken contract between the two services is the most likely integration failure.

---

*Last updated: October 3, 2025*
*Rescale dates once the real defense date is confirmed.*
