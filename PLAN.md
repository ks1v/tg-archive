# TG Archive — Service Plan

## Overview

A daily-run Python service that archives public Telegram channels into a local SQLite
database and syncs state with a Notion workspace. Media is stored as files on disk.
Run logs are written to Notion and notifications are sent to a Telegram report channel.

---

## Architecture

```
Notion (frontend)
  ├── TG Archive Channels DB  ←─── sync ──→ channel table (SQLite)
  └── TG Archive Logs DB      ←─── sync ──→ run table (SQLite)

Python script (cron, daily)
  ├── reads Channels from Notion
  ├── connects to Telegram via Telethon (MTProto)
  ├── archives posts → post table (SQLite)
  ├── archives media → file tree + media table (SQLite)
  ├── archives comments → comment table (SQLite)
  ├── writes run result → Notion Logs + channel status
  └── sends notification → Telegram report channel
```

---

## Tech Stack

| Component       | Choice                  | Rationale                                           |
|-----------------|-------------------------|-----------------------------------------------------|
| Telegram client | `telethon` (MTProto)    | Full API access to public channels; async support   |
| Notion client   | `notion-client` (SDK)   | Official Python SDK                                 |
| Local DB        | SQLite (WAL mode)       | Zero-config, sufficient for daily cron, 281TB limit |
| Scheduling      | system cron / `schedule`| Simple; no daemon overhead                          |
| Config          | `.env` + `python-dotenv`| Standard, secret-safe                              |
| Python version  | 3.11+                   | Good asyncio, match statements, tomllib             |

---

## Media Storage Decision: File Tree vs Blobs

### Blobs in DB
**Pros:** atomic with metadata, single backup, transactional integrity, no orphaned files
**Cons:** DB grows to GB/TB scale; slow queries (table scans touch blob pages); memory
pressure when streaming large files; SQLite not designed for large binary objects;
harder to serve via HTTP without loading into memory

### File Tree on Disk
**Pros:** OS-level access, fast DB queries (only paths stored), nginx/CDN can serve
directly, standard backup tools (rsync, rclone), easy manual inspection, selective
deletion, media can be processed independently of DB

**Cons:** sync drift risk (file deleted while DB record remains); separate backup
strategy; orphan management needed

### Decision: **File Tree**

Store files at `media/{channel_username}/{YYYY}/{MM}/{post_id}_{original_filename}`.
Store relative path in `media.file_path`. Run a periodic reconcile job to detect orphans.

---

## Notion Setup

### Parent page: `TG Archive`
- Contains a card/link to this repo
- Child database 1: **TG Archive Channels**
- Child database 2: **TG Archive Logs**

### TG Archive Channels columns
| Column      | Type           | Notes                                  |
|-------------|----------------|----------------------------------------|
| name        | Title          |                                        |
| url         | URL            | t.me link                              |
| is_on       | Checkbox       | Whether archival is active             |
| last status | Text           | Status of last run                     |
| last run    | Date           | Datetime of last run                   |
| total runs  | Rollup         | Count of related Log records           |
| total posts | Rollup         | Sum of `posts` field across Logs       |

### TG Archive Logs columns
| Column   | Type     | Notes                              |
|----------|----------|------------------------------------|
| name     | Title    | Auto: "{channel} {datetime}"       |
| channel  | Relation | → TG Archive Channels              |
| datetime | Date     | Run start datetime                 |
| status   | Select   | ok / error / partial               |
| posts    | Number   | Posts archived this run            |

---

## Database Schema

### Table: `channel`
```sql
CREATE TABLE channel (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    tg_id     INTEGER UNIQUE,          -- Telegram internal channel ID (stable across username changes)
    name      TEXT    NOT NULL,        -- current value from Notion
    url       TEXT    NOT NULL UNIQUE, -- t.me link
    username  TEXT,                    -- resolved @username
    is_on     INTEGER NOT NULL DEFAULT 1,  -- boolean from Notion checkbox
    load_dttm TEXT    NOT NULL         -- ISO8601, datetime of last sync with Notion
);
```

### Table: `run`
```sql
CREATE TABLE run (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id  INTEGER NOT NULL REFERENCES channel(id),
    run_dttm    TEXT    NOT NULL,      -- run start datetime (ISO8601)
    load_dttm   TEXT,                  -- run end datetime (ISO8601)
    cnt_post    INTEGER NOT NULL DEFAULT 0,
    cnt_media   INTEGER NOT NULL DEFAULT 0,
    cnt_comment INTEGER NOT NULL DEFAULT 0,
    status      TEXT    NOT NULL DEFAULT 'running',  -- running / ok / error / partial
    log         TEXT                   -- full indented tree log text
);
```

### Table: `post`
```sql
CREATE TABLE post (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id       INTEGER NOT NULL REFERENCES channel(id),
    post_id          INTEGER NOT NULL,          -- Telegram message ID
    url              TEXT,
    text             TEXT,
    author_name      TEXT,                       -- post signature if set
    author_id        INTEGER,                    -- Telegram user/channel ID of author
    views            INTEGER,
    forwards         INTEGER,
    reactions        TEXT,                       -- JSON: [{"emoji":"👍","count":5}, ...]
    entities         TEXT,                       -- JSON: text formatting ranges
    is_pinned        INTEGER NOT NULL DEFAULT 0,
    no_forwards      INTEGER NOT NULL DEFAULT 0, -- channel restricted forwarding/saving
    grouped_id       INTEGER,                    -- Telegram album/group ID (multiple media in one post)
    forward_from_channel_id INTEGER,
    forward_from_post_id    INTEGER,
    edit_date        TEXT,                       -- ISO8601 if edited
    post_dttm        TEXT    NOT NULL,           -- original post datetime
    load_dttm        TEXT    NOT NULL,
    UNIQUE(channel_id, post_id)
);
```

### Table: `media`
```sql
CREATE TABLE media (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    post_id     INTEGER REFERENCES post(id),     -- NULL if from comment
    comment_id  INTEGER REFERENCES comment(id),  -- NULL if from post
    media_type  TEXT    NOT NULL,  -- image / video / audio / sticker / animation / document / voice
    file_name   TEXT,
    file_type   TEXT,              -- extension: jpg, mp4, ogg, webp, ...
    file_path   TEXT,              -- relative path in media/ tree
    file_size   INTEGER,           -- bytes
    mime_type   TEXT,
    width       INTEGER,           -- for image/video
    height      INTEGER,           -- for image/video
    duration    INTEGER,           -- seconds, for audio/video/voice
    load_dttm   TEXT    NOT NULL,
    CHECK (post_id IS NOT NULL OR comment_id IS NOT NULL)
);
```

### Table: `comment`
```sql
CREATE TABLE comment (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    comment_id       INTEGER NOT NULL UNIQUE,  -- Telegram message ID in discussion group
    post_id          INTEGER NOT NULL REFERENCES post(id),
    url              TEXT,
    text             TEXT,
    author_name      TEXT,
    author_id        INTEGER,
    reply_to_tg_id   INTEGER,   -- Telegram message ID this comment replies to (threading)
    reactions        TEXT,      -- JSON
    entities         TEXT,      -- JSON
    comment_dttm     TEXT    NOT NULL,
    load_dttm        TEXT    NOT NULL
);
```

**Schema improvements over original spec:**
- `channel.tg_id` — Telegram internal ID; username can change, ID never does
- `post.views`, `post.forwards`, `post.reactions` — engagement metrics
- `post.entities` — structured text formatting (bold, links, hashtags, mentions)
- `post.is_pinned`, `post.edit_date` — fidelity fields
- `post.forward_from_*` — provenance for forwarded posts
- `run.cnt_comment`, `run.status = 'running'` — better run lifecycle
- `media.file_path` instead of blob; `mime_type`, `width`, `height`, `duration`
- `comment.reply_to_tg_id` — enables thread reconstruction
- `comment.reactions`, `comment.entities` — parity with posts

---

## Module Structure

```
tg-archive/
├── main.py               # Entry point; wires everything together
├── config.py             # Config dataclass + .env loading
├── models.py             # Dataclasses for inter-module data transfer
├── db.py                 # SQLite schema init + all CRUD functions
├── notion_client.py      # Notion API wrapper
├── telegram_client.py    # Telethon wrapper for channel/post/media access
├── archiver.py           # Core orchestration logic
├── notifier.py           # Telegram notification sender
├── logger_setup.py       # Tree-indent logger
├── .env.example          # Template for required env vars
├── requirements.txt
├── PLAN.md
└── TODO.md
```

---

## Function Signatures

### `config.py`
```python
def load_config() -> Config
```

### `logger_setup.py`
```python
def setup_logger(name: str, level: int = logging.DEBUG) -> logging.Logger

class IndentLogger:
    """Context manager that adds one level of tree indentation to log output."""
    def __enter__(self) -> "IndentLogger": ...
    def __exit__(self, *_) -> None: ...
    def info(self, msg: str) -> None: ...
    def debug(self, msg: str) -> None: ...
    def warning(self, msg: str) -> None: ...
    def error(self, msg: str, exc: Exception | None = None) -> None: ...
```

### `db.py`
```python
def init_db(db_path: str) -> sqlite3.Connection

# channel
def upsert_channel(conn: Connection, channel: ChannelRow) -> int
def get_active_channels(conn: Connection) -> list[ChannelRow]
def update_channel_tg_id(conn: Connection, channel_id: int, tg_id: int, username: str) -> None

# run
def insert_run(conn: Connection, channel_id: int, run_dttm: str) -> int
def update_run(conn: Connection, run_id: int, cnt_post: int, cnt_media: int, cnt_comment: int, log: str, status: str) -> None
def get_last_successful_run(conn: Connection, channel_id: int) -> RunRow | None

# post
def post_exists(conn: Connection, channel_id: int, tg_post_id: int) -> bool
def insert_post(conn: Connection, post: PostRow) -> int

# media
def insert_media(conn: Connection, media: MediaRow) -> int

# comment
def comment_exists(conn: Connection, tg_comment_id: int) -> bool
def insert_comment(conn: Connection, comment: CommentRow) -> int
```

### `notion_client.py`
```python
def get_channels(notion: Client, channels_db_id: str) -> list[ChannelRow]
def create_run_record(notion: Client, logs_db_id: str, channel_notion_id: str, run_dttm: datetime) -> str
def update_run_record(notion: Client, record_id: str, status: str, posts: int) -> None
def update_channel_last_run(notion: Client, channel_notion_id: str, status: str, last_run: datetime) -> None
def init_notion_workspace(notion: Client, parent_page_id: str, repo_url: str) -> tuple[str, str]
```

### `telegram_client.py`
```python
async def connect(config: Config) -> TelegramClient
async def disconnect(client: TelegramClient) -> None
async def resolve_channel(client: TelegramClient, url: str) -> ChannelInfo
async def iter_posts(client: TelegramClient, channel_username: str, since_date: datetime | None, limit: int | None = None) -> AsyncIterator[Message]
async def iter_comments(client: TelegramClient, channel_username: str, post_tg_id: int) -> AsyncIterator[Message]
# Note: comments exist only if channel has a linked discussion group (linked_chat_id in channelFull).
# Access via GetRepliesRequest on the discussion group, not the channel itself.
# Returns empty iterator (not error) if no linked group exists.
async def download_media_file(client: TelegramClient, message: Message, dest_dir: Path) -> list[MediaFileInfo]
```

### `notifier.py`
```python
async def send_run_summary(client: TelegramClient, notify_channel: str, channel_name: str, result: RunResult) -> None
async def send_error_alert(client: TelegramClient, notify_channel: str, channel_name: str, error: str) -> None
```

### `archiver.py`
```python
async def archive_all(config: Config, conn: Connection, tg: TelegramClient, notion: Client) -> None
async def archive_channel(config: Config, conn: Connection, tg: TelegramClient, notion: Client, channel: ChannelRow) -> RunResult
async def process_post(conn: Connection, tg: TelegramClient, channel_id: int, message: Message, stage_dir: Path, log: IndentLogger) -> tuple[int, int]  # (cnt_media, cnt_comment)
async def save_media(conn: Connection, tg: TelegramClient, post_db_id: int | None, comment_db_id: int | None, message: Message, dest_dir: Path, log: IndentLogger) -> int
async def save_comment(conn: Connection, comment_msg: Message, post_db_id: int, log: IndentLogger) -> int
```

### `main.py`
```python
async def main() -> None
def run() -> None  # sync entry point: asyncio.run(main())
```

---

## Function Call Tree

```
run()
└── main()
    ├── load_config()
    ├── setup_logger()
    ├── init_db()
    ├── connect()                          [telegram_client]
    ├── notion.Client()
    ├── archive_all()                      [archiver]
    │   └── get_active_channels()          [db]
    │       └── [for each channel]:
    │           └── archive_channel()
    │               ├── get_last_successful_run()       [db]
    │               ├── resolve_channel()               [telegram_client]
    │               ├── update_channel_tg_id()          [db]
    │               ├── upsert_channel()                [db]
    │               ├── insert_run()                    [db]
    │               ├── create_run_record()             [notion_client]
    │               ├── update_channel_last_run()       [notion_client] (status=running)
    │               ├── [for each post via iter_posts()]:
    │               │   └── process_post()
    │               │       ├── post_exists()           [db]  → skip if true
    │               │       ├── insert_post()           [db]
    │               │       ├── save_media()
    │               │       │   ├── download_media_file()  [telegram_client]
    │               │       │   └── insert_media()         [db]
    │               │       └── [for each comment via iter_comments()]:
    │               │           └── save_comment()
    │               │               ├── comment_exists()   [db]  → skip if true
    │               │               ├── save_media()       (comment attachments)
    │               │               └── insert_comment()   [db]
    │               ├── update_run()                    [db]
    │               ├── update_run_record()             [notion_client]
    │               ├── update_channel_last_run()       [notion_client] (status=ok/error)
    │               └── send_run_summary()              [notifier]
    └── disconnect()                       [telegram_client]
```

---

## Logging Format

Every action is logged at an appropriate indent level using a tree structure:

```
[2024-03-20 03:00:01] archive_all: starting
  [channel] прошмандовцы (https://t.me/blacklistrussianemigration)
    resolved: @blacklistrussianemigration (tg_id=1234567890)
    last run: 2024-03-19 03:00:44 | fetching posts since then
    [post 12345] 2024-03-19 14:22:01 — "Some text..."
      media: 2 items downloaded
        image → media/blacklistrussianemigration/2024/03/12345_photo_0.jpg (84 KB)
        video → media/blacklistrussianemigration/2024/03/12345_video_0.mp4 (2.1 MB)
      comments: 3 archived
        [comment 98001] user: John Doe — "reply text"
        [comment 98002] user: anon — (no text, 1 media)
          image → media/.../98002_photo_0.jpg
        [comment 98003] user: Jane — "another reply"
    [post 12346] ...
    run complete: 14 posts, 22 media, 41 comments | status: ok
  [channel] next_channel ...
[2024-03-20 03:04:17] archive_all: done
```

---

## Improvements & Suggestions

### Data Model
1. **`channel.tg_id`** — use Telegram's numeric ID as the stable identifier; usernames change
2. **Post engagement metrics** — archive `views`, `forwards`, `reactions` for analytics
3. **`post.entities`** — store as JSON; enables reconstructing formatted text (links, hashtags, bold)
4. **`post.edit_date`** — detect edited posts on re-runs and update the record
5. **`post.grouped_id`** — Telegram sends multi-photo albums as separate messages sharing a `grouped_id`; needed to reconstruct albums correctly
6. **`post.no_forwards`** — channel owner may restrict saving/forwarding; flag this so the UI can mark restricted posts
7. **Comment threading** — `comment.reply_to_tg_id` allows full thread reconstruction
8. **Soft deletes** — add `is_deleted` flag to posts/comments (post may be deleted between runs)

### Backend
7. **Deduplication by `(channel_id, post_id)`** — always check before insert; safe to re-run
8. **Incremental sync by `post_id`** — use max archived `post_id` rather than `run_dttm`; TG IDs are monotonically increasing per channel, making this more reliable than datetime
9. **Flood wait handling** — catch `telethon.errors.FloodWaitError` and sleep for required duration
10. **Rate limiting** — respect Telegram's ~400 requests/60s limit; add small delays between channel iterations
11. **Media stage directory** — download to a temp stage, move to final path only on DB insert success; prevents partial files
12. **`--dry-run` flag** — run without writing to DB or Notion; useful for testing

### Notion
13. **Rollup columns** — `total posts` rollup requires Notion formula or relation; verify supported aggregation type at setup time
14. **Run log body** — store the indented tree log as the page content of each Log record for easy review

### Operations
15. **`.env.example`** — document all required env vars (TG API id/hash, phone, Notion token, DB path, media path, notify channel)
16. **Retry + backoff** — wrap all Notion and Telegram API calls with retry decorator (3 attempts, exponential backoff)
17. **`requirements.txt` pinning** — pin major versions to avoid breaking changes

---

## Required Environment Variables

```
TG_API_ID=           # from my.telegram.org
TG_API_HASH=         # from my.telegram.org
TG_PHONE=            # your Telegram phone number (for session auth)
TG_SESSION_NAME=     # e.g. tg_archive_session
TG_NOTIFY_CHANNEL=   # @username or -100xxx channel to send run summaries

NOTION_TOKEN=        # Integration secret from notion.so/my-integrations
NOTION_PARENT_PAGE_ID=  # Page ID where TG Archive databases live

DB_PATH=./tg_archive.db
MEDIA_PATH=./media
```
