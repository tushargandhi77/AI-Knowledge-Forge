# KnowledgeForge — Team Execution Plan

**Team size:** 4
**Sprint goal:** Ship a working, demo-ready **Frontend + Advanced RAG assistant + Gamified points system** on top of a Structured Engineering Logbook — everything else in the original proposal (PII pipeline, MCP, voice input, full agentic CLI) is roadmap, not sprint scope.

> Assumption: this is a standard hackathon window (24–48 hrs of build time). If your actual window is longer/shorter, tell me and I'll rescale the day-by-day plan.

---

## 1. What "done" looks like for this sprint

A judge/user should be able to:
1. Open the web app, log in.
2. Add a **log entry** (bug fix / knowledge note) — text input, minimum viable version of the "structured logbook."
3. Ask a question in a **chat loop** — the RAG agent answers, grounded in logged entries, and cites which entry it pulled from.
4. See **points** go up when they contribute, and see a **leaderboard**.

That's the whole demo. Everything else is a stretch goal.

---

## 2. Scope decision: what's IN vs OUT for this sprint

| In scope (build now) | Out of scope (roadmap — see Section 8) |
|---|---|
| Next.js frontend (logbook form + chat UI + leaderboard) | Voice input, image upload |
| FastAPI backend | PII scrubbing pipeline |
| RAG pipeline (embeddings + retrieval + LLM answer w/ citations) | MCP server integrations (Jira/Confluence/GitHub) |
| LangGraph for the *Q&A agent loop* (multi-step: retrieve → reason → answer → cite) | Web-search-augmented agent, multi-agent cross-analysis |
| MongoDB Atlas (data + vector search) | Full agentic CLI / dev-agent product |
| Basic points system + leaderboard | Badges, audit trails, RBAC beyond basic auth |
| JWT auth (single team namespace is fine for demo) | Multi-tenant namespacing |

This mirrors the proposal's **Pillar 1 (logbook) + Pillar 2 (RAG/agentic Q&A) + Pillar 4 (gamification)**, deliberately deferring **Pillar 3 (PII scrubbing)** and the MCP/CLI vision to post-hackathon.

---

## 3. Tech stack (locked — matches your proposal doc)

| Layer | Tech | Notes |
|---|---|---|
| Frontend | Next.js + Tailwind CSS | Logbook form, chat interface, leaderboard |
| Backend API | FastAPI (Python) | Async REST, orchestrates AI calls |
| AI orchestration | LangChain + **LangGraph** | RAG pipeline + agentic Q&A loop |
| Vector + data store | MongoDB Atlas Vector Search | One DB for structured logs + embeddings — don't stand up a separate vector DB, it'll cost you time you don't have |
| Auth | JWT (simple, no RBAC yet) | Enough for demo login |
| Hosting | Vercel (frontend) + Render/Railway (FastAPI) or a single VM if you want less DevOps overhead | Pick whichever your team already knows — don't learn a new host mid-hackathon |

---

## 4. MVP architecture

```
[Next.js Frontend]
  ├─ Logbook Form  ──POST──▶ [FastAPI /entries] ──▶ embed + store ──▶ [MongoDB Atlas: docs + vectors]
  ├─ Chat UI       ──POST──▶ [FastAPI /ask]     ──▶ [LangGraph agent loop] ──▶ [MongoDB vector search] ──▶ LLM ──▶ cited answer
  └─ Leaderboard   ──GET───▶ [FastAPI /points]  ──▶ [MongoDB: points collection]
```

**LangGraph agent loop (Pillar 2, MVP version):**
`receive question → retrieve top-k chunks from Mongo vector search → LLM reasons over chunks → draft answer → attach citations (entry IDs) → return`
This is a single-cycle graph for the sprint. Add branching (re-retrieve if confidence low, multi-hop) once the base loop works — don't build the fancy version first.

---

## 5. Team split (4 people)

| Role | Owns | Primary work |
|---|---|---|
| **Frontend Lead** | Next.js app | Logbook form, chat UI (loop-style, message history), leaderboard page, auth screens |
| **Backend Lead** | FastAPI service | `/entries`, `/ask`, `/points` endpoints, JWT auth, MongoDB connection/schema |
| **AI/RAG Lead** | LangChain + LangGraph pipeline | Embedding pipeline on entry creation, vector search query, LangGraph agent graph, citation formatting |
| **Data/Infra + Points Lead** | MongoDB Atlas setup, points logic, deployment | Atlas cluster + vector index config, points-award rules, leaderboard aggregation query, CI/deploy (Vercel + backend host), demo data seeding |

Two people (Frontend + Backend) can pair-integrate early so the contract (API shapes) is agreed by hour 2–3. AI/RAG lead can build against mocked entries while real ingestion is wired up in parallel.

---

## 6. Day-by-day plan (48-hour window example)

**Hours 0–2 — Alignment**
- Agree on API contract: `POST /entries`, `POST /ask`, `GET /points`, `GET /leaderboard`, `POST /auth/login`.
- Agree on Mongo schema: `entries { _id, title, body, embedding, author_id, created_at }`, `points { user_id, total, history[] }`.
- Repo scaffolded, everyone pushes a "hello world" to their layer.

**Hours 2–10 — Core build**
- Frontend: logbook form + submit, chat UI skeleton (input + message list), auth pages.
- Backend: CRUD for entries, JWT login/signup, points-award-on-entry-create logic.
- AI/RAG: embedding function on entry save, Mongo vector index created, basic retrieval query working standalone (test via script, not UI yet).
- Data/Infra: Atlas cluster live, vector search index configured, seed ~15–20 realistic dummy log entries so RAG has something to retrieve from demo day.

**Hours 10–20 — Integration**
- Wire chat UI → `/ask` → LangGraph loop → Mongo retrieval → LLM → response with citations rendered in UI.
- Wire leaderboard UI → `/points`.
- End-to-end test: log an entry → ask a question about it → get a cited answer.

**Hours 20–30 — Points/gamification polish + agent quality**
- Points rules: e.g. +10 for new entry, +2 for entry later cited by the AI in an answer (this directly rewards *useful* documentation, which is the actual pain point in your proposal).
- Leaderboard UI polish (ranks, avatars/initials, simple badges if time allows).
- Improve LangGraph prompt/graph: better grounding, refuse-if-no-context behavior, cleaner citations.

**Hours 30–40 — Hardening**
- Error states, loading states, empty states (no entries yet, etc.).
- Auth edge cases.
- Seed a realistic demo dataset + rehearse the demo script.

**Hours 40–48 — Buffer + demo prep**
- Bug bash. Freeze features. Record backup demo video in case live demo/network fails.
- Prepare the "future vision" slide (Section 8 below) — judges reward roadmap thinking even on unbuilt features.

---

## 7. Points/gamification system — concrete v1 rules

Keep it simple enough to implement in an hour, not a full economy:

| Action | Points |
|---|---|
| Create a new log entry | +10 |
| Your entry gets retrieved/cited in someone else's AI answer | +2 (caps at +20/entry to avoid gaming) |
| First entry of the day (streak incentive) | +5 bonus |

`GET /leaderboard` = simple aggregation sorted by `points.total`, descending. Badges (Bug Slayer, Knowledge Champion, etc. from the proposal) are a nice-to-have visual layer on top of point thresholds — only do this if core loop is done early.

---

## 8. Deferred to post-hackathon roadmap (don't build now, but demo-mention)

These are straight from your proposal and are legitimately good — just not sprint-scope:

1. **PII scrubbing pipeline (Pillar 3)** — pre-ingestion NLP detection + redaction + review diff + audit trail.
2. **MCP integrations** — connecting the LangGraph agent to Jira/Confluence/GitHub MCP servers for live external context.
3. **Voice input & image upload** for the logbook.
4. **Web-search-augmented agent** for external research (CVEs, docs, community solutions).
5. **Multi-agent / multi-hop LangGraph workflows** — cross-referencing multiple past issues, pattern surfacing across the whole knowledge base.

---

## 9. The bigger picture: from web app → agentic CLI product

You mentioned the real long-term goal is an **agentic, CLI-first, closed-source developer tool** — something in the spirit of a coding agent that plugs into MCP servers and operates on a company's proprietary codebase/data. That's a materially different product surface from the web app, so treat it as **Phase 2**, built on top of what you learn from the RAG/LangGraph work in Phase 1:

**Phase 2 direction (not sprint work, but worth having a one-slide vision for):**
- The LangGraph agent graph you build for the web Q&A loop is reusable — it's the same "retrieve → reason → act → cite" pattern a CLI coding agent needs, just with more tools (file read/write, shell exec, git) added as LangGraph nodes.
- MCP is your extensibility layer either way — same protocol serves both the web assistant (Jira/Confluence/GitHub context) and a future CLI agent (repo tools, internal APIs).
- Closed-source/proprietary packaging is a distribution decision (private PyPI/npm registry, license-gated binary, or a hosted API you gate with keys) — doesn't affect the Phase 1 architecture, so no need to design for it yet.
- Suggested order once the hackathon is done: (1) harden the RAG+LangGraph core, (2) add a thin CLI wrapper that calls the same backend, (3) expand the agent's tool set (file/shell/git access) behind MCP, (4) only then worry about packaging/licensing for closed-source distribution.

---

## 10. Risks & mitigations

| Risk | Mitigation |
|---|---|
| MongoDB Atlas Vector Search setup eats hours (index config is fiddly) | Data/Infra lead sets this up **first**, hour 0–2, before anyone depends on it |
| LangGraph learning curve if team is new to it | Start with a single linear graph (no branching) — it's just retrieve→reason→answer to start |
| Frontend/backend integration slips to the last hours | Lock the API contract in hour 0–2 and build against mocked responses in parallel |
| Demo fails live (network/API down) | Record a backup video by hour 40 |
| Scope creep back toward PII/MCP/voice | Re-read Section 2 — if it's in the "out of scope" column, it doesn't get built this sprint |
