# Display representation — format notes (displayRep)

The displayRep is the third file in the per-game scheme (see "File
scheme" below). It is a single, self-contained, fully-resolved JSON
file per game, derived deterministically from the internalRep
(turns + steps) plus the ID maps. It contains the same game data
(turns, steps, events, rosters, board states) but with every ID/code
resolved to its display string, so the frontend only displays and
never needs a decoding table. The ID maps and the resolution logic
stay server-side; the displayRep carries no lookup tables.

Generator: `build_display_rep.py` (version `build_display_rep.py
1.0.0`). It reads a game's internalRep turns/steps JSON (+ the ID
map, and for FUMBBL the saved replay stream) and writes
`display_<game_id>.json`. Deterministic: same input produces
byte-identical output (sorted keys, 2-space indent, no wall-clock
timestamps; game_date/upload_date are copied from the internalRep).

## File scheme (decided Nov 13, 2026)

Each game has three stored files, plus one metadata row:

| file | role | example |
|---|---|---|
| (a) `originalFile` | The source as received — the source of truth. BB3: the `.bbr`. FUMBBL: the replay JSON stream (`.jsonl`). The raw `.bbr` stays private (it contains player IPs); the FUMBBL JSON is safe. | `2026-11-13_16-35_….bbr`, `replay_stream_1954282.jsonl` |
| (c) `internalRepresentation` | The existing `turns_*.json` + `steps_*.json` (internal IDs, per `INTERNAL_FORMAT.md`). The input to the displayRep generator. Unchanged by this refactor. | `turns_bbr-20261113-1635.json`, `steps_bbr-20261113-1635.json` |
| (d) `displayRepresentation` | The NEW fully-resolved file (this spec). One file per game. A pure derived artifact: regenerable from (c) + the ID maps, so there is never a sync problem. | `display_bbr-20261113-1635.json` |

### Metadata — no separate (b) file (decided Nov 13, 2026)

There is **no separate metadata file** (no `originalMetadata.txt`).
The upload time already lives in the internalRep's `upload_date`
field, and Step 2 creates a Supabase table row per game which is the
natural single source of truth for metadata. A separate `.txt` file
would be a third copy of the same info and a sync risk. The files
stay self-describing via the internalRep's date fields.

The Supabase table row (Step 2) will hold, per game:

| column (suggested) | value |
|---|---|
| `game_key` | the dedup key, e.g. `bb3\|2026-11-13T16:35:00Z\|codegoon\|gatusso` (primary key) |
| `source` | `bb3` or `fumbbl` |
| `source_game_id` | game id in the source's own terms (`bbr-YYYYMMDD-HHMM` for BB3; replay id for FUMBBL) |
| `game_date` | when played, UTC ISO 8601 (the game start time for BB3) |
| `upload_date` | when imported, UTC ISO 8601 |
| `original_path` | storage object name/path for (a) the original file |
| `internal_path` | storage object name/path for (c) the internalRep (turns + steps) |
| `display_path` | storage object name/path for (d) the displayRep |

The table row is the catalog (for the dropdown); the three files are
the content. The row's `game_key` is the dedup key, so re-uploading
the same game is detected before any file is written.

## DisplayRep format — `display-rep-1.0.0`

One file per game (preferred for simplicity over a turns/steps
pair). It mirrors the internalRep structure so the mapping is
obvious, but every displayable value is a resolved string.

### Top-level fields

| field | meaning |
|---|---|
| `format` | `display-rep-1.0.0` |
| `source` | `bb3` or `fumbbl` |
| `source_game_id` | as in the internalRep |
| `game_date` / `upload_date` / `game_key` / `source_date` / `source_timezone` | copied from the internalRep (see `INTERNAL_FORMAT.md`) |
| `ruleset` | e.g. `BB2020` (BB3) / `BB2025` (FUMBBL) |
| `competition` | BB3: `pregame.competition` (e.g. `Open Ladder`). FUMBBL: the match division + scheduler (e.g. `Competitive (Blackbox)`, `League`). |
| `teams` | `{home, away}`: name, coach, race (display name), teamId. `race_id` is NOT carried (the race name is the display value). |
| `finalScore` / `blocksByActiveTeam` | as in the internalRep |
| `roster` | resolved roster (see below), keyed by pid |
| `turns` | turn-end snapshots, as in the internalRep, with each snap player enriched with `name` + `position` |
| `steps` | per-step deltas, as in the internalRep, with each player entry enriched with `name` + `position` |
| `events` | as in the internalRep, with `attackerName` / `defenderName` added |
| `derived_from` | `{internalRep_importer_version, id_map_version, generator}` — provenance, no timestamps |
| `notes` | derivation notes, including any fallbacks/unresolved ids |

### Roster — resolved fields

For each player (keyed by pid), the displayRep carries the
internalRep roster fields plus the resolved display fields:

| field | meaning |
|---|---|
| `name` | player display name. If the internalRep name is null (one case: a star player whose name the `.bbr` did not supply), it falls back to the position name and is recorded in `notes`. |
| `position` | position display name (e.g. `Wood Elf Wardancer`, `Thrall Lineman`). |
| `position_id` | the numeric id, kept as provenance (not needed for display). |
| `position_confidence` | `high` (direct map/lookup hit), `fingerprint` (BB3 fingerprint fallback), `unknown` (unresolved). |
| `chess_piece` | the chess-piece label (`Pawn`, `Rook`, `Knight`, `Bishop`, `Queen`, `King`, `Star`, `Circle`). BB3: from the internalRep (assigned by the importer via `chess_rule.py`). FUMBBL: assigned by the generator with the same shared rule, per team (see below). |
| `ma` / `st` / `ag` / `pa` / `av` | stats as in the internalRep. |
| `skills` | all skill display names (innate + acquired). The frontend reads this; it never needs a skill table. |
| `skills_innate_names` / `skills_acquired_names` | the innate and acquired skill display names separately (the roster popup shows innate plain and acquired bold). BB3 only; for FUMBBL these are the current skills / empty (see "FUMBBL notes"). |
| `skills_innate` / `skills_acquired` | the original numeric code arrays, kept as provenance (BB3 only). Not needed for display. |

The key fix this enables: the roster popup's skill-name table no
longer needs to exist in the frontend. The BB3 skill codes are
resolved server-side via the full ID map, so an acquired code 1
renders as `Strip Ball` (not `Skill 1`) and every code in the nine
games resolves. Zero `Skill N` strings appear in any displayRep.

### Turns / steps / events — what is resolved

- **Turn snapshots** (`turns[].snap.players`): each player entry
  gains `name` and `position` (from the roster). The `x`, `y`,
  `state`, `team`, `nr` values are unchanged from the internalRep.
- **Steps** (`steps[].players` deltas and `steps[].full` snapshots):
  each player entry gains `name` and `position`. A null entry (a
  player removal) is left as-is. Log lines are already display text
  and are unchanged.
- **Events**: `attackerName` / `defenderName` are added (resolved
  from the roster by pid). The `attacker` / `defender` pid fields
  are kept as provenance.

No board, score, turn, step, or event data is changed — only names
are resolved. Verification replays the displayRep's turns/steps
against the internalRep's and requires identical boards.

## How the generator resolves each source

### BB3

- Skill codes (`skills_innate`, `skills_acquired`) are resolved via
  the thin BB3 ID map (`bb3_id_map_1.4.0.json`, version
  `bb3-idmap-1.4.0.1` with Vampire). A code not in the map renders
  as `Skill <code>` and is recorded in `notes` (none in the nine
  games — every code resolves).
- Positions, races, and chess pieces are already resolved in the
  internalRep and are carried through unchanged.
- The map wins over the fingerprint fallback (the fingerprint is
  only used by the importer when the map lacks an entry, so it does
  not appear in the displayRep generation path except as the
  `position_confidence` value already recorded in the internalRep).

### FUMBBL

- The internalRep has **no roster**, so the generator rebuilds it
  from the saved replay stream (`replay_stream_<id>.jsonl`) using
  `parse_replay.rosters()` plus the raw player data. Skills come
  from the stream's `skillArray`, which already holds display
  names (e.g. `Frenzy`, `Horns`), so no skill-code resolution is
  needed.
- Positions are resolved via a small embedded lookup
  (`FUMBBL_POSITION_NAMES` in `build_display_rep.py`, 20 entries)
  derived from the BB2020 rulebook rosters cross-checked against
  each position's statline and skills in the game data. It covers
  the four races in scope (Wood Elf, Orc, Chaos Dwarf, Black Orc).
  It is a display lookup only and does not replace a full FUMBBL
  ID map (agreed design, not yet built). A positionId not in the
  table falls back to the player's `playerType` (e.g. `Big Guy`)
  and is recorded in `notes` (none in the three games — all 20
  resolve).
- Chess pieces are assigned with the shared `chess_rule.py`, per
  team (collisions are within a team). FUMBBL skills are the
  player's **current** skills (starting + acquired), so a player
  who acquired a role-changing skill could in principle shift
  piece versus a starting-skills classification. In practice the
  assignments are sane (Thrower Queen, Wardancer Knight, Treeman
  King, Lineman Pawn, etc.). One observed consequence: a Black Orc
  reads as Knight in one game (it has Brawler in its skill list)
  and as Rook in another (no Brawler listed) — the shared rule
  applied as-is to the skills present in each game's data.

## FUMBBL position lookup (embedded, 20 entries)

Derived from the BB2020 rulebook + the statlines/skills in the
three games' data. Wood Elf: 66176 Lineman, 66177 Thrower, 66178
Catcher, 66179 Wardancer, 66180 Treeman. Orc: 66215 Lineman, 66216
Thrower, 66217 Blitzer, 66218 Black Orc, 66219 Goblin, 66220 Troll.
Chaos Dwarf: 66316 Hobgoblin, 66317 Blocker, 66318 Bull Centaur,
66319 Minotaur, 66320 Hobgoblin, 66321 Flamesmith. Black Orc: 66327
Goblin, 66328 Black Orc, 66329 Troll. (Orc 66218 reads MA5, one
above the standard Black Orc MA4 — treated as a +MA characteristic;
the ST4/AV10/Block profile is unambiguously a Black Orc.)

## Known gaps / fallbacks (in the nine games)

- **Bugman (bbr-20261110-0728, home pid 12).** The internalRep
  roster has `name: null` for this player (the `.bbr` did not
  supply a name; position `Bugman`, a star player). The displayRep
  falls back to the position name (`Bugman`) so the display never
  has a null name. Recorded in that file's `notes`. The position
  and skills (Tackle, Wrestle, Loner (5+), Thick Skull) resolve
  normally.
- **FUMBBL skills are not split innate/acquired.** The stream's
  `skillArray` is the player's current skills; the displayRep lists
  them all in `skills` (and in `skills_innate_names`) with
  `skills_acquired_names` empty. The popup cannot bold "acquired"
  FUMBBL skills until a starting-skills source exists.
- **Vargheist position_id still unknown.** The fingerprint is
  defined (MA5 ST5 AG4+ PA− AV10+ + Bloodlust/Claws/Frenzy/
  Regeneration) but no position_id is assigned; it was not in any
  of the nine games' rosters. When a future game contains one, the
  fingerprint will recognise it and the id can be added to the map.
- **No unresolved skill codes or positions in the nine games.**
  Every BB3 skill code resolves via the ID map (zero `Skill N`),
  every FUMBBL positionId resolves via the embedded lookup, and
  every race is named (zero `Unknown`).

## Verification (Nov 13, 2026)

For each of the nine games, the displayRep was checked against its
internalRep: identical turn/step/event counts, identical finalScore,
game_date and game_key, identical turn-end boards (every player's
x/y/state/team/nr), identical step commands and player positions,
zero `Skill N` strings, zero `Unknown` race/position, every roster
player has a name, position and chess piece, and every skill is a
display name (no numeric codes in `skills`). All nine pass.
Determinism was verified by running the generator twice and
comparing MD5 hashes — byte-identical across runs.

Files: `build_display_rep.py` (generator, version 1.0.0),
`data/display_<game_id>.json` (nine files, one per game).
