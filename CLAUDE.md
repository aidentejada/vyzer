# CLAUDE.md

> Working codename: **`<APP_NAME>`** (rename before first commit).
> This file is the source of truth for the project. Read it fully before doing anything.

---

## 1. What we're building

An **open-source, local-first desktop application** that helps people find jobs/internships/growth opportunities and prepare for interviews. Think self-hosted and downloadable (PewDiePie's "Odysseus" ethos), **not** a web app. The user downloads it, runs it on their own machine, and plugs in their own AI API key.

The central bet: **a single static résumé is structurally inefficient.** Different roles weight different skills and projects, so the app generates a *per-job* résumé that reframes and re-selects from a master profile — without fabricating anything.

### Non-negotiable principles
- **Local-first.** All user data lives on the user's machine. No central server holds profiles, résumés, or keys.
- **Bring-your-own-key (BYO-key).** The user supplies their own LLM API key; it is stored locally and never transmitted anywhere except directly to the chosen provider.
- **Provider-agnostic AI.** Abstract the LLM behind an interface from day one. Support Claude, OpenAI, and local/open models. Never hard-code one provider.
- **Open source.** Prefer free/open API resources and link to them. Keep dependencies permissively licensed.
- **No fabrication, ever.** The résumé generator reframes and selects real content. It must never invent experience, skills, employers, dates, or achievements. This is a hard product and ethical constraint.
- **Privacy by construction.** Because there's no server, there's no central data to leak. Honor that — don't add telemetry that exfiltrates user data.

---

## 2. Recommended tech stack

- **Shell:** Tauri (Rust core + web frontend). Lightweight, secure, cross-platform, no bundled browser bloat. Plays to a local-first, privacy-first design.
- **Frontend:** React + TypeScript + Vite.
- **Local storage:** SQLite (via Tauri's SQL plugin) for structured profile data; local filesystem for generated PDFs and cached company-intel documents.
- **AI layer:** a provider-agnostic `LLMProvider` interface with concrete adapters (`ClaudeProvider`, `OpenAIProvider`, `LocalProvider`).
- **PDF generation:** an HTML/CSS-to-PDF pipeline so résumé templates are styleable with CSS (e.g. a headless renderer or a Rust PDF crate — evaluate in Phase 1).
- **RAG/embeddings:** a local vector store (e.g. SQLite-backed vector extension or an embedded store) so retrieval also stays on-device.

> If Tauri proves heavy for any contributor, Electron is the fallback — but default to Tauri.

---

## 3. Architecture overview

```
┌─────────────────────────────────────────────┐
│                  Frontend (React)             │
│  Profile · Résumé Studio · Job Board ·        │
│  Learning · Network · App Assistant ·         │
│  Interview Prep · Settings(keys/providers)    │
└───────────────┬───────────────────────────────┘
                │ Tauri commands (IPC)
┌───────────────┴───────────────────────────────┐
│                 Core (Rust)                    │
│  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Local Store  │  │  AI Service (provider- │  │
│  │ SQLite + FS  │  │  agnostic LLMProvider) │  │
│  └──────────────┘  └───────────────────────┘  │
│  ┌──────────────┐  ┌───────────────────────┐  │
│  │ RAG / Vector │  │  Connector Layer       │  │
│  │ store        │  │  (GitHub, imports, etc)│  │
│  └──────────────┘  └───────────────────────┘  │
│  ┌───────────────────────────────────────────┐ │
│  │ PDF Engine (HTML/CSS templates → PDF)     │ │
│  └───────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
```

**The Master Profile is the spine.** Everything reads from it.

---

## 4. Data model (Master Profile)

Normalized, relevance-taggable entities so the résumé generator can filter and rank per role:

- `Experience` — org, title, dates, bullets, skills-used (tags).
- `Project` — name, description, role, links, tech/skill tags, source (manual | github).
- `Skill` — name, category, proficiency, evidence refs.
- `Achievement` — description, metric, date, linked experience/project.
- `EducationItem`, `Certification` — standard fields.
- `ResumeProfile` — a generated, per-job artifact: selected entity IDs, reframed bullet text, target role/company ref, chosen layout/template.
- `JobTarget` — pasted/linked JD, parsed requirements, company-intel doc refs, fit score.

GitHub and LinkedIn-archive imports write into the same schema (no parallel models).

---

## 5. Feature modules

### M1 — Profile & Portfolio Ingestion
Build the master profile: experiences, skills, projects, achievements. Accelerated imports: (a) **GitHub** — link account, auto-pull repos as projects (name, description, language, stars, README highlights); (b) **LinkedIn** — ingest the user's own exported LinkedIn data archive (ToS-safe; see §7).

### M2 — Dynamic Résumé Generation *(core)*
User pastes/links a job description. System parses requirements, then generates a tailored résumé by **selecting** the most relevant projects/skills and **reframing** existing experience to the role's language. Reframing only — never fabrication.

### M3 — Company & Role Intelligence (RAG)
Retrieval layer that parses the JD and pulls company context (mission, products, recent news, tech stack, values) so tailoring and suggestions are grounded, not generic. Cached locally per company.

### M4 — Fit Analysis
Tells the user whether they're **overqualified, well-matched, or underqualified** for a role, based on the gap between master profile and parsed requirements.

### M5 — Gap-Closing / Upskilling Engine
When Fit Analysis flags *underqualified*, surface three closing paths: free learning resources, courses/certifications the user could earn, and concrete **AI-buildable projects** to demonstrate the missing skill. Each buildable project ships with a **"Copy Prompt"** button that produces a ready-to-paste prompt for Claude or any IDE.

### M6 — Job Discovery & Early-Application Alerts
In-app job board aggregating sources. Critically, catch roles on **company career pages before** they hit big aggregators so users apply early. Speed-to-apply is a first-class feature. (See §7 for realistic v1 sourcing.)

### M7 — Career Growth & Compensation Intelligence
Learning page for growth, backed by compensation data: show how a specific course/cert/skill translates into higher pay elsewhere, making upskilling ROI-driven.

### M8 — Network Leverage
Import the user's LinkedIn network (from their own export) and surface who's most worth reaching out to — for referrals, collaboration, or guidance — mapping the highest-value contacts already in their network.

### M9 — Application Assistant
For application/screening questions, the system **does not generate answers** (authenticity is the point). It draws on the JD + company RAG to **suggest what the user should highlight about themselves** to match the role. The user writes their own words; the tool aims them.

### M10 — Interview Preparation Manager
When the user lands an interview/screening, pull targeted prep resources — role-specific, company-specific, and format-specific.

### M11 — Résumé Layout & PDF Customization
Full control over the exported résumé PDF:
- **Reorder sections** (drag-and-drop the order of Experience / Projects / Skills / Education / etc.).
- **Toggle sections** on/off per résumé.
- **Multiple templates / themes** (CSS-driven), selectable per generated résumé.
- **Typography & spacing controls** (font family, size, margins, density).
- Layout choices are stored on the `ResumeProfile`, so each per-job résumé can have its own ordering/template.
- Export to PDF via the HTML/CSS template engine; live preview before export.

---

## 6. Build phases (do them in order)

**Phase 1 — Core loop (self-contained, highest value).**
Master Profile schema (M1) → GitHub import → Dynamic Résumé Generation (M2) → Fit Analysis (M4) → Résumé Layout & PDF export (M11). This is the demoable magic and the friendliest integration. Ship this end-to-end before anything else.

**Phase 2 — Intelligence layer (LLM orchestration on top of Phase 1).**
Company RAG (M3) → Gap-Closing engine (M5) → Application Assistant (M9) → Interview Prep (M10).

**Phase 3 — External/fragile integrations (review connector code carefully).**
Job aggregation + early alerts (M6) → LinkedIn-archive import & Network Leverage (M8) → Compensation intelligence (M7).

---

## 7. Constraints, guardrails & integration realities

- **No fabrication.** Repeat: the generator reframes/selects real content only. Add tests asserting generated bullets trace back to real profile entities.
- **LinkedIn.** Scraping violates LinkedIn's ToS and their API is heavily gated. **Do not scrape.** Use the user's own downloadable LinkedIn data export for profile and network import.
- **Handshake.** No public API. Treat as out-of-scope for automated pull in v1 (manual paste at most). Don't build fragile unofficial access.
- **Early job sourcing (M6).** Realistic v1: aggregate from ATS-hosted boards that expose feeds/APIs (Greenhouse, Lever, Ashby). Per-company crawlers are ongoing maintenance — keep each connector behind a clean interface so one breaking source can't take down the app.
- **Compensation data (M7).** Pick a defensible source (public/community datasets, BLS for baselines). Document where numbers come from.
- **Keys & secrets.** Store the user's API key in the OS keychain/secure store, never in plaintext config committed to the repo. Never log keys.
- **No telemetry that exfiltrates user data.** If analytics are added later, they must be opt-in and local/aggregate only.

---

## 8. Conventions

- TypeScript strict mode on the frontend; clippy-clean Rust on the core.
- Every connector implements a shared interface; failures are isolated and surfaced gracefully.
- Write tests alongside JD-parsing, résumé-tailoring, and fit-scoring logic — these silently regress.
- Small, reviewable commits with descriptive messages. The human reviews all connector and key-handling code.
- Keep the AI provider abstraction clean: adding a new model/provider should not touch feature code.

---

## 9. Phase 1 — definition of done

A user can: create a profile, import GitHub repos as projects, paste a job description, get a tailored (non-fabricated) résumé with a fit assessment, reorder/toggle sections and pick a template, preview it, and export a clean PDF — all fully offline except the direct call to their chosen LLM provider.
