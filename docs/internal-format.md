# Internal representation — format notes

The internal representation is the single format every importer emits
and the renderer (SkullStudy scrubber) reads. `turns_*.json` holds a
full snapshot at each turn end; `steps_*.json` holds per-step deltas
(step 0 carries a `full` snapshot). Source-specific handling lives
only in the importer, never in the renderer.

## Top-level metadata

Both files carry these metadata fields (order as written):

| field | meaning |
|---|---|
| `format_version` | format version (currently `1`) |
| `vocab_version` | vocabulary/display-table version (BB3 only) |
| `ruleset` | e.g. `BB2020` (BB3) / `BB2025` (FUMBBL) |
| `ruleset_evidence` | how the ruleset was determined |
| `source` | `bb3` or `fumbbl` |
| `importer_version` | importer name + version that wrote the file |
| `source_game_id` | game id in the source's own terms (`bbr-YYYYMMDD-HHMM` for BB3; replay id for FUMBBL steps) |
| `game_date` | game date/time, canonical UTC, ISO 8601 (see below) |
| `upload_date` | date/time the game was uploaded/imported, canonical UTC, ISO 8601, set at import time (see below) |
| `game_key` | deterministic cross-system unique game id for duplicate detection (see below) |
| `source_date` | the original timestamp string exactly as the source gave it (provenance) |
| `source_timezone` | the source timezone as a string (provenance) |
| `replay_version` / `id_map_version` / `id_map_source_hash` | BB3 only |

## Game date vs upload date (decided Nov 13, 2026)

The game date and the upload date are two distinct values:

- `game_date`: when the game was **played** — the source timestamp
  normalized to UTC, ISO 8601, e.g. `2026-11-10T08:54:00Z`. This is
  the source timestamp; a user may upload a replay days after the
  game (e.g. a Nov 10 game uploaded Nov 13), so it must not be
  confused with the upload time.
- `upload_date`: when the game was **uploaded/imported** to the site
  — set at import time, ISO 8601 UTC, e.g. `2026-11-13T16:20:00Z`.
  The importer records the current UTC time when it runs (a single
  import run uses one timestamp for the turns and steps files it
  writes, so the pair always agrees). An explicit upload timestamp
  can be passed to the importer (`--upload-date`) for a batch
  re-import, so all files in the batch share one value.

Field-name note: the user asked for these as `gameDate` /
`uploadDate`, but the internal rep uses snake_case for top-level
keys, so they are `game_date` / `upload_date`. (An earlier draft
named the game date `played_at_utc`; it was renamed to `game_date`
when the upload date was added, so the pair reads as a unit.)

## Game date/time — canonical UTC (decided Nov 13, 2026)

Game date/time is stored as a **canonical UTC timestamp**, normalized
at import time. The renderer and downstream consumers never reason
about source timezones.

- `game_date`: ISO 8601 UTC, e.g. `2026-11-10T08:54:00Z`.
- `source_date`: the original timestamp string exactly as the source
  gave it. BB3: the `.bbr` `<Date>` string, e.g. `2026-11-10_08-54`.
  FUMBBL: the API `date` + `time`, e.g. `2026-11-04 05:16:53`.
- `source_timezone`: BB3 `"UTC"`; FUMBBL `"Europe/Stockholm"`.
- The field is neutrally named `game_date`: it asserts nothing
  about whether the timestamp is the game start or end time.

## Duplicate detection — `game_key` (decided Nov 13, 2026)

`game_key` is a deterministic cross-system unique id for a game,
used to detect whether a game has already been uploaded. Its
components are platform + game date + coach names, normalized so
the same game always produces the same key:

- **Format:** `{source}|{game_date}|{coach_a}|{coach_b}`
  where `source` is the existing `source` value (`bb3` or `fumbbl`),
  `game_date` is the full UTC timestamp (not just the calendar date,
  so two games between the same coaches on the same day do not
  collide), and `coach_a` / `coach_b` are the two coaches' names
  lowercased and sorted alphabetically (so home/away order does not
  change the key). The separator is `|`. A missing coach name falls
  back to the team name, then to `unknown`.
- **Examples:**
  - BB3, Mosswood (codegoon) vs Clactonian Crypt Creepers
    (AndyPandy), Nov 10 08:54 UTC:
    `bb3|2026-11-10T08:54:00Z|andypandy|codegoon`
  - FUMBBL, Mosswood (codegoon) vs Moooooo (Wotan83), Oct 18
    07:52:21 UTC: `fumbbl|2026-10-18T07:52:21Z|codegoon|wotan83`
- The key is computed by the importer from the same values it
  writes (`source`, `game_date`, coach names), so it is stable
  across re-imports of the same game.

Per-source timezone (verified Nov 13, 2026):

- **BB3**: the `.bbr` has a root-level `<Date>` element,
  `YYYY-MM-DD_HH-MM`, which is already UTC. (The importer used to
  ignore it, which is why the format had no date at all.) The file
  also carries a server match id, `<MatchId>` inside
  `CompetitionInfos` (Base64 UUID) — that is the id for any server
  lookup, NOT the filename UUID, which is the local replay's own id
  and does not appear in the file.
- **FUMBBL**: the public match API
  (`https://fumbbl.com/api/match/get/<matchId>`) returns a single
  `date` + `time` — there is no start/end pair. That timestamp is
  Europe/Stockholm server local time: CET (UTC+1) in winter, CEST
  (UTC+2) in summer — NOT UTC. Verified live Nov 13, 2026: FUMBBL
  "Server time" read 17:10 while UTC was 16:10. The importer
  converts Europe/Stockholm -> UTC with `zoneinfo` (standard
  library), which handles the CET/CEST DST switch from the date —
  never a hardcoded offset. The FUMBBL replay stream has no
  wall-clock timestamps (only the in-game turn clock), so the date
  comes from the API match JSON, passed to the importer with
  `--match-json <path>` (a saved `/api/match/get/<id>` response).

## Resolved: BB3 timestamp meaning is the START time

The BB3 `<Date>` is the game **start** time (settled Nov 13, 2026 by
the user's clock test). The user reported a game finished ~5 minutes
before 09:59 PST (so ~09:54 PST = 17:54 UTC); that game's `<Date>` is
`2026-11-13_16-35` = 16:35 UTC = 08:35 PST, ~79 minutes before the
finish. A `<Date>` that were the end time would have read ~17:54 UTC.
The game is a full 16-turn game (33 turn entries), so a ~79-minute
duration is a normal full game. Caveat: the test pins `<Date>` to the
start side (kickoff or match creation, within a few minutes) — the
user reported only the finish time, not the kickoff time, so the
exact kickoff-vs-creation offset is not measured. `game_date` is
therefore the game start time in UTC. The FUMBBL single timestamp's
start-vs-end meaning is likewise unverified.

## Display representation (displayRep) — see DISPLAY_FORMAT.md

The displayRep is the third file in the per-game scheme: a single,
self-contained, fully-resolved JSON per game
(`data/display_<game_id>.json`), derived deterministically from the
internalRep + the ID maps by `build_display_rep.py`. It resolves
every ID/code to its display string (notably BB3 skill codes to
names, which fixes the roster popup's `Skill 1` gap) so the frontend
only displays and never needs a decoding table. There is no separate
metadata file; the Supabase table row (Step 2) is the metadata
catalog. Full spec: `DISPLAY_FORMAT.md`.
