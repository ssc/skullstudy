# SkullStudy — Deployment Plan (Cloudflare Container)

**Status:** Plan only — nothing built, no accounts created, no money spent.
**Date:** November 12, 2026
**Decision so far:** the Cloudflare Container is the most straightforward conversion worker (user, Nov 12, 2026). Hosting is otherwise staged: Cloudflare Pages first, Supabase only when login / uploads / favorites need it.

## 1. What "done" looks like

A public website, `skullstudy` (domain TBD), where:

- Any visitor can open a game at a persistent, shareable URL and step through it turn by turn and move by move — no BB3/FUMBBL client, no login required to view.
- A signed-in user can upload a BB3 `.bbr` file or a FUMBBL `.jnlp` file and get a shareable link back, usually within ~30 seconds.
- A recent-uploads list shows new games. Deep links point at an exact game / turn / step, so a coach can send "look at turn 12 here" and the recipient lands on that board.
- The same internal game format serves both sources, so the viewer never cares whether a game came from BB3 or FUMBBL. That format is also the seed for the TableShare "table-stream" and, later, graded study problems.

Out of scope for this deployment (later phases): graded problems, personal stats history, annotations, manual realtime entry, in-person photo review. Those are built on top of this, not part of it.

## 2. Architecture

```
User's browser
  │
  ├─ Cloudflare Pages ── static site (the scrubber SPA)
  │     skullstudy.pages.dev → custom domain when ready
  │
  ├─ Cloudflare Worker ── HTTP front door for uploads
  │     POST /api/upload/bbr   (multipart .bbr)
  │     POST /api/upload/jnlp  (multipart .jnlp, parsed in memory)
  │     GET  /api/games/:id    (status + metadata)
  │     GET  /api/games        (recent uploads)
  │     └─ calls the Container, never does conversion itself
  │
  ├─ Cloudflare Container ── the conversion worker (Python, unchanged)
  │     Docker image: python:3.12-slim + bb3_import.py + parse/steps
  │     + pathfind.py + chess_rule.py + bb3_id_map_1.4.0.json
  │     + websocket-client 1.9.2 + lzstring 1.0.4 (FUMBBL only)
  │     Instance: `basic` (1/4 vCPU, 1 GiB RAM) — scale to zero when idle
  │     POST /convert/bbr   { game_id, original_path }
  │     POST /convert/jnlp  { game_id, replay_id, coach, auth_token }  ← token in memory only
  │     Returns: turns + steps JSON, or an error (never the token)
  │
  └─ Supabase (added when uploads go live, not before)
        Auth — Google + email, login only needed to upload / favorite
        Storage — two private buckets:
            `originals/`  .bbr files (~600 KB) and raw FUMBBL streams (~3.5–4.1 MB)
            `processed/`  turns_*.json + steps_*.json (~655 KB uncompressed, ~31 KB gzipped)
            Note: original .bbr files contain player IPs; the processed
            internal rep has them stripped. Originals stay private.
        Postgres — `games` table (see §5)
```

Why this split: Pages is $0 and unlimited static bandwidth. The Worker is the auth/validation layer (small, fast, no heavy CPU). The Container is the only place that runs the existing Python — no port to TypeScript, no 2s CPU cap (Supabase Edge Functions), no 128 MB memory ceiling (Python Workers). Supabase is the only place that holds user data, so it can be swapped later without touching the viewer or the converter.

## 3. The container, in detail

**Dockerfile (sketch):**
- Base: `python:3.12-slim` (matches the local venv, Python 3.12.3).
- Copy in: `bb3_import.py`, `parse_replay.py`, `steps_replay.py`, `fetch_replay.py`, `getfumbbl.py` (logic only — see token note), `pathfind.py`, `chess_rule.py`, `bb3_id_map_1.4.0.json`.
- `pip install websocket-client==1.9.2 lzstring==1.0.4` (the only non-stdlib deps; BB3 needs none).
- A thin HTTP shim (stdlib `http.server` or FastAPI — TBD at build time) exposing `/convert/bbr` and `/convert/jnlp`. It does no game logic itself; it shells out to / imports the existing modules.
- No durable disk: all outputs go straight to Supabase Storage. No secrets baked into the image; Supabase service-role key arrives as an environment variable / Worker secret.

**Why a container, not a Python Worker:** Python Workers (GA Sep 21, 2026) give 30s CPU on the $5 Paid plan, but cap memory at 128 MB per isolate and run Python via Pyodide — the 11.7 MB BB3 XML in a DOM tree plus the Pyodide runtime is a real OOM risk, and the FUMBBL `websocket-client` (raw sockets) is untested there. The Container runs the code truly unchanged, including sockets.

**Instance + cost:** `basic` (1/4 vCPU, 1 GiB). Workers Paid is $5/month and includes 375 vCPU-minutes + 25 GiB-hours of memory per month. Measured locally: BB3 import ~1.0s wall / 0.88s CPU; FUMBBL fetch ~14s wall (mostly I/O wait, which doesn't count as CPU) + ~0.15s parse. At 100 uploads/month that's ~2 vCPU-minutes — effectively $5 flat. Supabase Pro ($25/month) is a separate, later cost, only when the site goes public with logins.

## 4. Upload flows

### 4a. BB3 `.bbr` (self-contained file, no network, no credential)

1. User (signed in) picks a `.bbr` in the browser → POST to the Worker.
2. Worker validates (size, file type, auth), uploads the original to `originals/`, inserts a `games` row with `status = 'pending'`, and returns `{ game_id }` immediately.
3. Worker calls the Container `/convert/bbr` with `{ game_id, original_path }` (async — the user doesn't wait on this call).
4. Container downloads the `.bbr` from Storage, runs `bb3_import.py` (decode Base64 → zlib → 11.7 MB XML → turns + steps, with the built-in verification: replaying step deltas must reproduce all turn-end boards, 0 mismatches), **pushes `turns_*.json` + `steps_*.json` directly to Supabase Storage `processed/`**, and the row flips to `ready` (or `failed`, with an error message). Push mechanism (decided Nov 12, 2026): the Container does NOT hold a Supabase service-role key. The Worker mints short-lived, path-scoped signed upload URLs for the `processed/` paths and passes them to the Container with the job; the Container PUTs the files to those URLs. The status flip to `ready`/`failed` goes through the Worker (or a signed callback to it), so database credentials never leave the Worker. Alternative, if the project ever goes Cloudflare-only for blobs: the Container writes to R2 via a service binding (no credentials at all) and Supabase stores only the R2 URLs + metadata.
5. The browser polls `GET /api/games/:id` (or subscribes to the row) and shows the shareable link when `status = 'ready'`.

### 4b. FUMBBL `.jnlp` (live network fetch, credential involved)

1. User picks a `ffblive.jnlp` in the browser → POST **directly to the Worker** (multipart), **not** to a Storage bucket.
2. Worker parses the JNLP **in memory only**: extracts `-gameId` (replay id), `-coach`, and `-auth`. The JNLP file itself is **never stored, never logged**.
3. Worker inserts a `games` row (`status = 'pending'`) and calls the Container `/convert/jnlp` with `{ game_id, replay_id, coach, auth_token }`.
4. Container opens `ws://fumbbl.com:22223/command`, sends LZString-compressed binary frames (`clientRequestVersion` → `clientJoin` in replay mode with the token as password → `clientReplay`), streams the `serverReplay` batches (100 commands each; ~2,936–3,244 commands, ~14s, 3.5–4.1 MB raw JSONL), saves that raw stream to `originals/` as the source of truth, runs `parse_replay.py` + `steps_replay.py`, uploads turns + steps to `processed/`, flips the row to `ready` / `failed`, and **drops the token from memory**. The token is never written to disk, to the database, or to logs.
5. Same polling / link flow as BB3.

**Verified fact that makes this safe:** the saved FUMBBL stream contains only server-sent messages (`serverVersion`, `serverGameState`, `serverModelSync`, `serverReplay`) — 0 occurrences of auth/password/IP in the verified `replay_stream_1954282.jsonl`. The token is only ever client-sent in `clientJoin` and is never echoed back.

**Known failure modes (must be handled as normal errors, not bugs):**
- Token expired (JNLP downloaded hours/days earlier) → server answers with a password challenge or silence → `failed`, message: "Token expired — please re-download a fresh JNLP from the match page and try again."
- FUMBBL server unreachable from a datacenter IP (untested — the local scripts run from the user's home IP) → `failed`, retryable.
- Partial stream (idle timeout mid-replay) → resume via `clientReplay` with `replayToCommandNr` set to the last received command.

## 5. Data model

**`games` table (Supabase Postgres):**

| Column | Notes |
|---|---|
| `id` | uuid, primary key; also the public URL slug |
| `source` | `'bb3'` or `'fumbbl'` |
| `uploader_id` | Supabase Auth user id (nullable for the seeded demo games) |
| `status` | `'pending'` → `'processing'` → `'ready'` / `'failed'` |
| `created_at` | for the recent-uploads ordering |
| `home_coach`, `away_coach`, `home_race`, `away_race` | from the internal rep |
| `home_score`, `away_score` | |
| `played_at` | game date from the replay, if present |
| `original_path` | Storage path of the `.bbr` / raw stream (private bucket) |
| `turns_path`, `steps_path` | Storage paths of the processed files |
| `replay_version`, `id_map_version`, `id_map_source_hash` | provenance, from the importer (BB3) |
| `error` | failure message, only when `status = 'failed'` |
| `importer_version` | e.g. `bb3-import-1.5.0` — so a re-import can be told apart |

**Row-level security:** anyone can read rows with `status = 'ready'`; only the uploader can read their own `pending` / `processing` / `failed` rows. Originals bucket: private, service-role only. Processed bucket: public-read for `ready` games (or signed URLs — TBD).

**URL scheme (to lock before launch):**
- Game: `/g/<id>`
- Deep link: `/g/<id>#t=<turn>&s=<step>` (exact format TBD with the scrubber's existing share-link format)
- Recent uploads: `/` (homepage list) + `GET /api/games?status=ready&limit=20`

**Idempotency:** the Container dedupes on `game_id`, so a retried / redelivered job can't create a second game. The `games` row is the single source of truth for status — the Container never invents a second row.

## 6. Phased build order

| Phase | What | Exit criteria |
|---|---|---|
| 0. Accounts (user) | Cloudflare account + Workers Paid ($5/mo); Supabase project (Free for dev); custom domain, if wanted | Accounts exist; API tokens in hand (Muse never creates accounts — billing is the user's) |
| 1. Static demo | Current scrubber (5 baked-in games) deployed to Cloudflare Pages from Git; custom domain; deep links work | A stranger can open a link and step through a game |
| 2. Container, BB3 only | Dockerfile + Worker + `/convert/bbr`; upload → `pending` → `ready` end-to-end with the two known BB3 games; verification gate enforced (0 mismatches or `failed`) | Uploading either known `.bbr` reproduces the exact baked-in turns/steps byte-for-byte |
| 3. FUMBBL | `/convert/jnlp` in the same Container; token-in-memory-only; expired-token error path; datacenter-IP reachability test | Uploading a fresh JNLP for a known game reproduces its known turns/steps |
| 4. Supabase wiring | Auth (Google), `games` table + RLS, Storage buckets, recent-uploads list, `failed` surfacing in the UI | A second user (not the owner) can sign in, upload, and share a link |
| 5. Polish before public | OG meta tags (so links unfurl with score/coaches), upload size limits + abuse caps, error pages, a "how to find your replay file" help page for BB3 | Share a link in a chat and it looks right; a new user can find their `.bbr` without asking |

Deliberately **not** in this plan: graded problems, personal stats, annotations, manual entry, TableShare live capture, any charging/paywall. Those come after the upload loop is proven with real users.

## 7. Open decisions (user's)

1. **Domain name** — `skullstudy.com`? Something else? Needed before Phase 1's custom-domain step, not before the Pages deploy (`.pages.dev` works meanwhile). Domain points at the Cloudflare Pages project (Pages creates a proxied CNAME to `<project>.pages.dev`, flattened at the apex, with automatic SSL). Registrar choice (Nov 12, 2026): Cloudflare Registrar is the simplest for this project — at-cost, `.com` ~$11.17/yr after the Nov 1, 2026 Verisign increase, renewal at the same price, free WHOIS privacy + DNSSEC, but domains must use Cloudflare nameservers (fine here) and there's a 60-day transfer lock. Alternatively register anywhere (e.g. Porkbun, Namecheap) and point the nameservers at Cloudflare — same Pages setup, one extra step. Keeping DNS at the registrar and adding a manual CNAME works for a `www` subdomain but not cleanly for the apex, so it's not recommended.
2. **Supabase vs Cloudflare-only backend** — the plan uses Supabase for Auth/Postgres because Cloudflare has no built-in auth; if the user prefers staying 100% Cloudflare, the alternative is D1 + a third-party auth provider (e.g. Clerk, $25/mo) — more assembly, same $5 Container.
3. **Public from day one, or invite-only uploads?** — viewing is free either way; uploads could start as owner-only while the FUMBBL flow is proven.
4. **Keep the original `.bbr` / raw stream, or delete after conversion?** — keeping it is the provenance story (re-import after an importer fix); deleting it saves ~600 KB–4 MB per game. Recommendation: keep, it's cheap.
5. **FUMBBL outreach timing** — the demo-first plan (Nov 11) was to show FUMBBL a working demo and ask about a sanctioned import path. The JNLP flow works without it, but a sanctioned path is a moat no copy can reproduce.

## 8. Risks

- **FUMBBL reachability from Cloudflare IPs** — untested; could be blocked or rate-limited. Mitigation: test in Phase 3 with a fresh JNLP before promising the flow; fallback is the existing local scripts.
- **Token expiry** — a stale JNLP fails opaquely (silence / challenge). Mitigation: clear error message + "re-download" guidance, as in §4b.
- **Container cold start** — first upload after idle takes seconds to spin up. Acceptable for uploads (async flow hides it); not a factor for viewing (Pages serves that).
- **Memory in the Container** — `basic` has 1 GiB; the BB3 XML parse peaks well under that locally. If a future BB3 version ships much larger replays, bump to `standard-1` (4 GiB).
- **Copyability** — the viewer code is replicable by any chatbot; the defensible assets are correctness across all races, the accumulated game/problem library, the link graph, and the FUMBBL relationship (see the Nov 12 discussion in this chat). This plan invests in the first three by shipping the upload loop early.
- **Cost creep** — at the volumes in §3 the bill is $5 (Cloudflare) + $0–25 (Supabase) per month. The thing to watch is not storage but a spam/abuse spike in uploads; Phase 5's caps (per-user daily upload limit, file-size limit) are the guardrail.
