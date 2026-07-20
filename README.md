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

- **One box:** Hetzner VPS · Docker Compose · Caddy (HTTPS) · FastAPI · worker
- **One database:** PostgreSQL 16 + pgvector (vectors, full-text search, graph tables, job queue)
- **Capture:** Android PWA with Web Share Target — no app store, no native build
- **AI tiers:** Python heuristics (free) → Claude Haiku (classify) → Claude Sonnet (enrich/mentor), hard daily budget cap
- **Embeddings:** all-MiniLM-L6-v2, 384-dim, CPU

## Roadmap

| Milestone | Delivers |
|---|---|
| M0 | Infra live: VPS, HTTPS, auth, backups |
| M1 | Capture from phone, anywhere, <10s |
| M2 | Hybrid semantic + keyword search |
| M3 | Auto-classification (budget-capped) |
| M4 | Entity graph, related captures, projects |
| M5 | Morning briefing, rediscovery, email ingestion |
| M6 | Deep enrichment, mentor chat over your own data |
| M7 | Hardening, export, voice, optional native app |

MVP = M0–M3, ~1 month of build sessions.
