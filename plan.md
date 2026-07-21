# DOPE OS — Executable Build Plan v1.5

> Derived from `Personal_OS_Technical_Specification_v1.0.pdf` (Kimi-generated, 2026-07-20),
> audited and corrected. This plan supersedes the spec where they conflict — see
> [Appendix A: Spec deviations & fixes](#appendix-a-spec-deviations--fixes).

## Locked decisions (from planning session, 2026-07-20)

| Decision | Choice |
|---|---|
| Audience | Single user (Huzi). No multi-tenancy, no third-party auth service. Keep `user_id` column for future-proofing only. |
| Access | **Laptop-first, €0/mo (v1.5).** Engram runs on the laptop (Docker Compose via WSL2). Dashboard reachable from the phone over **Tailscale** (exactly how HuziOS is reached today). Telegram bot uses **long-polling** (`getUpdates`) — works behind NAT, no public endpoint, no domain. The two surfaces that genuinely need public HTTPS later (email-worker target M5, claude.ai MCP connector M6) go through **Tailscale Funnel** (free). **VPS = graduation path, not day one:** the stack is Docker Compose + Postgres dump + restic, so moving to a Hetzner CX22 (~€4.35/mo) is an afternoon, triggered by real annoyance (laptop-uptime misses, expired Telegram queue, briefing gaps) — not speculation. Cloudflare Pages deferred together with the VPS. Oracle free ARM: still optional scout only, never the sole home of data. |
| HuziOS | **Stays untouched, local, and daily-driven (v1.5)** — it is ACCA-exam-critical and works. No rework during Engram M0–M3. At **M3.5** the glass UI is ported onto Engram's API (strangler fig: panels retire one by one, never a freeze-and-rewrite). Chat stays on Claude Code CLI over the vault until the MCP mentor (M6) proves out — the port is an aesthetic+shell reuse; every panel's data layer is a rewrite and is costed as such. |
| Vault | **Index, don't migrate (v1.5).** Obsidian remains the source of truth for authored/study content — the SR flashcard plugin, kanban, and editing UX are things Engram will not rebuild. Engram ingests the vault **read-only** into the same embedding/search space at M3.5 (trivial while laptop-hosted: the vault is a local folder; becomes a sync job only on VPS graduation). Full migration reconsidered at V3 only if Engram ever grows native editing + spaced repetition. |
| Phone | Android. **Capture via Telegram bot (v1.4, primary)** — share sheet → Telegram → bot handles text, links, photos, files, voice, with offline queueing for free. Dashboard quick-note box = the always-available **private path** (direct HTTPS to VPS, no Telegram transit) for sensitive captures. PWA Web Share Target deferred to V3. **Decision checkpoint after MVP (M3):** leaning "Telegram for everything"; revisit with two weeks of real usage data — the ingestion API is channel-agnostic, so this is a habit choice, not an architecture lock. |
| AI tiers | **No local LLM** (no GPU). Tier 1 = Python heuristics (free). Tier 2 = cloud **fast model** (classification). Tier 3 = cloud **smart model** (enrichment, briefing). **Interactive reasoning (mentor) = existing Claude Pro sub via claude.ai MCP connector — no Claude API.** Hard daily budget cap on the API tiers. |
| AI provider | **Replaceable by design** (v1.1). All pipeline LLM calls go through the OpenAI-compatible API standard — provider is env config, not code. **Picked by data policy first, price second (v1.2): start OpenAI (no-training default on API traffic)**; swap candidates: Kimi/Moonshot paid, Claude, others passing the data-policy gate. See [AI provider portability](#ai-provider-portability). |
| Budget | **€0/mo hosting (laptop)** + **one-time ~$10 API topup lasting ~a year** + Claude Pro (already paying). Domain ~$10/yr deferred to M5 — email ingestion is the only feature that requires one. VPS ~€5/mo only on graduation. Total new spend ≈ **~$1/mo AI**. |
| Build mode | Claude Code writes the code; Huzi directs, reviews, and tests. Pace is set by review bandwidth, not coding time. |
| Priorities | 1) Capture + search. 2) Auto-organization + linking. Briefing/rediscovery next; mentor/insights later. |

## Goal

A personal memory system: capture anything from the phone in under 10 seconds, have it
automatically organized, linked, and searchable by meaning — accessible from anywhere,
running for under $10/month, with zero manual filing.

## Architecture (right-sized)

One box, one database, one app:

```
Android phone ──share──> Telegram cloud <──long-poll── Bot   CAPTURE (primary)
Laptop/desktop ────────> Telegram cloud <──────┘ (getUpdates — no public IP)
Sensitive stuff ──quick-note──> Dashboard ─────┐             CAPTURE (private path)
                                               │
Dashboard (Engram UI → HuziOS glass UI at M3.5)│             CONSUMPTION
search · browse · related · projects · briefing card
        │ HTTPS over Tailscale (phone ⇄ laptop, as HuziOS today)
        v
LAPTOP (Docker Compose via WSL2)          ← VPS graduation path documented,
├─ Caddy          (reverse proxy, one entrypoint)          not day one
├─ FastAPI        (API + auth: single-user JWT)
├─ Worker         (pipeline: extract → classify → embed → link)
├─ Bot poller     (Telegram getUpdates loop)
├─ PostgreSQL 16  (+ pgvector) — captures, entities, relations,
│                   full-text search, vector search, job queue
├─ Obsidian vault (local folder, read-only indexed at M3.5 —
│                   Obsidian stays source of truth for study content)
└─ /data volume   (raw files) + nightly restic backup → Backblaze B2
│
v
LLM API — any OpenAI-compatible provider, set via env
(LLM_FAST = classify · LLM_SMART = enrich/briefing)
CPU embeddings local (multilingual-e5-small, 384-dim —
handles Malay+English, provider-independent)
MCP server ──Tailscale Funnel (public HTTPS)──> claude.ai custom
connector (mentor mode runs on the existing Claude Pro sub, $0 API)
```

**What replaced the spec's 6 datastores:** Postgres does vectors (pgvector), full-text
search (tsvector), graph (`entities` + `capture_relations` tables + recursive CTEs),
metadata (JSONB), and the job queue (`FOR UPDATE SKIP LOCKED`). Files live on the VPS
disk. Neo4j, Qdrant, Meilisearch, MinIO, Redis, Celery: all deferred until data volume
proves the need (likely never for one user).

## AI provider portability

The spec's first principle — *"Data is the primary asset. AI is a replaceable processing
layer"* — is enforced by four hard rules (added v1.1):

1. **One client, OpenAI-compatible.** All LLM calls use the `openai` Python SDK with a
   configurable `base_url`. OpenAI is native; Kimi (Moonshot), Anthropic, DeepSeek, and
   OpenRouter all expose OpenAI-compatible endpoints. Swapping provider = editing `.env`:
   ```env
   LLM_BASE_URL=https://api.openai.com/v1     # or https://api.moonshot.ai/v1 (Kimi), etc.
   LLM_API_KEY=sk-...
   LLM_FAST=<current cheap model>              # tier 2: classification, tagging
   LLM_SMART=<current strong model>            # tier 3: enrichment, briefing
   ```
   No provider name appears anywhere in code — only `FAST` and `SMART` slots.
   **Fallback chains (v1.4, adopted from ZeroClaw's provider design):** each slot takes an
   optional `LLM_FAST_FALLBACK`/`LLM_SMART_FALLBACK` (provider+model) — on outage or
   rate-limit the router retries the fallback instead of stalling the pipeline; fallback
   providers must also pass the data-policy gate (rule 5).
2. **Pricing lives in config, not code.** The AI Router's budget cap reads a per-model
   `{input_price, output_price}` table from config, so cost tracking survives any swap.
3. **Lowest-common-denominator structured output.** JSON requested in the prompt +
   Pydantic validation + one retry on parse failure. No provider-specific JSON modes or
   tool-calling formats, so every OpenAI-compatible provider behaves identically.
4. **Eval set gates every swap.** ~30 real captures with known-correct classifications,
   stored in the repo (built during M3). After any provider/model change, re-run the eval;
   accept the swap only if accuracy stays ≥ the current baseline. This catches silent
   quality regressions that a "it seems to work" check misses.
5. **Data-policy gate on providers (v1.2).** Capture content is life data. Any provider
   used in the pipeline must have an explicit no-training policy on API traffic
   (e.g. OpenAI API default; Kimi paid tier). **Free tiers that reserve training rights
   are banned for capture content regardless of price** — it would violate the spec's
   own §15 data-sovereignty principle. Price is the tiebreaker only *after* this gate.

Embeddings are already portable: they run locally (multilingual-e5-small, CPU), so
chat-model swaps never require re-embedding.

## Milestones

Each milestone is a checkpoint with a testable exit criterion. Build order = priority order.

### M0 — Infrastructure live (re-based on the laptop in v1.5)
Docker Compose on the laptop via WSL2 (Caddy, FastAPI hello-world, Postgres+pgvector)
+ HTTPS over Tailscale (Tailscale cert or Caddy internal CA) + single-user JWT login
+ nightly encrypted restic backup → Backblaze B2 + Windows wake-timer/Task Scheduler
setup so scheduled jobs actually fire. No domain, no VPS, no Cloudflare account needed
yet. Document the **VPS graduation runbook** (compose up on Hetzner + `pg_dump` restore +
restic re-point) in the repo from day one so the move stays an afternoon.
**Exit test:** you log in from your phone over Tailscale from outside the house
(mobile data, not home Wi-Fi); a restore from backup works on a clean machine/container.

### M1 — Capture works (MVP core; re-scoped in v1.4, ~half the original size)
`POST /api/v1/capture` (text, URL, photo, file, audio) · **Telegram bot** (long-polling
`getUpdates` loop — v1.5, no webhook/public endpoint) accepting text, links, photos,
files, voice notes, and forwarded messages —
offline queueing, retry, and compression inherited from Telegram itself · dashboard
**quick-note box** as the direct private path · blake2b exact-hash dedup · captures list
view (newest first). ~~PWA share target, IndexedDB retry queue, chunked upload~~ → V3.
**Exit test:** share a TikTok link, a photo, and a voice note to the bot in <10s each;
type one sensitive note via dashboard quick-note; all four appear in the captures list;
sharing the same link twice creates one capture.
*Honest expectation (v1.2):* TikTok/IG shares arrive as a URL; login walls mean extraction
gets link + thumbnail + caption (oEmbed/yt-dlp where it works), **not** full video content.
The capture is still searchable by its metadata — judge M1 against that, not the spec's fantasy.

### M2 — Search works
Tier-1 extraction (URL scrape/metadata, dates via dateparser, amounts, emails, phones;
Tesseract OCR for images) · CPU embeddings (**multilingual-e5-small**, 384-dim — chosen
over English-centric MiniLM because captures are mixed Malay + English) on every capture ·
hybrid search endpoint (pgvector cosine + Postgres FTS, merged) · search UI.
**Exit test:** capture 50+ real items, then find a specific one by meaning ("that article
about sleep") and by keyword, in the top 3 results.

### M3 — Auto-organization (priority feature)
Fast-model classification per capture (categories, tags, title, priority, content_type,
deadline, action_required — the spec's Stage-2 prompt, pointed at the `LLM_FAST` slot) ·
AI Router with daily budget cap + per-model pricing config + spend tracking table ·
classification eval set (~30 labeled captures, the swap gate from the portability rules) ·
category/tag browse UI · thumbs up/down feedback stored on captures.
**Exit test:** a week of real captures lands >80% correctly categorized with sensible
titles, at <$0.15/day API spend.

### M3.5 — Convergence: HuziOS glass UI + vault indexing (new in v1.5)
Port the HuziOS glass UI onto Engram's API — honestly costed as a **rewrite of each
panel's data layer** (the CSS/layout/shell is what actually transfers): search, captures
list, category/tag browse, quick-note, briefing card placeholder · **vault read-only
indexer**: point the embedding pipeline at the local Obsidian vault folder (skip
`.obsidian/`, `30 Chats/`, `Materials/` PDFs initially), re-index on file change
(watchdog), so one search spans captures *and* study notes · HuziOS panels retire
one-by-one as their Engram equivalent lands (strangler fig); ACCA study + chat panels
stay on local HuziOS untouched.
**Exit test:** the glass dashboard, opened on the phone via Tailscale, finds both a
Telegram capture and an Obsidian study note in one search; the old HuziOS notes panel
is retired without anything lost.

### M4 — Linking & graph
`entities` + `capture_entities` + `capture_relations` tables (spec schema, corrected SQL) ·
entity extraction (from fast-model output + heuristics) · similarity links (cosine > 0.82) ·
temporal-proximity + shared-entity + project links · projects CRUD · "Related" panel
on every capture.
**Exit test:** open any capture and see genuinely related items; open a project and see
everything that belongs to it without ever having filed anything.

### M5 — Daily loop: briefing + rediscovery
Morning briefing (cron 06:55 **pinned to Asia/Kuala_Lumpur** — the VPS runs on German/UTC
time; an unpinned cron would deliver your "morning" briefing at ~1pm MY time: pending
tasks, due dates, yesterday's captures, 1–3
rediscovered items → one smart-model call → dashboard card + **pushed as a Telegram bot
message** — ntfy dropped, the bot is already a push channel, v1.4) ·
rediscovery scoring (spec §8, simplified: project-context + semantic + anniversary) ·
email ingestion via **Cloudflare Email Routing → Worker → POST to the Tailscale Funnel
URL** (`capture@yourdomain` — this is the milestone that finally requires buying the
~$10/yr domain, and hardening the Cloudflare account with 2FA; no mail credentials
stored anywhere; IMAP polling documented as fallback only — or defer email entirely if
the domain purchase doesn't feel worth it yet).
**Exit test:** for one week, the morning card is worth reading, and a forwarded email
becomes a searchable capture.

### M6 — Deep enrichment + mentor v1 (redesigned in v1.2)
Selective smart-model enrichment (spec §4.4 trigger rules: high priority / long content /
finance / low confidence) · **remote MCP server** exposed via **Tailscale Funnel**
(public HTTPS, free — claude.ai must be able to reach it) with `search_captures`,
`get_related`, `get_project`, `get_patterns`, `log_study_session` · added as a **custom
connector in claude.ai** — the Claude app (Pro sub, already paid) becomes the mentor
interface with full memory access: mobile, voice, projects included, zero API cost,
and no custom chat UI to build · insights v1 (capture-habit patterns only).
A minimal `search_captures`-only MCP can optionally ship as early as **M2.5** — it is a
thin layer over the search endpoint that already exists by then.
**Exit test:** in the Claude mobile app, ask about something you captured a month ago
and get an answer citing the right captures via the connector. Only then does HuziOS's
local Claude Code chat panel become retirable.

### M7 — Hardening
Voice-note **transcription** (faster-whisper tiny, CPU — the audio itself already arrives
via the bot since M1) · security pass (rate limit, input validation, file-type checks,
headers, bot-token rotation) · restore drill + data export (JSON + files) ·
performance pass · optional: PWA share target or tiny native Android app if the
Telegram flow ever feels limiting.
**Exit test:** full export restores on a clean machine; dashboard loads <2s on phone.

## Dependencies

**Before M0 (decisions/purchases — the only things Claude Code can't do for you):**
- [ ] Docker Desktop (WSL2 backend) installed and running on the laptop
- [ ] Tailscale on laptop + phone (already in use for HuziOS — verify it works on mobile data)
- [ ] OpenAI API key with a **one-time ~$10 topup** and auto-recharge OFF
- [ ] **Backblaze B2** bucket for backups (free tier covers it — offsite so a laptop
      disk failure can't take the data with it)
- [ ] Claude Pro sub stays active (mentor mode rides on it via MCP connector, M6)
- [ ] Telegram bot created via @BotFather (free, 2 minutes; token goes in `.env`)

**Deferred purchases (no longer pre-M0, v1.5):**
- Domain ~$10/yr on Cloudflare Registrar + account 2FA hardening → **at M5** (email ingestion is the only feature needing it)
- Hetzner CX22 ~€4.35/mo → **on graduation only** (runbook written in M0, trigger = real laptop-uptime pain)

**Internal blockers:**
- M1 blocks everything (no data → nothing downstream matters)
- M3's AI Router (budget cap) must exist before M6 (smart-model enrichment) to prevent cost surprises
- Embedding model choice is **locked at M2** (384-dim multilingual-e5-small) — changing later means re-embedding everything (cheap at personal scale, but a migration)

## Estimated time

Build mode is "Claude Code writes it, Huzi reviews" — so estimates are in
review-sessions, not coding-hours. Assume 2–4 sessions/week. **Low confidence overall**
(solo, first project of this shape); M0–M2 estimates are firmer than M5–M7.

| Milestone | Sessions | Calendar (rough) |
|---|---|---|
| M0 Infra | 2–3 | Week 1 |
| M1 Capture | 2–3 (halved in v1.4 — Telegram does the hard parts) | Week 1–2 |
| M2 Search | 3–4 | Weeks 2–3 |
| M3 Auto-org | 3–4 | Weeks 3–4 |
| M3.5 Glass UI port + vault index | 3–5 (panel data layers are rewrites, not ports) | Weeks 4–6 |
| M4 Linking | 4–5 | Weeks 6–7 |
| M5 Briefing/rediscovery | 4–5 | Weeks 8–9 |
| M6 Enrichment/mentor | 4–6 | Weeks 10–12 |
| M7 Hardening | 3–4 | Week 13+ |

MVP you'd actually use daily = **M0–M3, roughly a month**. Everything after runs on real
accumulated data, which is exactly what M4–M6 need to be tuneable.

## Risks

| Risk | Why it's real here | Mitigation |
|---|---|---|
| **Abandonment mid-build** (the spec itself jokes about a 50% abandonment pattern) | 24-week horizon, solo | Plan is cut so M1–M3 (~1 month) already delivers daily value; every milestone is independently useful |
| **Capture friction kills adoption** | If sharing takes >10s or fails offline, you stop using it and the data moat never forms | Telegram bot in M1 — offline queueing and retry inherited from a battle-tested app you already use daily |
| **Telegram privacy** (v1.4) | Bot chats are not E2E-encrypted; every bot capture transits and rests on Telegram's cloud before reaching the VPS | Dashboard quick-note is the permanent private path for finance/health/personal; the split is a per-capture habit, revisited at the M3 checkpoint |
| **Telegram platform dependency** (v1.4) | Account ban, bot API outage, or policy change breaks primary capture | Ingestion API is channel-agnostic — PWA share target (V3) is a contained add-on, not a rewrite; quick-note keeps working through any Telegram outage |
| **API cost creep** | No local tier to absorb load; every smart feature is metered | Budget-cap table + router before smart-model features; provider console spend limit as backstop |
| **Laptop availability** (v1.5) | Laptop asleep/off = no bot polling, no briefing, no dashboard; **Telegram's update queue drops unfetched messages after ~24h** — a weekend away can silently lose queued captures from the bot's view (they stay visible in your own chat; re-forward to recover) | Windows wake timers + Task Scheduler (set up in M0); poll-on-wake catches anything <24h old; repeated pain = the documented VPS graduation trigger, an afternoon move |
| **WSL2/Docker-on-Windows friction** (v1.5) | File-watcher quirks across the WSL boundary, Docker Desktop updates, Windows Update reboots | Vault + /data live on the WSL filesystem side; `restart: always` on all services; graduation runbook is the escape hatch |
| **HuziOS port undersold as a "port"** (v1.5) | Only CSS/layout/shell transfer; every panel's data layer is a rewrite against Engram's API — could silently eat weeks | Costed honestly as M3.5 (3–5 sessions) after MVP value exists; strangler fig means a half-done port never breaks the daily driver |
| **Classification quality disappoints** ("zero manual organization" is the promise) | Cheap fast models on terse captures can misfile | Feedback buttons from M3 day one; misfiles feed prompt tuning and the eval set; escalate low-confidence to the smart model within budget |
| **Provider swap regresses quality silently** | Prompts are tuned against one model's behavior; a swap (OpenAI → Kimi) can misfile without erroring | Eval set (M3) re-run gates every provider/model change; keep old config until new one passes baseline |
| **Mentor depends on the Claude Pro sub** | Cancel Pro and mentor mode loses its interface | MCP is an open standard — any MCP client can connect to the same server (e.g. Claude Code, or a self-hosted agent runtime like ZeroClaw); a chat UI on the cheap API is a contained V3 fallback |
| **Data loss** (spec anti-pattern #10: "this is someone's life") | A single laptop is an even more fragile single point of failure than a VPS (theft, spills, disk death) | Encrypted nightly offsite backups to B2 from M0, restore actually tested in M0 and re-drilled in M7 |
| **Cloudflare concentration** (v1.3; mostly dissolved by v1.5) | Laptop-first removes Cloudflare from the critical path until M5 (email only) | Backups stay at Backblaze B2 regardless; 2FA-harden the account when the domain is bought at M5 |

## MVP

**M0–M3:** capture anything from the phone from anywhere (Telegram bot + private
quick-note), exact-dedup, hybrid semantic+keyword search, auto-classification with a
budget-capped fast model, browsable by category/tag. One laptop, one database, one bot,
one (deliberately minimal) dashboard — HuziOS keeps running untouched beside it as the
daily driver. This tests the core assumption —
*"if capture is frictionless and retrieval is smart, I'll actually use it"* — for
pocket change in API calls and €0 hosting.

## V2

**M3.5–M6:** HuziOS glass UI ported onto Engram (strangler fig), Obsidian vault indexed
read-only into search, entity/relation graph in Postgres, related-capture surfacing,
projects, morning briefing, rediscovery, email ingestion (first purchase: domain),
selective deep enrichment, mentor chat over your own data via claude.ai MCP connector,
first insights.

## V3 (rough)

PWA Web Share Target, native Android app + home-screen widget, browser extension,
in-bot search commands (/search, /recent), spaced-repetition study system + quiz
generation (spec §9), finance pattern tracking, reflection engine (spec §11),
notification intelligence, third-party API/webhooks — and only if data volume ever
demands it: dedicated vector/graph/search stores.

---

## Appendix A: Spec deviations & fixes

Flags found in the source spec and how this plan resolves them:

1. **Embedding dimension mismatch** — spec schema says `Vector(1536)` but its own stack
   picks `all-MiniLM-L6-v2` (384-dim). **Fix:** 384-dim everywhere; pgvector column
   `vector(384)`; model upgraded to `multilingual-e5-small` in v1.2 (same dims,
   handles Malay+English). Migration path documented if we ever switch models.
2. **Qdrant AND pgvector both specced** — **Fix:** pgvector only.
3. **Invalid SQL** — double-quoted string literals (`DEFAULT "{}"`, `IN ("HIGH",...)`)
   and malformed GIN index syntax. **Fix:** rewrite schema with single quotes, correct
   `USING GIN (column)` syntax, during M1.
4. **Duplicate graph layers** (Neo4j + Postgres relation tables). **Fix:** Postgres
   tables are the single source of truth; Neo4j dropped.
5. **"IMAP webhook"** doesn't exist. **Fix (v1.2):** Cloudflare Email Routing → Worker →
   ingestion API — genuinely real-time, credential-less; IMAP polling kept only as a
   documented fallback for domainless setups.
6. **Stale, hardcoded model names** (`claude-sonnet-4`, `gpt-4o`, `gemini-1.5-pro`).
   **Fix (v1.1):** no model names in code at all — abstract `LLM_FAST`/`LLM_SMART` slots
   behind an OpenAI-compatible client, provider and models set in `.env`
   (see [AI provider portability](#ai-provider-portability)).
7. **Local LLM tier assumed GPU** — user has none. **Fix:** tier 2 becomes the cloud fast
   model; heuristics tier absorbs everything it can for free; hard daily budget cap enforced
   in the router (spec's throttling ladder kept: 80% → downgrade, 100% → heuristics only).
8. **Six datastores for one user** contradicts the spec's own anti-pattern #9
   ("don't over-engineer v1"). **Fix:** Postgres + disk + Caddy; queue via
   `SKIP LOCKED`; Redis/Celery/Meilisearch/MinIO deferred behind real need.
9. **Multi-user auth (Supabase/Clerk)** for a single-user system. **Fix:** single-user
   JWT (one password + long-lived refresh), `user_id` kept in schema for optionality.
10. **iOS/Android native app in Phase 1** — **Fix:** Android PWA Web Share Target at M1
    (works day one, no store, no Mac); native app demoted to optional M7.
11. **Screenshot accessibility-service monitoring** — Play-policy minefield and not
    needed for a personal sideload; **deferred to V3**, share-sheet covers screenshots fine.
12. **24-week team-sized roadmap** — **Fix:** re-phased to milestone/review-session
    model above; MVP value lands at ~1 month instead of week 4 of 24.

## Appendix B: Changelog

- **v1.5 (2026-07-21):** HuziOS/Engram convergence + hosting flip, after an adversarial
  self-review of the "HuziOS as frontend, Engram as backend" rework idea. Decisions:
  (1) **Laptop-first, €0 hosting** — Telegram long-polling (no webhook/public IP),
  Tailscale for phone access (as HuziOS today), Tailscale Funnel for the two genuinely
  public surfaces (M5 email target, M6 claude.ai MCP connector); Hetzner VPS demoted to
  a documented graduation runbook; domain purchase deferred to M5; Cloudflare
  concentration risk largely dissolved. (2) **HuziOS untouched through M3** (ACCA-exam
  critical daily driver; avoids the ghost-town/parallel-rework trap) — glass UI ported
  at new **M3.5**, honestly costed as a per-panel data-layer rewrite; chat stays on
  local Claude Code until the M6 mentor proves out. (3) **Vault: index, don't migrate**
  — Engram can't replace Obsidian's editor/SR-flashcards/kanban without rebuilding
  Obsidian, so the vault stays source of truth and is indexed read-only into search at
  M3.5 (trivial while co-located on the laptop); full migration reconsidered only at V3.
  Also evaluated and rejected: Supabase (re-confirmed — pauses/pricing, and it can't
  host the Python worker so the box remains needed anyway), Google Play/Search Console
  (private PWA needs neither), Coolify (RAM overhead on a small box; compose + runbook
  suffices). New risks: laptop availability + Telegram's ~24h update-queue expiry,
  WSL2 friction, port-scope honesty.
- **v1.4 (2026-07-20):** Capture re-architected around Telegram after evaluating
  ZeroClaw (github.com/zeroclaw-labs/zeroclaw — platform rejected as Jarvis-first,
  three ideas adopted). Telegram bot becomes primary capture channel (M1 halved:
  offline queue/retry/compression inherited; voice arrives day one; briefing pushes
  via bot, ntfy dropped). Dashboard quick-note = permanent private path (bot chats
  are not E2E-encrypted — new privacy + platform-dependency risk rows). PWA share
  target deferred to V3. Decision checkpoint at M3: "Telegram for everything" vs
  hybrid is a per-capture habit, not an architecture lock. Also adopted: provider
  fallback chains (LLM_*_FALLBACK slots, portability rule 1); ZeroClaw named as
  concrete mentor-fallback MCP client.
- **v1.3 (2026-07-20):** Hosting decision finalized after market scan (Hetzner +30–40%
  price hikes and Oracle free-tier halving, both mid-2026): Hetzner CX22 Germany
  (~€4.35/mo) as the API host; PWA served from Cloudflare Pages KL edge for free,
  making paid Singapore hosting unnecessary; Oracle free ARM demoted to optional
  scout box. Full-plan audit fixes: backups moved R2 → Backblaze B2 to break the
  Cloudflare single-account concentration (new risk entry + account-hardening
  dependency); briefing cron pinned to Asia/Kuala_Lumpur (German server would fire
  at ~1pm MY); stale "$20/mo" goal and "€4.50" price corrected; stale "mentor" removed
  from LLM_SMART env comment.
- **v1.2 (2026-07-20):** Cost + privacy restructure after user challenge. Mentor mode
  moved off the API entirely — Engram exposes a remote MCP server and claude.ai (existing
  Pro sub) becomes the mentor interface; M6 no longer builds a chat UI. Pipeline provider
  locked to no-train-on-API-traffic providers (start: OpenAI), free tiers banned for
  capture content (data-policy gate, portability rule 5). Budget model changed to
  one-time ~$10 topup (~a year) instead of $5–10/mo. Embeddings switched
  MiniLM → multilingual-e5-small (mixed Malay+English captures). Email ingestion
  switched IMAP polling → Cloudflare Email Routing webhook. TikTok/IG capture depth
  expectation documented. Frontend builds moved to GitHub Actions (VPS RAM).
- **v1.1 (2026-07-20):** AI provider made replaceable — OpenAI-compatible client with
  `LLM_FAST`/`LLM_SMART` env slots, per-model pricing config, portable JSON output,
  eval-set swap gate. Starting provider: OpenAI (was: hardcoded Claude); Kimi/Moonshot
  and Claude remain drop-in candidates.
- **v1.0 (2026-07-20):** Initial audited plan from Personal_OS_Technical_Specification_v1.0.

## Appendix C: Next step

Per the planning skill: this plan defines *what* and *in which order*. The first build
session (M0) should start by running the **system-architect** pass to fix the concrete
repo layout, docker-compose file, corrected Postgres schema DDL, and API surface — then
scaffold it. Say **"start M0"** in a new session in this folder to kick off.
