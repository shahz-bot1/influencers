# Influencer watch — cron runbook

Five daily runs (UK): **07:00** `overnight` · **11:00/15:00/19:00** `intraday` ·
**23:00** `daily`. Each run is told its slot in the cron body. All times
Europe/London. State lives in `~/workspace/influencer-watches/`:
`channels.json`, `seen.json`, `pending.json`, `calls.json`,
`last_check.json`, `captions/`, `digests/`.

## Procedure (every slot)

1. Run `python3 ~/workspace/influencer-watches/bin/scan.py`. It detects new
   videos since the last check, fetches English captions, and queues them in
   `pending.json`. Read `pending.json` — those are the videos to analyze.
   The scan covers two sources per channel: the /videos tab AND the
   "Collaborations" shelf on the channel home page (pulled via the MWEB
   youtubei client — the only client that renders the shelf; the desktop
   client does not). Collab videos are queued even when hosted on other
   channels' pages, regardless of age; entries carry `collab: true` and a
   `collab_byline` (e.g. "The Motley Fool and Couch Investor") — attribute
   the call to the watched creator whose shelf it came from (`channel`),
   and note the collaboration in the alert.
2. For each pending video, read its caption file and extract stock calls.
   Videos with `caption_ok: false` and an `unplayable_reason` (e.g.
   members-only) cannot be analyzed yet — move them to `seen.json` with
   `alerted: false` and a `skipped` field holding the reason. Do NOT treat
   them as processed: a members-only video can go public later (members
   just get it earlier), so later scans retry `skipped` entries automatically.
   Only videos that were fully analyzed — captions read, calls extracted or
   confirmed none — are never retried.
   Caption-failure rule (public videos only, NOT members-only): when a
   public video's captions can't be fetched, it is retried on the next
   cycle — `scan.py` counts attempts in `caption_retries` on the pending
   entry. Preserve `caption_retries` when moving the entry to `seen.json`.
   After the 3rd failed attempt the entry carries `caption_failed: true`:
   move it to `seen.json` with the failure flag and NO `skipped` marker
   (never retried again), and include a one-line failure notice in that
   run's user-facing message — even if the run would otherwise stay silent:
   `⚠️ Couldn't read captions after 3 tries — <Creator> "<title>"
   (<published>). Not analyzed.` It reports exactly once per video.
   One call-event per (video, ticker). Direction: BUY / CONDITIONAL_BUY /
   HOLD / SELL / AVOID / UNCERTAIN. Capture: ticker, his stated price (if any),
   one-line thesis, and one-line base / bull / bear case each where he gives
   one. Never invent a call or a scenario — a passing mention is UNCERTAIN.
   Auto-captions garble names and numbers; sanity-check tickers against the
   company discussed.
3. Load `calls.json` (all previous calls). Flag **stance flips**: same creator,
   same ticker, different direction from their most recent previous call.
4. Move each analyzed video from `pending.json` to `seen.json`
   (`{video_id: {channel, title, published_at, analyzed_at, alerted: bool}}`)
   and append its calls to `calls.json`. Save the digest as
   `digests/YYYY-MM-DD-HHMM-<slot>.md`.
5. Decide what reaches the user per the slot rules below. If nothing should
   be sent, call `muse.nothing_to_do()` and end — no user-facing message.

## Slot rules

- **intraday (11:00, 15:00, 19:00):** if any analyzed video has ≥1 explicit
  call (BUY, CONDITIONAL_BUY, HOLD, SELL, AVOID, or a stance flip), send ONE
  batched alert for the run (see format). HOLDs always alert — the user wants
  them with a short video summary. Videos with only UNCERTAIN mentions, or
  with no captions and no calls, stay silent — except a caption-failure
  notice (step 2), which always goes out. Nothing new and no failures →
  `muse.nothing_to_do()`.
- **overnight (07:00):** same bar, but covering the 23:00→07:00 window. If
  calls were found overnight, send the morning summary. Nothing →
  `muse.nothing_to_do()`.
- **daily (23:00):** always send the daily aggregate, even on quiet days
  (this is the heartbeat). Group by influencer: videos uploaded that day,
  every recommendation with direction and his price level, stance flips,
  tickers covered, plus a short (1–2 line) summary of each video. If a
  creator posted nothing, say so in one line.

## Alert format (phone-readable, no headers)

🎬 New calls — <Creator> "<video title>" (<published, e.g. "3h ago")
📝 <1–2 line summary of what the video covers / what he's updating>
- TICKER — BUY @ $X (stated) — one-line thesis; bull: … / bear: …
- 🔄 FLIP: <Creator> was AVOID on TICKER, now BUY — one line on why

Daily aggregate format:

Influencer digest — Wed 24 Sep
<Créator>: 2 videos. AAPL — BUY @ $310 (adding) — <short video summary>; TSLA — SELL (trimming into strength) — <short video summary>. No flips.
<Creator>: no videos today.

Keep each video to a few lines. Link nothing. Never promise the user a
follow-up; if a run fails partway, leave unanalyzed videos in `pending.json`
for the next run (never alert twice for the same video_id — a video already
in `seen.json` without a `skipped` marker was processed before and must not
be re-alerted; this includes videos marked `caption_failed`).

## Operating rules

- Observations go to `~/memory/YYYY-MM-DD.md`, never `MEMORY.md`.
- This thread is alerts-only: no swing/medium-term discussion here.
