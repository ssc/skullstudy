# SkullStudy — Supabase schema

Backend for the hosted SkullStudy replay site. Supabase project:
`zqywiqsgkmgdmriqplrz` (`https://zqywiqsgkmgdmriqplrz.supabase.co`).
Static frontend is hosted on Cloudflare Pages (`skullstudy.pages.dev`).

## `public.games` table

One row per game. Created with:

```sql
create table public.games (
  id bigint generated always as identity primary key,
  game_key text not null unique,
  source text not null,
  source_game_id text not null,
  game_date timestamptz not null,
  upload_date timestamptz not null,
  hashid text unique,
  home_team text,
  away_team text,
  home_coach text,
  away_coach text,
  home_race text,
  away_race text,
  home_score int,
  away_score int,
  competition text,
  original_path text,
  turns_path text,
  steps_path text,
  display_path text,
  created_at timestamptz not null default now()
);

alter table public.games enable row level security;
create policy "Public read access" on public.games for select using (true);
```

Writes go through the `service_role` key, which bypasses RLS. Reads (the
dropdown and deep links) use the public `anon` key under the read policy.

### Columns

| Column | Meaning |
| --- | --- |
| `id` | Canonical game ID. Monotonically incrementing integer (1, 2, 3, …), auto-generated. This is the source of truth. |
| `game_key` | Dedup key: `{source}\|{game_date}\|{coach_a}\|{coach_b}`. Source is `bb3` or `fumbbl`; `game_date` is the full UTC timestamp (ISO 8601); coaches are lowercased and sorted alphabetically. Unique. |
| `source` | `bb3` or `fumbbl`. |
| `source_game_id` | The platform's own ID: a FUMBBL replay ID (e.g. `1954282`) or a BB3 game id (e.g. `bbr-20261110-0728`). |
| `game_date` | When the game was played, UTC (`timestamptz`). BB3 `<Date>` is already UTC and is the game **start** time; FUMBBL API date/time is converted from Europe/Stockholm at import. |
| `upload_date` | When the game was imported, UTC. Stored in the internalRep too, so a re-import can preserve it. |
| `hashid` | The public URL ID: the integer `id` Hashids-encoded with `min_length=5` (5 chars, covers ~3.7M games). Stored as a column so the frontend looks games up by it directly. |
| `home_team`, `away_team` | Team names as displayed (e.g. `S10.OL Mosswood Mystics`). |
| `home_coach`, `away_coach` | Coach names (e.g. `codegoon`). |
| `home_race`, `away_race` | Race display names (e.g. `Wood Elf`, `Necromantic Horror`). |
| `home_score`, `away_score` | Final score. |
| `competition` | e.g. `Open Ladder`. |
| `original_path` | Storage path of the original file (`.bbr` or FUMBBL replay JSON) in the private `games` bucket. |
| `turns_path` | Storage path of the internalRep turns file in the private `games` bucket. |
| `steps_path` | Storage path of the internalRep steps file in the private `games` bucket. |
| `display_path` | Storage path of the displayRep file in the public `games-public` bucket. |
| `created_at` | Row insert time. |

## Game IDs

There are four identifiers, each with one job:

- `id` — the canonical internal ID (integer, 1, 2, 3, …). Source of truth.
- `game_key` — the dedup key (platform + UTC date + coaches). Detects
  whether a game was already uploaded. Unique per game.
- `source_game_id` — the platform's own ID (FUMBBL replay ID / BB3 game id).
  Used to cross-reference the platform, not for URLs on the hosted site.
- `hashid` — the public URL ID. The integer `id` Hashids-encoded with
  `min_length=5`, used as `#game=<hashid>&move=<n>`. Hides the game count
  and stops blind enumeration by sequential guessing.

Hashids is reversible: the same integer + the same salt always produces the
same hashid, and the hashid decodes back to the integer. The integer `id` is
the source of truth, so a hashid can always be recomputed from `id` + salt.

`min_length` is a one-time choice. Changing it changes the encoding and
breaks decoding of strings encoded under the old setting, unless both
decoders are kept and dispatched on length (a short string is the old
format, a longer one the new format). The salt must never change.

### Where the salt lives

The salt lives **only** in the Cloudflare Container that does upload
processing (as an environment secret). That is the only place a hashid is
computed: at upload time, when a new game gets its integer `id` turned into
a hashid and stored in the table. Everything else (frontend, OG function)
treats the hashid as an opaque string already in the table and looks games
up by it. The salt is never in the repo, never in the frontend, and never
in the database.

## Storage buckets

| Bucket | Visibility | Contents |
| --- | --- | --- |
| `games` | private | Original files and internalReps (turns + steps). Never sent to the browser. |
| `games-public` | public | displayReps only. The frontend fetches these directly. |

File layout per game (under the game key or source id):

- `original` — the original file: `.bbr` (BB3) or the saved FUMBBL replay
  JSON stream. Private.
- `turns.json` / `steps.json` — the internalRep (turns = per-turn-end
  snapshots, steps = per-step deltas). Private. See
  `../fumbbl-replay/INTERNAL_FORMAT.md` in the tooling repo.
- `display.json` — the displayRep: same game data, fully resolved to
  display names (race/position/skill names, chess labels), so the frontend
  only displays and never needs a decoding table. Public. Generated by
  `build_display_rep.py`; spec in `../fumbbl-replay/DISPLAY_FORMAT.md`.

There is no separate metadata file. The table row is the metadata catalog:
it holds the game key, source, dates, and the three file paths. The upload
time already lives in the internalRep's `upload_date`, so there is no third
copy of it.

## Frontend reads

- Dropdown: one query on `public.games` ordered by `game_date` desc,
  using the team/coach/race/score/competition columns. No displayRep fetch
  needed to build the list.
- Game view: look up the row by `hashid`, fetch `display_path` from the
  public bucket, render it. The original and internalRep are never
  requested by the browser.
