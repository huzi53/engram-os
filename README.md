# Engram

> *engram (n.) — the physical trace a memory leaves in the brain.*

A personal memory OS: capture anything from your phone in under 10 seconds, have it
automatically organized, linked, and searchable by meaning — self-directed, hosted for
€0/month on the laptop you already own, with zero manual filing.

**Capture → Store → Organize → Link → Understand → Reason → Act**

## Status

📋 **Planning complete (v1.5) — build not started.**
The full audited build plan is in [plan.md](plan.md) (milestones M0–M7, architecture,
budget, risks, and the 12 spec flags that were found and fixed). v1.5 adds the
HuziOS/Engram convergence: laptop-first €0 hosting, HuziOS glass UI as the eventual
interface (M3.5), Obsidian vault indexed rather than migrated.

## Architecture at a glance

- **One box (yours):** the laptop — Docker Compose via WSL2 · Caddy · FastAPI · worker · Telegram bot poller. Phone reaches the dashboard over Tailscale; Tailscale Funnel exposes the two surfaces that need public HTTPS (email ingestion, claude.ai MCP connector). A Hetzner-VPS graduation runbook is documented from M0 — an afternoon move, made only when laptop uptime actually hurts
- **One database:** PostgreSQL 16 + pgvector (vectors, full-text search, graph tables, job queue)
- **Capture:** Telegram bot via long-polling (text, links, photos, files, voice — offline queueing free, no public IP needed) + dashboard quick-note as the direct private path; no app store, no native build
- **Interface:** HuziOS's glass UI, ported panel-by-panel onto Engram's API at M3.5 (strangler fig — HuziOS stays the untouched daily driver until then); the Obsidian vault stays source of truth for study content and is indexed **read-only** into the same search space
- **AI tiers:** Python heuristics (free) → cloud fast model (classify) → cloud smart model (enrich/briefing), hard daily budget cap
- **AI provider:** replaceable by design — any OpenAI-compatible API via `.env` config, restricted to providers that don't train on API traffic (start: OpenAI); swaps gated by a classification eval set
- **Mentor mode:** Engram exposes a remote MCP server; claude.ai (existing Pro sub) is the mentor interface — zero API cost for reasoning
- **Embeddings:** multilingual-e5-small (Malay + English), 384-dim, CPU
- **Running cost:** €0/mo hosting + one-time ~$10 API topup (~a year); domain ~$10/yr only at M5 (email); backups on Backblaze B2 free tier

## Roadmap

| Milestone | Delivers |
|---|---|
| M0 | Infra live on the laptop: Docker, Tailscale HTTPS, auth, B2 backups |
| M1 | Capture via Telegram bot (long-polling) from any device, <10s |
| M2 | Hybrid semantic + keyword search |
| M3 | Auto-classification (budget-capped) |
| M3.5 | HuziOS glass UI ported onto Engram + Obsidian vault indexed read-only |
| M4 | Entity graph, related captures, projects |
| M5 | Morning briefing, rediscovery, email ingestion |
| M6 | Deep enrichment, mentor via claude.ai MCP connector |
| M7 | Hardening, export, voice, optional native app |

MVP = M0–M3, ~1 month of build sessions.
