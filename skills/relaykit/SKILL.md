---
name: relaykit
description: Send text messages from an application through RelayKit. Use when adding, changing or testing SMS in an app that uses RelayKit, or when a task mentions RelayKit, RELAYKIT_API_KEY or api.relaykit.ai.
---

# RelayKit

RelayKit sends text messages for applications: appointment reminders, login codes, order updates, support replies, team alerts. The application never writes the words. It names a message — a `namespace` and an `event` — and passes the data that message needs; RelayKit renders the wording the app's owner chose in their RelayKit workspace and sends it. Carrier registration, opt-out handling and the rules about what a text may say are RelayKit's side of the line. Everything below is plain HTTPS against `https://api.relaykit.ai/v1` with a bearer key. There is no package to install.

## Procedure

1. **Read the catalog before writing a send.** `GET https://relaykit.ai/corpus.json` lists every message with its `corpus_id` — the exact `namespace:event` to send — and its `variables`, the exact `data` fields that send needs. The few names in this file are illustrations; the catalog is the truth. A workflow step's `variable_aliases` are that industry's example values, not extra fields, and a step whose `corpus_id` is null is on the page but not yet sendable by name — skip it. Today it has these namespaces:

<!-- gen:skill-vocabulary:start -->
| Namespace | What it covers | Messages |
|---|---|---|
| `verification` | Verification | 4 |
| `appointments` | Appointments | 15 |
| `order-updates` | Order updates | 10 |
| `digital-delivery` | Digital delivery | 3 |
| `customer-support` | Customer support | 9 |
| `team-alerts` | Team alerts | 15 |
| `community` | Community | 9 |
| `waitlist` | Waitlist | 6 |
| `account-events` | Account events | 18 |
| `documents` | Documents | 6 |
| `marketing` | Marketing | 4 |
<!-- gen:skill-vocabulary:end -->

2. **Get the key from the person, never from the network.** The app reads `RELAYKIT_API_KEY` from its environment. If it isn't set, stop and ask: their key is in the RelayKit workspace at app.relaykit.ai, under Settings → API keys. Don't invent a key, don't look for a signup endpoint, and don't put a key in source.

3. **Send by name.** One request, one message:

<!-- gen:skill-send:start -->
```
POST https://api.relaykit.ai/v1/messages
Authorization: Bearer rk_test_...
Content-Type: application/json

{
  "namespace": "appointments",
  "event": "reminder-proximate",
  "to": "+15551234567",
  "data": {
    "workspace_name": "Acme Engineering",
    "provider_name": "Dr. Sarah Chen",
    "appointment_time": "Tue, March 4th, 2:00 PM",
    "cancel_link": "yourapp.com/cancel"
  },
  "tone": "friendly"
}
```

One of three things comes back:

```json
200 { "id": "msg_...", "status": "sent", "timestamp": "..." }
202 { "id": "msg_...", "status": "queued", "timestamp": "..." }
200 { "id": null, "status": "blocked", "reason": "..." }
```
<!-- gen:skill-send:end -->

   `data` carries every variable the message names — the catalog's `variables` array for that message, nothing more and nothing less. Pass the ones the catalog tags `workspace settings` too; the API doesn't fill them, and a missing one is a 400 that names it. `tone` is optional: `standard`, `friendly` or `brief`, lowercase even where the catalog capitalises it; leave it out for `standard`.

4. **Preview before the first real send from any new call site.** `POST /v1/messages/preview` takes the same body and the same auth, sends nothing, and returns the rendered text as `message`. Show it to the person.

5. **Verify with a phone, not a status code.** A test key (`rk_test_…`) only reaches numbers that have verified as testers: `POST /v1/sandbox/recipients` with `{ "phone": "+1…" }`, RelayKit texts that phone a code, and the person replies YES (or enters the code). Send to a verified tester, then ask the person whether their phone buzzed. That is the test. A 200 is not. `GET /v1/messages` also lists RelayKit's own texts to that phone — the verification code, under `namespace: system` — so count only your namespaces.

## Pitfalls

- **Consent comes first.** The place the app collects a phone number says, next to the field, what texts the person will get. Don't add a send to a flow that never asked.
- **`to` is E.164** — `+` and country code, no spaces.
- **`blocked` is not an error.** It is a 200 with `status: "blocked"` and a `reason` (the recipient opted out, the number isn't a verified tester, a rate limit). Read the reason, surface it, move on. There is nothing to retry.
- **`queued` (202) happens only on marketing sends** — RelayKit is holding the text until the recipient's quiet hours end. Nothing else queues.
- **Marketing needs `marketing_consent: true`** on every send in the `marketing` namespace and on `appointments.time-to-rebook`. Without it the request is a 422 `marketing_consent_required`, and that is the app's attestation, not a flag to set blindly.
- **Retries carry an `Idempotency-Key` header** — any string, unique per logical send. The same key inside 24 hours replays the first result instead of sending twice. A blocked result is replayed too — once the cause is fixed (the tester verified, consent recorded), retry with a new key.
- **A test key reaches verified testers only.** A send to anyone else is blocked with `recipient_not_verified`. A live key comes later, from the same workspace, once the business is registered — not from code.
- **A few older messages have one rendering.** Asking for a `tone` on one is a 422 `tone_not_available`; drop the field.

## When a send fails twice

Stop. Re-read `https://relaykit.ai/docs/v1.md`, check whether the key is a test key or a live key and whether the recipient is a verified tester, and ask the person. Don't loop on retries, and don't change the message name or the fields until an error goes away — a 422 names what is missing.

## The rule that doesn't bend

Message wording lives in RelayKit, not in the code. Never write a text message body in application code, never assemble one from strings, and never send through another SMS provider as a fallback. When the words should change, the app's owner changes them in the workspace and the code stays the same.

## Pointers

- `https://relaykit.ai/docs/v1.md` — every request and response shape, plus message history, opt-outs and consent.
- `https://relaykit.ai/corpus.json` — the catalog.
- `https://relaykit.ai/AGENTS.md` — this guide, readable.
- If a RelayKit MCP server is connected, prefer its tools for the catalog and for test sends.
