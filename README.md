# SkullStudy

A Blood Bowl replay viewer: step through FUMBBL and Blood Bowl 3 games
turn by turn and move by move, in the browser — no game client needed.

Live demo: https://skullstudy.pages.dev

## What's here

- `index.html` — the whole viewer, self-contained (2.6 MB). The five demo
  games are baked in:
  - FUMBBL — Mosswood Mystics (Wood Elf) 0–1 vs Chaos Dwarfs
  - FUMBBL — Mosswood Mystics WC2027b 1–2 vs Russell's Orcs
  - FUMBBL — Mosswood Mystics (Black Orcs) 2–0 vs ancientspear
  - BB3 — Mosswood Mystics (Wood Elf) 3–1 vs AndyPandy's Necromantic Horror
  - BB3 — Chaos Renegade 1–2 vs Mosswood Mystics
- Game data in the internal turns + steps format lives in
  `~/workspace/fumbbl-replay/data/` on the author's machine (not in this
  repo yet); the importers that produce it are `bb3_import.py`,
  `parse_replay.py`, and `steps_replay.py` in that same folder.

## Deploy

This repo is connected to Cloudflare Pages with Git integration.
Every push to `main` redeploys the site. No build step: the framework
preset is unset, the build command is blank, and the build output
directory is the repo root (`/`).

## Status

Phase 1 (static demo) only. Uploads, login, and the games table come
later — see `SkullStudy-deployment-plan.md` in the BB3 Play Logs goal
folder for the full plan (Cloudflare Pages + Worker + Container +
Supabase).

## Notes

- The page title still reads "FUMBBL Replay Scrubber" — renaming the
  header to SkullStudy is a pending item in the BB3 Play Logs chat.
- Viewing is free; nothing in this repo contains player IPs (the
  importer strips them) or FUMBBL auth tokens.
