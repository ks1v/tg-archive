# TODO — Deferred Items

Items that require external access, credentials, or decisions beyond the planning stage.

---

## Notion Setup

- [ ] Create page `TG Archive` in the target Notion workspace
- [ ] Create child database **TG Archive Channels** with columns as specified in PLAN.md
- [ ] Create child database **TG Archive Logs** with columns as specified in PLAN.md
- [ ] Set up rollup columns: `total runs` (count of Logs), `total posts` (sum of `posts` in Logs)
- [ ] Add a card/bookmark to this repo (`https://github.com/ks1v/tg-archive`) on the TG Archive page
- [ ] Populate **TG Archive Channels** with first record:
  - name: `прошмандовцы`
  - url: `https://t.me/blacklistrussianemigration`
  - is_on: checked
- [ ] Share the Notion integration with the TG Archive page (required for API access)
- [ ] Verify rollup aggregation works correctly after first run

---

## Telegram Data Model — Research & Improvements

- [ ] Review the full Telethon `Message` object docs and confirm all fields mapped in `post` table
- [ ] Confirm how comments/replies are accessed for channels without a linked discussion group
  - Telethon: `GetRepliesRequest` only works for channels with a linked supergroup
  - Decide: skip comments silently or flag in run log?
- [ ] Investigate `reactions` structure in Telethon (changed in TG API layer 143+)
- [ ] Check if `post_author` (signature) is available for anonymous admins vs named contributors
- [ ] Decide whether to archive **forwarded** posts from other channels (currently: yes, with `forward_from_*` fields)
- [ ] Consider adding `poll` media type for Telegram polls (currently not in schema)
- [ ] Consider adding `geo` field for location messages

---

## Local Database Selection

Full analysis for the record (decision in PLAN.md is SQLite):

| DB         | Pros                                          | Cons                                          | Verdict         |
|------------|-----------------------------------------------|-----------------------------------------------|-----------------|
| SQLite     | Zero-config, file-based, WAL concurrent reads, stdlib | Single writer, no full-text search by default | **Chosen**      |
| DuckDB     | Columnar, fast analytics, Parquet export       | Not great for frequent small inserts, newer   | Consider for analytics layer |
| PostgreSQL | Full-featured, FTS, concurrent writes, mature  | Requires server, overkill for local cron      | If multi-user needed |

- [ ] Evaluate whether FTS (full-text search over `post.text`) is needed — if yes, add SQLite FTS5 virtual table or switch to PostgreSQL
- [ ] Decide on DB location: local file vs shared volume vs cloud (S3 + Litestream for SQLite replication)
- [ ] Evaluate Litestream for continuous SQLite replication to S3 as a backup strategy

---

## Media Storage

- [ ] Confirm available disk space and set media size budget
- [ ] Decide on media deduplication strategy (same file from two channels — symlink? hash check?)
- [ ] Implement periodic reconcile job: scan `media/` tree and flag files not in DB (and vice versa)
- [ ] Decide on video size limits — full video or just thumbnail?
- [ ] Consider using `file_hash` (SHA256) column in `media` table for deduplication

---

## Credentials & Config

- [ ] Create a Telegram API app at https://my.telegram.org to get `TG_API_ID` and `TG_API_HASH`
- [ ] Create a Notion integration at https://www.notion.so/my-integrations
- [ ] Set up `.env` from `.env.example`
- [ ] Decide on session management: single shared session file or per-user isolation
- [ ] Create a dedicated Telegram channel/group for run notifications (`TG_NOTIFY_CHANNEL`)

---

## Operations & Deployment

- [ ] Choose deployment target: local cron, Docker container, cloud VM, GitHub Actions scheduled workflow
- [ ] Set up cron schedule (e.g. `0 3 * * *` — 3 AM daily)
- [ ] Define alerting: what happens if the script fails entirely (no Notion update, no TG notification)?
  - Consider a dead-man's-switch (healthcheck ping to uptime monitor)
- [ ] Set up log rotation for script stdout/stderr
- [ ] Define retention policy: how long to keep archived posts? Keep all forever?
- [ ] Decide whether to add a simple read-only web UI (e.g. Datasette over the SQLite DB) for browsing archives

---

## Implementation Tasks (post-plan)

- [ ] Implement all modules as defined in PLAN.md
- [ ] Write `init_notion_workspace()` to automate Notion table creation via API
- [ ] Add `--dry-run` CLI flag
- [ ] Add `--channel` CLI flag to run a single channel manually
- [ ] Add `--since` CLI flag to override last-run date for a channel
- [ ] Write basic tests for `db.py` (schema init, upsert idempotency, dedup checks)
- [ ] Add post edit detection: on re-run, if `post.edit_date` changed, update `post.text` and `post.entities`
