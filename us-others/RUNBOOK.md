# Influencer watch 2 (new list) — cron runbook

One daily run: **05:00** Europe/London (`daily`). State lives in
`~/workspace/influencer-watches-2/`: `channels.json`, `seen.json`,
`pending.json`, `calls.json`, `last_check.json`, `captions/`, `digests/`,
`video_summaries.json`.

## Browse escalation (applied 2026-09-30)

`bin/scan.py` `channel_recent_videos` no longer retries a single failing
lookup. For each channel it tries, in order:

1. Videos tab, WEB client (the original path)
2. Uploads playlist (`UU` + channel id), WEB client — bypasses the tab renderer
3. Videos tab, ANDROID client
4. Uploads playlist, ANDROID client

It returns `(videos, path_used, path_errors)`. The stdout summary now
includes `browse_failures` (`{channel_id: <all-paths error>}`) for channels
where every path failed — carry that into the digest Notes line along with
which fallback path recovered any channel (`NOTE ... recovered via ...`
on stderr). A channel dark behind "all browse paths failed" for several
days despite all four paths needs per-channel investigation (ID
re-verification), not more retries.

## Procedure (every run)

1. Run `python3 ~/workspace/influencer-watches-2/bin/scan.py`. It detects new
   videos since the last check, fetches English captions, and queues them in
   `pending.json`. Read `pending.json` — those are the videos to analyze.
   First run: pass `--max-lookback-hours 24` so the window covers the prior
   day.
2. For each pending video, read its caption file and extract stock calls.
   Videos with `caption_ok: false` and an `unplayable_reason` (e.g.
   members-only) cannot be analyzed yet — move them to `seen.json` with
   `alerted: false` and a `skipped` field holding the reason. Do NOT treat
   them as processed: a members-only video can go public later, so later
   scans retry `skipped` entries automatically. Only videos that were fully
   analyzed — captions read, calls extracted or confirmed none — are never
   retried.
   Caption-failure rule (public videos only, NOT members-only): when a
   public video's captions can't be fetched, it is retried on the next
   cycle — `scan.py` counts attempts in `caption_retries` on the pending
   entry. Preserve `caption_retries` when moving the entry to `seen.json`.
   After the 3rd failed attempt the entry carries `caption_failed: true`:
   move it to `seen.json` with the failure flag and NO `skipped` marker
   (never retried again), and include a one-line failure notice in the
   daily summary: `⚠️ Couldn't read captions after 3 tries — <Creator>
   "<title>" (<published>). Not analyzed.` It reports exactly once per
   video.
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
   `digests/YYYY-MM-DD-HHMM-daily.md`.
5. ALWAYS send the daily summary to this chat (format below) — this is the
   heartbeat; it sends even on quiet days. Before composing it, check
   `seen.json` for entries with `caption_failed: true` and no
   `failure_reported` flag: for each, add a one-line notice to the summary —
   `⚠️ Couldn't read captions after 3 tries — <Creator> "<title>"
   (<published>). Not analyzed.` — then set `failure_reported: true` on the
   entry so it reports exactly once.

## Caption retry jobs (09:00 / 13:00 / 17:00 UK, silent)

Public videos whose captions fail at the 05:00 scan are retried the same day
by `bin/retry_captions.py` — one attempt per run, at most 3 tries total
(shared `caption_retries` counter with the 05:00 scan). Recovered caption
files are picked up by the next 05:00 scan via its caption-file check and
analyzed in the normal flow; anything recovered folds into the 05:00 digest
— these jobs never alert on their own. After the 3rd failed attempt the
entry is marked `caption_failed` and reported once in the 05:00 digest
(step 5). Members-only videos are never touched by these jobs; the 05:00
scan retries those indefinitely.

## Alert format (phone-readable, no headers)

Influencer digest — Sat 26 Sep
<Creator>: 2 videos. AAPL — BUY @ $310 (adding) — <short 1–2 line video
summary>; TSLA — SELL (trimming into strength) — <short video summary>.
No flips.
<Creator>: no videos today.

Keep each video to a few lines. Link nothing. If a run fails partway, leave
unanalyzed videos in `pending.json` for the next run (never alert twice for
the same video_id — a video already in `seen.json` without a `skipped`
marker was processed before and must not be re-alerted; this includes videos
marked `caption_failed`).

## Space refresh (once daily, only when something changed)

If this run added any new calls to `calls.json`, or a creator was added to /
removed from `channels.json` since the last refresh: (a) for each newly
analyzed video with readable captions, write a detailed video summary
(~150–250 words: title, creator, published date, what the video covers, key
arguments beyond the single call, other tickers discussed, calls made with
stance + stated price; never invent facts, note garbled caption spots) and
append it to `~/workspace/influencer-watches-2/video_summaries.json` keyed
by `video_id`; (b) update the `influencer-calls-new-list` web artifact via
`artifact.edit` so the space reflects the new calls — new ticker rows,
updated stance chips and per-stance averages, per-influencer call lists, and
the detailed video summaries for the new videos. Skip the edit entirely when
nothing changed.

## Operating rules

- Observations go to `~/memory/YYYY-MM-DD.md`, never `MEMORY.md`.
- This thread is alerts-only: no swing/medium-term discussion here.
- Keep this list fully separate from the original influencer watch
  (`~/workspace/influencer-watches/`): separate `channels.json`,
  `calls.json`, `seen.json`, separate space, separate thread.
