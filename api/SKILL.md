---
name: sendblue-api
description: Send and receive iMessage, SMS, and RCS via the Sendblue HTTP API. Covers authentication, sending text/media/group messages, send styles (effects), reactions, typing indicators, status callbacks, and inbound webhooks. TRIGGER when the user mentions Sendblue, iMessage automation, blue-bubble messaging from code, or imports nothing provider-specific but asks for "send an iMessage from a server".
---

# Sendblue API

Sendblue is a REST API that sends iMessage (blue bubbles), SMS, and RCS from a provisioned phone number. Everything is plain JSON over HTTPS — no SDK is required.

## Base URL & Auth

```
https://api.sendblue.com
```

Every request needs two headers:

```
sb-api-key-id: <YOUR_API_KEY_ID>
sb-api-secret-key: <YOUR_API_SECRET>
Content-Type: application/json
```

Keep both values server-side — never ship them to a browser or mobile client.

## Quick Start — Send a Message

```bash
curl -X POST https://api.sendblue.com/api/send-message \
  -H 'sb-api-key-id: $KEY_ID' \
  -H 'sb-api-secret-key: $SECRET' \
  -H 'Content-Type: application/json' \
  -d '{
    "number": "+15551234567",
    "from_number": "+1YOUR_SENDBLUE_NUMBER",
    "content": "Hello from the API!"
  }'
```

Phone numbers must be E.164 (`+` + country code + number, no spaces or dashes). `from_number` must be a line you own — list yours with `GET /api/lines`.

## Core Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/send-message` | Send a 1:1 message (text and/or media) |
| POST | `/api/send-group-message` | Send to multiple recipients |
| POST | `/api/create-group` | Create a named group thread |
| POST | `/api/send-reaction` | Send a tapback (love/like/dislike/laugh/emphasize/question) |
| POST | `/api/send-typing-indicator` | Show "typing…" in the recipient's thread |
| POST | `/api/mark-read` | Send a read receipt |
| POST | `/api/upload-file` / `/api/upload-media-object` | Upload media (direct or from URL) |
| GET | `/api/status` | Poll a message's delivery status |
| GET | `/api/evaluate-service` | Check whether a number is on iMessage |
| GET | `/api/v2/messages` / `/api/v2/messages/:id` | Read message history |
| GET / POST / PUT / DELETE | `/api/v2/contacts[...]` | Manage contacts |
| GET | `/api/lines` | List your Sendblue phone numbers |
| POST | `/api/account/webhooks` | CRUD webhook subscriptions |

## Send Message — Full Shape

Request body:

```json
{
  "number": "+15551234567",
  "from_number": "+1YOUR_SENDBLUE_NUMBER",
  "content": "Optional text",
  "media_url": "https://example.com/img.jpg",
  "send_style": "celebration",
  "status_callback": "https://yourapp.com/sendblue/status"
}
```

- `content` and/or `media_url` — at least one is required.
- `send_style` — iMessage-only effect. Valid values: `celebration`, `shooting_star`, `fireworks`, `lasers`, `love`, `confetti`, `balloons`, `spotlight`, `echo`, `invisible`, `gentle`, `loud`, `slam`. Ignored on SMS.
- `status_callback` — Sendblue POSTs delivery updates here. Use this instead of polling `/api/status`.
- Limits: text up to 18,996 chars; media up to 100 MB on iMessage, 5 MB on SMS.

Response includes `message_handle` (Apple GUID — store this; you need it for reactions and replies) and a `status` from this set: `REGISTERED`, `PENDING`, `QUEUED`, `ACCEPTED`, `SENT`, `DELIVERED`, `DECLINED`, `ERROR`. Only `DELIVERED` means it landed.

## Group Messages

`POST /api/send-group-message` accepts an array of numbers or an existing `group_id`. The response returns a `group_id` — persist it to send follow-ups into the same thread instead of creating a new one each time.

```json
{
  "numbers": ["+15551234567", "+15557654321"],
  "from_number": "+1YOUR_SENDBLUE_NUMBER",
  "content": "Hey team"
}
```

## Reactions

```json
POST /api/send-reaction
{
  "from_number": "+1YOUR_SENDBLUE_NUMBER",
  "message_handle": "<message_handle from prior send>",
  "reaction": "love"
}
```

Reactions only work on iMessage and need the original message's `message_handle`. Valid values: `love`, `like`, `dislike`, `laugh`, `emphasize`, `question`.

## Receiving Messages (Webhooks)

Configure webhook URLs in the dashboard or via `POST /api/account/webhooks`. Sendblue POSTs JSON to your endpoint. **Respond with 2xx promptly** — non-2xx triggers retries and duplicate deliveries.

Webhook event types: `receive`, `outbound` (status updates), `typing_indicator`, `call_log`, `line_blocked`, `line_assigned`, `contact_created`.

Inbound message payload (`receive`):

```json
{
  "accountEmail": "you@example.com",
  "content": "Reply text",
  "media_url": "https://...",        // 30-day expiration — re-host if you need to keep it
  "is_outbound": false,
  "number": "+15551234567",          // the contact
  "from_number": "+1YOUR_SENDBLUE_NUMBER",
  "service": "iMessage",             // or "SMS"
  "group_id": "...",                 // present for group messages
  "date_sent": "2024-01-01T12:00:00Z"
}
```

Status callback payload (`outbound`) mirrors the send-message response and updates as the message moves through `SENT` → `DELIVERED` (or `ERROR`).

## Common Pitfalls

- **E.164 only.** `5551234567` or `(555) 123-4567` will fail — always send `+15551234567`.
- **`from_number` must be one of your lines.** A spoofed or unprovisioned number returns an error.
- **`send_style` silently no-ops on SMS.** If the recipient is green-bubble, effects don't render — check service first with `/api/evaluate-service` if it matters.
- **Store `message_handle`.** You need it for reactions, replies, and correlating status callbacks back to your records.
- **Media URLs expire in ~30 days.** If you need durable media from inbound webhooks, download and re-host on receipt.
- **Status is async.** A 200 on `/api/send-message` means accepted, not delivered. Use `status_callback` rather than blocking on the synchronous response.
- **Webhook retries on non-2xx.** Return 200 even when you've decided to ignore the event; otherwise expect duplicate deliveries.
- **Rate limits apply per line.** Bursting many sends from one number trips Apple's spam heuristics — pace them or split across lines.

## When to Reach for the Docs

Full reference: <https://docs.sendblue.com/>. Hit the docs when you need: carousels (`/api/send-carousel`), FaceTime/contact-card sharing, advanced webhook filtering, or the contacts API beyond basic CRUD.
