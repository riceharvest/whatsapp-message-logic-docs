# API Endpoints

## Backend: Port 3000

### Auto-Generate Draft

```
POST /api/auto/generate
```

Generate a message draft without sending.

**Request body:**
```json
{
  "chatId": "31612644205@s.whatsapp.net",
  "styleName": "Casual",        // optional
  "customPrompt": "...",        // optional override
  "contextLimit": 20            // optional, max 100
}
```

**Response:**
```json
{
  "reply": "ewa ben vrij maar ga chillen",
  "cost": 3,
  "tokens": 1240
}
```

---

### Contacts

```
GET /api/contacts
```
Returns all contacts sorted by `lastMessageTs` descending.

```
POST /api/contacts/sync
```
Trigger manual full contact sync from wacli. Returns `{ success, synced, changed }`.

```
PUT /api/contacts/:id
```
Update contact settings.

**Body:**
```json
{
  "auto_mode": "draft",
  "style_prompt": null,
  "auto_keywords": ["order", "update"],
  "context_limit": 20,
  "label": "Lead",
  "ignored": false
}
```

---

### Messages

```
GET /api/chats/:id/messages?limit=50
```
Fetch messages for a chat via wacli (live from WhatsApp, not local DB).

---

### Send Reply

```
POST /api/chats/:id/reply
```
Send a manual reply.

**Body:** `{ "text": "Yesss" }`

---

### Outbound Queue

```
GET /api/queue
```
List all pending + draft messages.

```
DELETE /api/queue/drafts
```
Cancel all pending drafts.

---

### Usage Stats

```
GET /api/stats
```
Today's tokens + cost in cents.

---

### Styles

```
GET /api/styles
```
List all style presets (DEFAULT_STYLES + DB presets + styleguide.md if present).

```
POST /api/styles
```
Create a new custom style preset.

---

### Search

```
GET /api/search?q=order
```
Full-text search across contacts + messages.

---

### WhatsApp Admin

```
POST /api/admin/start
```
Start the wacli connection.

```
GET /api/admin/status
```
Return `{ connected: true|false }`.

---

## Frontend: Port 5173 (static)

Served by the Hono backend via `@hono/node-server/serve-static`. No separate API — all communication is via WebSocket + the REST endpoints above.

## WebSocket — `/ws`

Real-time updates pushed to the UI.

### Client → Server

No client-initiated messages (server is fire-and-forget).

### Server → Client

```ts
// New inbound message
{ type: "new_message", chatId, contactId, contact, body, fromMe: false, ts }

// Outbound message sent
{ type: "message_sent", chatId, body, ts }

// Draft generated
{ type: "draft_ready", chatId, reply, cost, ts }

// Contact updated
{ type: "contact_updated", contact, ts }

// Full contact list after sync
{ type: "contacts_synced", contacts, ts }

// Drafts cancelled
{ type: "drafts_cancelled", count, ts }
```

---

## Environment Variables

```env
OPENROUTER_API_KEY=sk-or-...     # Required
OPENROUTER_MODEL=openai/gpt-4o-mini  # Optional, default: openai/gpt-4o-mini
PORT=3000                        # Optional, default: 3000
```
