# India influencer watch — cron runbook

One daily run at **07:00 IST** (`influencer-watch-india-0700-ist`), before the
Indian market opens (09:15 IST). There is no after-hours session to cover, so a
single pre-market scan is enough. All times Asia/Kolkata. State lives in
`~/workspace/influencer-watches-india/`: `channels.json`, `seen.json`,
`pending.json`, `calls.json`, `last_check.json`, `captions/`, `digests/`,
`video_summaries.json`.

Channels (verified 2026-10-03):
- Rahul Jain — `UC2MU9phoTYy5sigZCkrvwiw` (@torahulj, English; SEBI-registered
  research analyst — his videos carry a "not advice, for knowledge only"
  disclaimer; treat calls as research, never as recommendations)
- Rahul Jain - Hindi — `UCc6CmEbFkEIHJKzZYCshKEQ` (@rahuljainhindi)
- Akshat Shrivastava — `UCqW8jxh4tH1Z1sWPbkGWL4g` (@AkshatZayn, Wisdom Hatch)

## Procedure (every run)

1. Run `python3 ~/workspace/influencer-watches-india/bin/scan.py`. It detects
   new videos since the last check, fetches **English** captions, and queues
   them in `pending.json`. Read `pending.json` — those are the videos to
   analyze. The scan covers two sources per channel: the /videos tab AND the
   "Collaborations" shelf (MWEB youtubei client). Collab videos are queued even
   when hosted on other channels, regardless of age; entries carry
   `collab: true` and a `collab_byline` — attribute the call to the watched
   creator whose shelf it came from (`channel`), and note the collaboration in
   the alert.
2. Hindi-channel rule: only English captions are fetched (manual preferred,
   else auto). A Hindi-channel video with no English captions is skipped —
   move it to `seen.json` with `alerted: false` and a `skipped` field holding
   the reason ("no English captions"). Do NOT treat it as processed for retry
   purposes in the members-only sense; captionless Hindi videos are not
   retried (unlike members-only videos, which can go public later).
   Caption-failure rule (public videos with English captions expected):
   `scan.py` counts attempts in `caption_retries`; after the 3rd failed attempt
   the entry carries `caption_failed: true` — move to `seen.json` with the
   failure flag and include a one-line failure notice in that run's message:
   `⚠️ Couldn't read captions after 3 tries — <Creator> "<title>"
   (<published>). Not analyzed.` Reports exactly once per video.
   One call-event per (video, ticker). Direction: BUY / CONDITIONAL_BUY /
   HOLD / SELL / AVOID / UNCERTAIN. Capture: ticker (NSE/BSE symbol as stated,
   e.g. RELIANCE, HDFCBANK), his stated price in ₹ (if any), one-line thesis,
   and one-line base / bull / bear case each where he gives one. Never invent
   a call or a scenario — a passing mention is UNCERTAIN. Auto-captions garble
   names and numbers; sanity-check tickers against the company discussed.
3. Load `calls.json` (all previous calls). Flag **stance flips**: same creator,
   same ticker, different direction from their most recent previous call.
4. Move each analyzed video from `pending.json` to `seen.json`
   (`{video_id: {channel, title, published_at, analyzed_at, alerted: bool}}`)
   and append its calls to `calls.json`. Save the digest as
   `digests/YYYY-MM-DD-HHMM-ist.md`.
5. Decide what reaches the user per the slot rules below.

## Slot rules (single daily run)

- If any analyzed video has ≥1 explicit call (BUY, CONDITIONAL_BUY, HOLD,
  SELL, AVOID, or a stance flip), send ONE batched alert for the run (see
  format). HOLDs always alert. Videos with only UNCERTAIN mentions, or with no
  captions and no calls, stay silent — except a caption-failure notice (step
  2), which always goes out.
- Quiet days: always send the heartbeat — one line per creator
  ("no new videos" / "1 video, no explicit calls"). The user asked to keep the
  heartbeat.
- After the alert/heartbeat, refresh the `influencer-calls-india` space when
  new calls landed or the creator roster changed.

## Alert format (phone-readable, no headers)

🎬 New calls — <Creator> "<video title>" (<published, e.g. "3h ago")
📝 <3–5 line summary: what the video covers, the creator's key arguments and
takeaways — a real summary, not just the calls>
📊 Recommendations:
- TICKER — BUY @ ₹X (stated) — one-line thesis; bull: … / bear: …
- 🔄 FLIP: <Creator> was AVOID on TICKER, now BUY — one line on why

If an analyzed video has no explicit calls, keep its 📝 summary and add one
line: "No explicit calls in this one." Never skip the summary — the user wants
every video summarized, recommendations or not.

Heartbeat format:

India digest — Sat 4 Oct
Rahul Jain: 1 video. <ticker calls or "no explicit calls">.
Rahul Jain - Hindi: no new videos (1 skipped — no English captions).
Akshat Shrivastava: no new videos.

Keep each video to a few lines. Link nothing. Never promise the user a
follow-up; if a run fails partway, leave unanalyzed videos in `pending.json`
for the next run (never alert twice for the same video_id — a video already
in `seen.json` without a `skipped` marker was processed before and must not
be re-alerted; this includes videos marked `caption_failed`).

## Operating rules

- Observations go to `~/memory/YYYY-MM-DD.md`, never `MEMORY.md`.
- Alerts go to the India-watch chat thread (the chat this watch was created
  in), not the other influencer alert chats.
- This thread is alerts-only: no swing/medium-term discussion here.
