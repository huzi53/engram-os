# 001 — Port HuziOS, or build Engram fresh?

## Verdict

**Not simpler to port. Build Engram fresh per `plan.md`; harvest three concepts from
HuziOS, port zero code.** The two projects share a slogan ("personal memory OS") and
one user, and almost nothing else: different language (Node/Express vs Python/FastAPI),
different data model (a filesystem Obsidian markdown vault vs PostgreSQL+pgvector),
different capture (edit files in Obsidian vs a Telegram bot), different hosting
(localhost/Tailscale vs a Hetzner VPS behind Caddy), and a different AI stance (three
hand-wired chat engines vs one OpenAI-compatible slot pair + claude.ai-over-MCP mentor).
Porting would mean rewriting every HuziOS line into a foreign runtime and storage model
while dragging along scope Engram deliberately dropped — strictly more work than executing
`plan.md`'s M0 scaffold, which already assumes a clean repo. `plan.md` is already the
successor: it derives from a separate Kimi-generated spec, never references HuziOS, and
its Appendix A "spec deviations" are a from-scratch design, not a migration. This is
option (c): Engram *is* the reimagining, and its plan already accounts for it.

## What HuziOS actually is (verified against files, not just the narrative)

A single-dependency (express) Node app, no build step, serving a vanilla-JS glass
dashboard at `localhost:4816`. Data lives as markdown in `vault/` (Obsidian). ~1035
lines of backend across 8 components (`app/components/*.js`), each `(opts) => router`
mounted at `/api/<name>`. Features: notes browser, ACCA study/mastery tracker with
AI-drill generation and spaced repetition (`study.js`, 327 lines, hardcoded ACCA
syllabus + `Journey.md` exemption parsing), a 3-engine chat (`chat.js`, 235 lines:
Claude Code CLI agentic over the vault, Gemini API, NVIDIA NIM, with a live 429
Gemini→NIM fallback), news/weather cards (MET Malaysia), and vault-zip backups.
Purpose: a solo ACCA student's study/dashboard cockpit. It works; it is 3 days old.

## What Engram-OS envisions

A universal capture-first memory system (`plan.md`, M0–M7). Capture anything from a phone
in <10s via a Telegram bot → FastAPI ingestion → Postgres+pgvector → auto-classify (budget-
capped fast model) → embed (multilingual-e5-small, CPU) → hybrid semantic+keyword search →
entity graph → morning briefing → mentor via claude.ai MCP connector. One VPS, one DB, one
bot, one Next.js dashboard on Cloudflare Pages. Overlap with HuziOS is thematic only
("my data, AI on top"); the architecture diverges at every layer.

## Reusability ledger (concrete)

**Directly reusable (copy-paste / near-copy): none.** The JS→Python + Express→FastAPI +
filesystem→Postgres gaps mean every file is a rewrite, not a move.

**Concepts worth harvesting (reference HuziOS, re-implement in Python when its milestone
arrives):**
- Provider 429 fallback. `chat.js` lines 148–154 (Gemini quota → reroute to NIM, transparently)
  is a working proof of exactly the `LLM_FAST_FALLBACK`/`LLM_SMART_FALLBACK` pattern in
  `plan.md` portability rule 1. Read it as a reference when building Engram's AI Router (M3);
  do not port it.
- Spaced-repetition + mastery + AI-drill design. `study.js` is the conceptual seed for
  Engram's V3 study/quiz system (`plan.md` V3, spec §9). Lift the *model* (per-topic mastery
  JSON, exam-style drill prompts) when V3 lands; rewrite in Python against Postgres.
- Glass dashboard aesthetic. `app/public/app.css` + shell is a visual-language reference for
  Engram's Next.js dashboard — inspiration only, not code (different framework).

**Drop entirely (Engram already replaced or descoped these):**
- 3-engine chat / Claude-CLI-agentic-over-vault → replaced by claude.ai MCP mentor (M6, no chat UI).
- Vault markdown as the datastore → replaced by Postgres+pgvector.
- Access-key middleware → replaced by single-user JWT (M0).
- Vault-zip backup → replaced by restic → Backblaze B2 (M0).
- News/weather cards, MET Malaysia, ACCA syllabus hardcoding → out of Engram's scope.

## Recommended path (phased)

1. **Adopt `plan.md` as the source of truth.** No HuziOS migration milestone exists or is
   needed. Verify: this file is the only reconciliation doc; `plan.md` M0–M7 stands unchanged.
2. **Start M0 as written** ("start M0" in the repo): VPS, HTTPS, JWT, Postgres+pgvector,
   nightly backup. Verify: log in from phone over the internet; a restore works (plan.md M0 exit test).
3. **At M3, open `HuziOS/app/components/chat.js` as a reference** while building the AI Router
   fallback — confirm the transparent-reroute UX (a visible "rerouted" chip) carries over. Verify:
   force a fast-model rate-limit, see the router retry the fallback without stalling.
4. **When V3 study system is scheduled, open `HuziOS/app/components/study.js`** for the mastery
   + drill model. Verify: N/A until V3.

## Cut

- No HuziOS→Engram code migration milestone — every line is a cross-language/cross-storage rewrite; costlier than fresh.
- No chat-UI port — Engram's mentor is claude.ai over MCP; the 3-engine chat is dead scope.
- No vault/filesystem-storage carryover — Postgres+pgvector replaces it wholesale.
- No changes to `plan.md` — it already supersedes HuziOS; this doc only records why.
- No new code in either project — planning only, per task.

## Frontend?

No. This is a decision/routing document; it changes no UI. (Engram's own dashboard build
arrives later inside `plan.md` M0+ and will need the frontend agent then — not here.)
