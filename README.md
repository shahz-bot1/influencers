# Influencer stock-call watches

Data backing the three YouTube influencer stock-call monitoring workflows.
Refreshed from the watch crons; this repo is the backup / versioned copy.

- `us-selected/` — original 7-creator watch (Couch Investor, Parkevtate Vosianc CFA,
  Mark Roussin CPA, The German Value Investor, Joseph Carlson After Hours,
  Business With Brian, MarketBeat). Five scans daily (07:00/11:00/15:00/19:00/23:00 UK).
- `us-others/` — second 10-creator list (Everything Money, Sven Carlin, Brian Feroldi,
  Daniel Pronk, Tom Nash, Joseph Hogue, UNRIVALEDD Investing, Asymmetric Investing,
  Jose Najarro, Patient Investor). One scan daily at 05:00 UK.
- `indian-selected/` — India watch (Rahul Jain English @torahulj, Rahul Jain Hindi,
  Akshat Shrivastava @AkshatZayn). One scan daily at 07:00 IST, pre-market.

Each folder holds:

- `calls.json` — the calls ledger (one event per video+ticker: direction, price, thesis)
- `video_summaries.json` — per-video summaries keyed by video_id
- `channels.json` — watched channels (ids, handles)
- `seen.json` — processed-video dedup state
- `RUNBOOK.md` — the scan/analysis procedure and alert format
- `digests/` — per-run digest notes

`captions/` and scratch state (`pending.json`, `last_check.json`) are intentionally
omitted — captions are re-fetchable from YouTube.
