# TODO

Running checklist for the Engram-OS + HuziOS repo work. Updated as steps complete.

## Done — 2026-07-21

- [x] Pulled `origin/main` (v1.4 → v1.5, "HuziOS/Engram convergence"); merged clean
      with local uncommitted M0 footnotes in `plan.md` (no conflict markers).
- [x] Found `huzi53/HuziOS` was **public** with the full Obsidian vault tracked in git
      (personal ACCA notes + copyrighted course PDFs/PPTs) — flagged to user.
- [x] Set `huzi53/HuziOS` to **private**.
- [x] Created `huzi53/huzios-public` — clean-history export of app code + docs only
      (`app/`, `CLAUDE.md`, `journey-into-HuziOS.md`, `start.cmd`, `start-hidden.vbs`).
      `vault/`, `backups/`, `.claude/`, `plans/` never entered its history.
- [x] Scanned both exports for hardcoded secrets/API keys/tracked `.env` — none found.
- [x] Added MIT `LICENSE` to `engram-os` and `huzios-public`.
- [x] Added GitHub topics to both public repos for discoverability.
- [x] Committed + pushed the v1.5 merge to `engram-os`.

## Pending — needs a decision before executing (see chat report)

- [ ] Reconcile the M0 scaffold (`docker-compose.yml`, `Caddyfile`, `.env.example`,
      `README.md` "Run it" section, `scripts/backup.sh` cron line) — all still built for
      the **old VPS/domain/Cloudflare M0** — against `plan.md` v1.5's **laptop-first M0**
      (Tailscale HTTPS, no domain/VPS, Windows Task Scheduler instead of crontab).
- [ ] Write the VPS-graduation runbook plan.md v1.5 requires documenting from day one.
- [ ] Decide fate of `plans/002-m0-infra-scaffold.md` (superseded doc vs. amend in place).
- [ ] Commit the M0 scaffold itself (`app/`, `db/`, `docker-compose.yml`, `Caddyfile`,
      `scripts/`, `.github/`, `.env.example`) — currently untracked, held back until the
      laptop-first rework above lands so we don't commit code about to be replaced.
