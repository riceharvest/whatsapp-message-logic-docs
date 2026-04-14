# API Endpoints

## Base URL

Backend serves HTTP on port 8080 (configurable via `--http-port` flag).

---

## Contacts

### `GET /api/contacts`

Returns all contacts sorted by `last_read_at` descending.

**Response:** `200`
```json
[{
  "id": 12,
  "name": "Jess",
  "phone": "31612345678",
  "jid": "31612345678@s.whatsapp.net",
  "notes": "met on Hinge, 3rd date",
  "mode": "draft",
  "llm_model": "",
  "stage": "flirty",
  "last_read_at": "2026-04-14T08:00:00Z",
  "created_at": "2026-03-15T10:00:00Z"
}]
```

---

### `GET /api/contacts/{id}`

Get a single contact.

---

### `PATCH /api/contacts/{id}`

Update contact settings.

**Request body:**
```json
{
  "mode": "auto",
  "notes": "updated notes",
  "llm_model": "openai/gpt-4o-mini",
  "context_messages": 6
}
```

All fields optional. Omitted fields are not changed.

---

### `PATCH /api/contacts/draft-mode`

Set mode for all non-ignored contacts.

**Request body:** `{"mode": "draft"|"auto"|"ignore"}`

---

### `GET /api/contacts/{id}/draft-history`

Get last N saved drafts for a contact.

**Query params:** `?limit=20`

---

### `POST /api/contacts/{id}/sync-history`

Sync message history from wacli for a contact.

---

### `POST /api/contacts/import`

Bulk import contacts.

---

### `GET /api/contacts/{id}/prompt-staleness`

Check if the contact's prompt/needs have changed significantly since last draft.

---

## Message Generation

### `POST /api/contacts/{id}/generate`

**Primary generation endpoint.** Generates draft(s) via SSE stream, saves to DB.

**Request body:** none

**Response:** `text/event-stream`

Each event:
```
data: {"idx":0,"text":"hey what's up 😊","draft_id":445,"model_used":"openai/gpt-4.1-nano","cost":0.0023}
```

Errors:
```
data: {"idx":0,"error":"generation already in progress"}
```

**Draft count:** 1 (manual generate). The web UI button and `/generate` command both use this endpoint.

**Alias:** `POST /api/contacts/{id}/generate-stream` — same handler.

---

### `POST /api/contacts/{id}/send-draft`

Send a saved draft (human-approved or auto-sent).

**Request body:**
```json
{
  "draft_id": 445,
  "edited_text": "optional edited version before sending"
}
```

**Response:**
```json
{"ok": true, "message": "sent", "wacli_id": "true_31612345678@whatsapp.net_ABC123"}
```

---

### `POST /api/contacts/{id}/reject-all-drafts`

Reject all pending drafts for a contact.

---

### `GET /api/contacts/{id}/momentum`

Returns conversation momentum data (reply rate, frequency trends).

---

### `GET /api/contacts/{id}/drift`

Returns style drift analysis — how much the recent drafts differ from chosen history.

---

### `GET /api/contacts/{id}/fact-relevance`

Returns extracted facts about the contact and their relevance to current draft.

---

### `GET /api/contacts/{id}/opener-suggestions`

Returns suggested openers for a contact with no/limited history.

---

### `POST /api/contacts/{id}/draft-diversity`

Force generation of stylistically diverse drafts (ignores normal duplicate suppression).

---

### `GET /api/contacts/{id}/topics`

Returns conversation topic analysis for this contact.

---

### `GET /api/contacts/{id}/dead-check`

Check if contact is "dead" (no activity for a threshold).

---

### `GET /api/contacts/{id}/urgent-messages`

Returns messages flagged as potentially urgent.

---

### `GET /api/contacts/{id}/health`

Returns contact-level health metrics.

---

## Settings

### `GET /api/settings`

List all settings.

---

### `POST /api/settings`

Set a setting.

**Request body:** `{"key": "context_messages", "value": "6"}`

---

## Style Guide

### `GET /api/style-guide`

Returns the current style guide text.

---

### `POST /api/style-guide`

Update the style guide text.

---

## Search

### `GET /api/search?q=query`

Full-text search across contacts and messages.

---

## AB Testing

### `GET /api/ab/model-stats`

Returns per-model draft quality stats (reply rate, avg score) for AB test analysis.

---

## Health

### `GET /health`

Returns `{"ok": true, "uptime": "4h30m", ...}`

---

## Outbound (Legacy)

### `POST /contacts/{id}/send-json`

Send a raw JSON message directly via wacli.

---

## WebSocket

The server pushes real-time events over WebSocket (path: `ws://host/ws`).

### Server → Client events

| Event | Payload | When |
|-------|---------|------|
| `new_message` | `{contact_id, name, direction, message, sent_at}` | New WA message arrived |
| `draft_created` | `{contact_id}` | Draft saved to DB |
| `drafts_cancelled` | `{contact_id, count}` | Pending drafts rejected |
| `contact_updated` | `{contact}` | Contact settings changed |

No client → server messages. Client just receives events and updates UI accordingly.