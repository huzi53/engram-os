# Engram

> *engram (n.) — the physical trace a memory leaves in the brain.*

A personal memory OS: capture anything from your phone in under 10 seconds, have it
automatically organized, linked, and searchable by meaning — self-directed, hosted for
under $20/month, with zero manual filing.

**Capture → Store → Organize → Link → Understand → Reason → Act**

## Status

📋 **Planning complete — build not started.**
The full audited build plan is in [plan.md](plan.md) (milestones M0–M7, architecture,
budget, risks, and the 12 spec flags that were found and fixed).

## Architecture at a glance

- **One box:** Hetzner CX22 VPS (Germany) · Docker Compose · Caddy (HTTPS) · FastAPI · worker — PWA served free from Cloudflare Pages (KL edge) so it's fast from Malaysia
- **One database:** PostgreSQL 16 + pgvector (vectors, full-text search, graph tables, job queue)
- **Capture:** Telegram bot (text, links, photos, files, voice — offline queueing for free) + dashboard quick-note as the direct private path; no app store, no native build
- **AI tiers:** Python heuristics (free) → cloud fast model (classify) → cloud smart model (enrich/briefing), hard daily budget cap
- **AI provider:** replaceable by design — any OpenAI-compatible API via `.env` config, restricted to providers that don't train on API traffic (start: OpenAI); swaps gated by a classification eval set
- **Mentor mode:** Engram exposes a remote MCP server; claude.ai (existing Pro sub) is the mentor interface — zero API cost for reasoning
- **Embeddings:** multilingual-e5-small (Malay + English), 384-dim, CPU
- **Running cost:** ~€4.35/mo VPS + ~$10/yr API topup + ~$10/yr domain; backups on Backblaze B2 free tier (kept off Cloudflare on purpose)

## Roadmap

| Milestone | Delivers |
|---|---|
| M0 | Infra live: VPS, HTTPS, auth, backups |
| M1 | Capture via Telegram bot from any device, <10s |
| M2 | Hybrid semantic + keyword search |
| M3 | Auto-classification (budget-capped) |
| M4 | Entity graph, related captures, projects |
| M5 | Morning briefing, rediscovery, email ingestion |
| M6 | Deep enrichment, mentor via claude.ai MCP connector |
| M7 | Hardening, export, voice, optional native app |

MVP = M0–M3, ~1 month of build sessions.
