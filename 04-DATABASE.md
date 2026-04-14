# Database Schema

## Tables

### `contacts`

Stores contacts synced from WhatsApp, plus per-contact control settings.

```sql
CREATE TABLE contacts (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    name       TEXT    NOT NULL,
    phone      TEXT    NOT NULL DEFAULT '',
    jid        TEXT    NOT NULL UNIQUE,
    notes      TEXT    NOT NULL DEFAULT '',
    mode       TEXT    NOT NULL DEFAULT 'draft' CHECK(mode IN ('draft','auto','ignore')),
    llm_model  TEXT    NOT NULL DEFAULT '',
    last_read_at TEXT  NOT NULL DEFAULT '',
    created_at TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `name` | TEXT | Display name from WhatsApp |
| `phone` | TEXT | Phone number |
| `jid` | TEXT UNIQUE | WhatsApp JID |
| `notes` | TEXT | Freeform context passed to LLM prompt |
| `mode` | TEXT | `draft` \| `auto` \| `ignore` |
| `llm_model` | TEXT | Per-contact model override |
| `last_read_at` | TEXT | ISO timestamp of last read |
| `created_at` | TEXT | ISO timestamp |

### `conversations`

All message history stored locally.

```sql
CREATE TABLE conversations (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    contact_id INTEGER NOT NULL REFERENCES contacts(id),
    direction  TEXT    NOT NULL CHECK(direction IN ('in','out')),
    message    TEXT    NOT NULL,
    sent_at    TEXT    NOT NULL,
    wacli_id   TEXT    NOT NULL DEFAULT ''
);
CREATE INDEX idx_conversations_contact ON conversations(contact_id, sent_at);
CREATE INDEX idx_conversations_wacli_id ON conversations(wacli_id);
```

### `drafts`

AI-generated reply drafts.

```sql
CREATE TABLE drafts (
    id                   INTEGER PRIMARY KEY AUTOINCREMENT,
    contact_id           INTEGER NOT NULL REFERENCES contacts(id),
    conversation_context TEXT    NOT NULL DEFAULT '',
    draft_text           TEXT    NOT NULL,
    status               TEXT    NOT NULL DEFAULT 'pending'
        CHECK(status IN ('pending','approved','rejected','sent')),
    created_at           TEXT    NOT NULL DEFAULT (datetime('now')),
    sent_at              TEXT,
    variant_result_id    INTEGER NOT NULL DEFAULT 0,
    batch_id             TEXT    NOT NULL DEFAULT ''
);
CREATE INDEX idx_drafts_contact_status ON drafts(contact_id, status);
```

Status flow: `pending` → `approved` (human accepted) → `sent` | `rejected` (human discarded or auto-rejected).

### `debounce`

Persisted debounce timers for incoming message bursts.

```sql
CREATE TABLE debounce (
    jid              TEXT PRIMARY KEY,
    first_detected   TEXT NOT NULL,
    last_message_at  TEXT NOT NULL,
    respond_after    TEXT NOT NULL,
    delay_min_secs   INTEGER NOT NULL,
    delay_max_secs   INTEGER NOT NULL
);
```

### `message_cursors`

Tracks last-processed message per JID for the WhatsApp poller.

```sql
CREATE TABLE message_cursors (
    jid               TEXT PRIMARY KEY,
    last_processed_id TEXT NOT NULL,
    last_processed_at TEXT NOT NULL
);
```

### `audit_log`

All significant events with optional LLM token counts.

```sql
CREATE TABLE audit_log (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp             TEXT    NOT NULL DEFAULT (datetime('now')),
    actor                 TEXT    NOT NULL DEFAULT 'system',
    action                TEXT    NOT NULL,
    entity_type           TEXT,
    entity_id             TEXT,
    details               TEXT    NOT NULL DEFAULT '{}',
    error                 TEXT,
    duration_ms           INTEGER DEFAULT 0,
    llm_call              BOOLEAN DEFAULT FALSE,
    llm_prompt_tokens     INTEGER DEFAULT 0,
    llm_completion_tokens INTEGER DEFAULT 0
);
CREATE INDEX idx_audit_log_timestamp ON audit_log(timestamp);
```

### `settings`

Key-value store for global configuration.

```sql
CREATE TABLE settings (
    key        TEXT PRIMARY KEY,
    value      TEXT NOT NULL,
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

**Key settings:**

| Key | Default | Purpose |
|-----|---------|---------|
| `llm_model` | `openai/gpt-4.1-nano` | Default generation model |
| `llm_temperature` | `0.7` | Generation temperature |
| `context_messages` | `40` (DB) → `4` (code) | Messages to include in context |
| `debounce_min_secs` | `120` | Min debounce delay |
| `debounce_max_secs` | `600` | Max debounce delay |
| `draft_on_startup` | `false` | Generate drafts for all contacts on startup |
| `global_contact_mode` | `''` | Override all contacts to this mode |

Note: `context_messages` has a DB default of 40, but the actual code default is 4 (the DB default is a legacy value — the settings table also stores the authoritative value which the code reads).

### `contact_facts`

Extracted facts about contacts from conversation analysis.

```sql
CREATE TABLE contact_facts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    contact_id INTEGER NOT NULL,
    fact_type TEXT NOT NULL,
    fact_text TEXT NOT NULL,
    source_msg_id TEXT NOT NULL DEFAULT '',
    extracted_at TEXT NOT NULL DEFAULT (datetime('now')),
    UNIQUE(contact_id, fact_type, fact_text)
);
CREATE INDEX idx_contact_facts_contact ON contact_facts(contact_id);
```

### `draft_outcomes`

Tracks whether a sent draft got a reply, for scoring.

```sql
CREATE TABLE draft_outcomes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    draft_id INTEGER NOT NULL,
    contact_id INTEGER NOT NULL,
    sent_at TEXT NOT NULL,
    reply_received INTEGER NOT NULL DEFAULT 0,
    reply_at TEXT NOT NULL DEFAULT '',
    reply_length INTEGER NOT NULL DEFAULT 0,
    reply_latency_secs INTEGER NOT NULL DEFAULT 0,
    score REAL NOT NULL DEFAULT 0,
    model TEXT NOT NULL DEFAULT '',
    variation_hint TEXT NOT NULL DEFAULT '',
    scored_at TEXT NOT NULL DEFAULT ''
);
CREATE INDEX idx_draft_outcomes_contact ON draft_outcomes(contact_id, reply_received);
```

### `app_logs`

Application-level health/log entries.

```sql
CREATE TABLE app_logs (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    level     TEXT NOT NULL,
    component TEXT NOT NULL,
    message   TEXT NOT NULL,
    context   TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```