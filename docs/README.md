# SkullStudy docs

Design documentation for the SkullStudy replay site.

- [supabase-schema.md](supabase-schema.md) — The `games` table, game IDs
  (integer `id`, dedup `game_key`, platform `source_game_id`, Hashids
  `hashid`), storage buckets, and how the frontend reads them.
- [internal-format.md](internal-format.md) — The internal representation:
  one format for BB3 and FUMBBL, turns (snapshots) + steps (deltas),
  coordinates, block dice, dates, and the dedup key.
- [display-format.md](display-format.md) — The displayRep: a self-contained,
  fully-resolved JSON per game (all IDs resolved to display names) so the
  frontend only displays and never needs a decoding table.
- [product-plan.md](product-plan.md) — Product plan: phases (shareable
  scrubber, manual entry, graded problems, AI/camera), hosting, and
  monetization notes.
- [deployment-plan.md](deployment-plan.md) — Deployment plan: Cloudflare
  Pages + Supabase, the Cloudflare Container upload path for BB3 and
  FUMBBL, and the staged rollout.
