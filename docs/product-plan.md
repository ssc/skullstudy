# Blood Bowl Study Tool — Product Plan

Living doc. Started Nov 10, 2026. Phase 1 is the user's words, captured; the feedback
notes are Muse's, for the user to accept or reject.

## Vision

One place where any Blood Bowl game — digital or tabletop — becomes a shareable,
scrubbable, discussable object, and where real game moments become study problems.

## Phase 1 — The shareable scrubber (current phase)

**The product:** load a BB3 `.bbr` or a FUMBBL replay, scrub through it turn by turn
(or command by command), and share it as a persistent link that opens at the exact
turn/step being discussed. Plus an **embed mode**, so the scrubber can live inside
a message-board post.

**The growth plan — organic, not paid:** every shared link is both content and
distribution. A coach posts a link after a game; the opponent, teammates, and forum
readers click it; the link *is* the demo — no install, no account, just a board you
can scrub. The natural next step for a viewer is uploading their own game. Discussion
happens where it already happens (forums, Discord, Reddit); the site is the replay,
not a new forum.

**Why this can spread:**

- The unit of sharing is a *moment* ("look at turn 8"), not a whole game. Deep links
  make arguments precise, which is what people actually argue about.
- The file sizes are tiny (hundreds of KB per game as JSON), so hosting is cheap and
  the viewer loads fast — no reason for a link to feel heavy.
- It starts from what already works: two verified importers and a scrubber, not new R&D.
  Phase 1 is mostly packaging + hosting + sharing.

## Feedback on Phase 1 (Muse)

**What's strong:**

- The loop is genuinely viral-shaped: viewer → uploader conversion is one click from
  any shared link, provided "upload your game" is on the shared page itself, not
  buried on a homepage.
- Embed mode is the right call for where Blood Bowl discussion lives. The killer use
  case is a forum thread arguing about one play, with the scrubber embedded at that turn.

**What could break the loop:**

1. **FUMBBL upload friction is the weak link.** BB3 is easy — a local `.bbr` file,
   drag and drop. FUMBBL currently needs the `.jnlp` + one-game auth token dance.
   If the FUMBBL upload path isn't nearly as smooth as BB3's, the loop runs on BB3
   games only. Worth designing the FUMBBL upload flow first, before launch.
2. **"Embed" means different things per surface.** Many forums restrict iframes
   (allowlisted domains only); Reddit doesn't allow iframes in posts; Discord doesn't
   run interactive embeds at all — it unfurls a link card. The thing that works
   *everywhere* is the link with a rich preview: a board-state image at the linked
   turn, coaches, races, score, "open at turn 8". Get the unfurl right first;
   true interactive embed is a bonus where the platform allows it.
3. **Don't own the discussion.** Building comments on-site competes with the forums
   where the audience already is, and an empty comment section hurts. Own the replay;
   let discussion stay where it is. Revisit only if the embeds start pulling
   discussion toward the site on their own.
4. **Seeding still matters.** "Organic" doesn't mean "starts itself". The loop needs
   first posters — a few coaches, a tournament, a league — putting links in real
   discussions. Worth naming who posts first before launch.
5. **Public-by-default needs a decision.** A shared link makes a game findable;
   coach names are in the data. Decide: unlisted links vs public listings, and basic
   abuse handling (spam uploads, other people's games). Small now, but decide before
   the first public link, not after.

**Open questions:**

- Site name / brand — same as TableShare, or a new name? (TableShare is the in-person
  app; this site may want its own identity, or be the umbrella.)
- URL scheme — `/game/<id>`, `/game/<id>#turn-8` or `/game/<id>/turn/8`? Decide with
  the deep-link design.
- Accounts: required to upload, or anonymous upload with the link as the key?
- JSON API for other AI/chatbots — part of Phase 1 or later?

## Scope note (user, Nov 10, 2026)

Realistically this project doesn't have the market to be very large scale — so we
may never get to Phase 1.5 or 2. That means Phase 1 must stand alone as a complete,
satisfying product, not a stepping stone that only pays off if later phases happen.
Don't over-invest in infrastructure for phases that may never come; design the
internal representation so later phases are *possible* without rework, but build
only what Phase 1 needs. If 1.5/2/3 happen, they're bonus, not the plan.

## Appendix — Storage: when does hosting the internal format break? (Nov 10, 2026)

Short answer: it doesn't, at any plausible scale for this market. Storage is a
non-issue; the binding constraints, if any, will be elsewhere (see below).

**Measured per-game sizes (avg of the two parsed games):**

- Internal format, raw JSON: turns ~97 KB + steps ~521 KB = **~602 KB/game**
- Internal format, gzipped: **~31 KB/game** (JSON compresses to ~5% — turns 83 KB→3.9 KB,
  steps 480 KB→24.7 KB)
- Original source for reference: `.bbr` 603 KB; FUMBBL raw stream 3.5–4.1 MB
  (3.5 MB gzips to 168 KB — but there's no reason to store it; the internal
  representation replaces it)

**Cost at scale (gzipped internal format only):**

| Games stored | Size | R2 $0.015/GB-mo | B2 $0.00695/GB-mo | S3 $0.023/GB-mo |
|---:|---:|---:|---:|---:|
| 1,000 | 0.03 GB | $0.00 | $0.00 | $0.00 |
| 10,000 | 0.32 GB | $0.00 | $0.00 | $0.01 |
| 100,000 | 3.17 GB | $0.05 | $0.02 | $0.07 |
| 1,000,000 | 31.7 GB | $0.48 | $0.22 | $0.73 |

- R2 and B2 both give the first **10 GB free** — that's ~**316,000 gzipped games**
  before the storage bill is even $0.01.
- Even storing the original `.bbr` alongside (another ~603 KB/game, uncompressible
  since it's already Base64+zlib) only doubles the numbers: 1M games ≈ $0.96/mo on R2.
- Egress is also trivial: each view serves ~31 KB gzipped. 1M views/month = 31.7 GB
  → $0 on R2 (egress free), ~$2.85 on S3 at $0.09/GB.
- Database/metadata is a few hundred bytes per game (id, coaches, races, score,
  date, ruleset) — 100k games is tens of MB, inside any free tier.

**So where would it actually break?** Not on storage. The plausible ceilings are:
(1) the free tier of whatever hosts the app/DB (e.g. Supabase free 500 MB database,
Pro $25/mo) — but metadata is tiny; (2) abuse/spam uploads, which is a moderation
problem, not a storage one; (3) a genuinely viral moment driving compute (page
renders, API calls), not bytes stored. One million games would require essentially
every digital Blood Bowl game ever played — the market doesn't have that many games.

**Design consequences:** store gzipped internal format only (steps + turns); keep the
original upload only if there's a reason (re-import after a format change — but the
format is versioned and regenerable from source only while the source exists, so
keeping the `.bbr` for BB3 is cheap insurance at ~$0.50/mo per million games).
Turn snapshots are derivable from steps by replaying deltas, so storing both is a
convenience, not a necessity — if size ever mattered, store steps only.

**If self-hosted on EC2 instead (Nov 10, 2026):** yes, it's cheaper in dollars —
roughly half of Supabase Pro — but the savings are ~$7–13/mo in absolute terms,
and you pay for them in ops time (patching, Postgres upgrades, backup testing,
monitoring, incident response). The cheapest path overall is still serverless
(R2 + Pages ≈ $0–1/mo); EC2 is the cheapest *serverful* path, not the cheapest path.

The cheap EC2 path is **one box running everything** — do not add RDS, or the
savings vanish (EC2 + RDS db.t4g.micro ≈ $32.61/mo, more than Supabase Pro).

- **Box:** t4g.small (2 vCPU, 2 GB RAM, Graviton/arm64) $12.26/mo + 30 GB gp3
  $2.40/mo + public IPv4 $3.65/mo = **~$18.31/mo** on-demand us-east-1. t4g.micro
  ($6.13) totals ~$12.18/mo but 1 GB RAM is tight for Postgres + app. A 1-year
  reserved/savings plan cuts compute ~35% (t4g.small path ≈ $14/mo). Or skip raw
  EC2: **Lightsail $12/mo flat** (2 GB RAM, 60 GB SSD, 3 TB transfer, IP included)
  is the same AWS underneath with one predictable number — for this project it's
  the better pick.
- **Stack (Docker Compose on the box):** `postgres:16-alpine` + app API + Caddy
  (reverse proxy, auto HTTPS). Ubuntu 24.04 LTS.
- **Data storage:** game blobs as gzipped files on disk, `/data/games/<id>.json.gz`
  (31 KB/game) — Postgres holds only a `games` metadata table (~500 B/row: id,
  coaches, races, score, date, ruleset, storage path). Do NOT store blobs as JSONB
  in Postgres: they bloat the DB, slow backups, and make pg_dump huge. A 60 GB
  disk holds ~1.9M gzipped games; a 40 GB disk ~1.26M. EBS gp3 is $0.08/GB-mo
  (3,000 IOPS + 125 MB/s baseline included).
- **Postgres on the box:** data dir on the EBS volume; `shared_buffers` ~256–512 MB
  for a 2 GB box; single instance, single AZ — no HA, which is fine at this scale.
  What matters is restore, not uptime: nightly `pg_dump` → R2/S3 (pennies) +
  weekly EBS snapshots ($0.05/GB-mo; 30 GB ≈ $1.50/mo) + keep the original
  uploads so games are re-importable. Test the restore once.
- **Frontend:** static site served by Caddy on the same box, or Cloudflare Pages
  (free) with EC2 doing API + DB only — cleaner split, same cost.
- **Egress — the one place a viral spike could cost real money on EC2:** 100 GB/mo
  free account-wide (~3.2M views/mo), then $0.09/GB — a 3 TB spike over the
  allowance is ~$261. Mitigations: Cloudflare free plan in front (DNS + CDN
  caching of game JSON + DDoS protection; repeat views never touch origin), or
  Lightsail, whose $12 bundle already includes 3 TB transfer (~95M views/mo).
  Cloudflare Tunnel also removes the need for a public IPv4 ($3.65/mo) entirely.
- **Growth path:** when the box gets tight, move blobs to R2/S3 and Postgres to
  RDS or a bigger box. Nothing in the app changes, because storage is already
  "files by ID + Postgres rows" — the same split Supabase would use.

Rates checked Nov 10, 2026: t4g.micro $6.13/mo, t4g.small $12.26/mo (us-east-1,
Oct 2026); public IPv4 $3.65/mo; EBS gp3 $0.08/GB-mo; EBS snapshots $0.05/GB-mo;
RDS db.t4g.micro ~$12/mo + gp3 $0.115/GB-mo; Lightsail $5 (0.5 GB/20 GB/1 TB),
$7 (1 GB/40 GB/2 TB), $12 (2 GB/60 GB/3 TB) with IPv4; transfer overage $0.09/GB.

**If hosted on Supabase instead (Nov 10, 2026):** the math changes shape, not just size.
Supabase isn't cheaper per GB — it's a flat fee with huge included headroom, and one
backend (Postgres + Auth + Storage + Realtime) for the whole product family, including
TableShare (whose BaaS brief already recommends Supabase). Assumes game JSON blobs
live in Supabase Storage and only metadata (~500 B/game) lives in Postgres — storing
blobs as JSONB in the database would be the expensive mistake (500 MB free DB =
~16k games vs 1M games' metadata).

**Decision (user, Nov 10, 2026): default to Supabase for speed of execution;
EC2/Lightsail tabled (kept in the appendix as the cheaper alternative).**

- **Free ($0/mo):** 1 GB file storage → ~**31,500 games** stored; 5 GB egress/mo →
  ~**158,000 views/mo**; 500 MB DB → metadata for ~1M games. The binding limit is
  the 1 GB of file storage. Caveat: free projects **pause after 7 days of
  inactivity** — fine for prototyping, but a public site wants Pro so the first
  visitor doesn't wait on a resume.
- **Pro ($25/mo, + compute with a $10/mo credit that covers the smallest Micro
  instance, so ~$25/mo all-in at the smallest size):** 100 GB file storage →
  ~**3.16M games** before overage; 250 GB egress/mo → ~**7.9M views/mo**; 8 GB DB →
  metadata for ~16M games. The bill is flat $25/mo until ~3.2M games stored or
  ~7.9M views/mo — marginal cost per game is **$0** for a very long time.
- **Beyond Pro included:** storage overage $0.0213/GB (10M games = 317 GB → ~$4.62/mo
  extra); egress overage $0.09/GB (10M views/mo → ~$6.03/mo extra). Egress, not
  storage, is the first overage you'd ever hit — and only at ~8M views/mo.
- **Comparison at 1M games stored:** R2 $0.48/mo (storage only; hosting + DB + auth
  are separate pieces to assemble) vs Supabase Pro $25/mo flat (covers 3.16M games)
  vs Supabase Free $0 (covers ~31.5k games). Supabase costs more at small scale;
  what it buys is one backend instead of several, and a $0 marginal cost per game
  until millions.
- **Spend cap (default on):** Supabase blocks over-quota usage rather than charging
  — no surprise bill, but a surprise outage if a cap is hit (e.g. egress). Decide
  deliberately whether to leave it on.

Rates checked Nov 10, 2026: Free 500 MB DB / 1 GB storage / 5 GB egress / 50k MAU;
Pro $25/mo, 8 GB DB / 100 GB storage / 250 GB egress / 100k MAU; overages DB
$0.125/GB, storage $0.0213/GB, egress $0.09/GB, MAU $0.00325.

## Phase 1.5 — Manual entry (user, Nov 10, 2026)

Input a game by hand, in realtime, into the same format the importers produce.
Same schema as Phase 1; the writer is a person, so the format must accept partial,
in-progress games and the editor needs guardrails (legal positions, whose turn it is).
Also the fallback/correction path for Phase 3: camera output gets fixed by hand here.

## Phase 2 — Graded problems (user, Nov 10, 2026)

Extract a real game situation into a study problem: a board snapshot + prompt, with
three views — "enter your answer here", "view what happened in the real game", and
"view 2 solutions from these 2 expert coaches". Problems are a data type, not a view:
snapshot + prompt + solution lines with odds + grading, with authored, attributed,
multiple expert solutions per problem. Manual entry (Phase 1.5) is what lets experts
author problems and solutions by hand; replay imports are the shortcut for extracting
real positions. UGC library of problems is the retention/creation engine.

## Phase 3 — Automated entry via AI/camera (user, Nov 10, 2026)

Automate capture of in-person games (TableShare: pitch cameras, later AI) into the
same event log. Hardest technical lift, last for that reason — and it reuses the
Phase 1.5 editor as its correction path. See also TableShare docs
(`~/workspace/blood-bowl-vision/`, TableShare side chat).

## Later / unphased

- **Numerical personal analysis** — e.g. "you are based X% of the time".
