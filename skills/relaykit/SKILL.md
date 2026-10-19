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
| `verification` | Verification | 5 |
| `appointments` | Appointments | 97 |
| `order-updates` | Order updates | 13 |
| `digital-delivery` | Digital delivery | 3 |
| `customer-support` | Customer support | 19 |
| `team-alerts` | Team alerts | 40 |
| `community` | Community | 10 |
| `waitlist` | Waitlist | 15 |
| `account-events` | Account events | 58 |
| `documents` | Documents | 11 |
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
    "your_brand": "Acme Engineering",
    "appointment_time": "Tue, Oct 6, 9:40 PM",
    "business_phone": "843-555-0142",
    "reschedule_link": "yourapp.com/reschedule",
    "provider_name": "Sam Rivera",
    "location": "118 Elm St",
    "service_name": "yearly service"
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

   `data` carries every variable the message names (the catalog's `variables` array for that message), with one exception: `your_brand`. RelayKit fills that one from the brand on the account, so leave it out; a value passed for it is ignored. Everything else the catalog lists is yours to pass, including the ones it tags `workspace settings`, and a missing one is a 422 `invalid_data` that names it. Links: use a short link on the business's own domain. Public shorteners like bit.ly get texts blocked.

   When the person's workspace skill names a card for the text, pass `card` (`"workflow/text"`) and leave `tone` out: RelayKit sends the card's words as the workspace shows them. `card` with `tone` is a 400. A card that is off, sends another `namespace` and `event`, or whose words cannot send is a 422 `card_not_sendable` that names the card and the reason, on a test key as on a live one. Without a card, `tone` is optional: `standard`, `friendly` or `brief`, lowercase even where the catalog capitalises it; leave it out for `standard`.

   Any of the three answers can carry `warnings: [{ code, message }]`. `reply_route_not_ready` is a text that asks for a reply, sent before the replies tests pass. `use_report` is a text the workspace sends when the app sends RelayKit a visit's moment (step 6). Fix what each says before going live.

4. **Preview before the first real send from any new call site.** `POST /v1/messages/preview` takes the same body and the same auth, sends nothing, and returns the rendered text as `message`. Show it to the person.

5. **Verify with a phone, not a status code.** A test key (`rk_test_…`) only reaches numbers that have verified as testers: `POST /v1/sandbox/recipients` with `{ "phone": "+1…" }`, RelayKit texts that phone a code, and the person replies YES (or enters the code). Send to a verified tester, then ask the person whether their phone buzzed. That is the test. A 200 is not. `GET /v1/messages` also lists RelayKit's own texts to that phone — the verification code, under `namespace: system` — so count only your namespaces.

6. **Send us each visit's moments.** In a workspace whose workflow runs on visits, the workspace skill marks those texts with a moment. Don't send them by name: send RelayKit what happened to the visit, and RelayKit sends each text at its moment, times the reminders, moves them when the visit moves and drops them when it is canceled. `POST /v1/visits` with `{ "ref": "<your id for the visit>", "moment": "booked", "workflow": "<workflow id>", "to": "+1…", "window": { "start": "…-04:00", "end": "…-04:00" }, "data": { … } }`, each time a visit is `booked`, `moved` or `canceled`, someone is `on_my_way` or `running_late`, the visit is `started` or `done`, or the customer has `paid`. The workspace skill lists which moments its texts need and the values each call carries.

7. **Take replies.** People can text back. RelayKit passes every reply except STOP, HELP and its other keywords to one route in the app: it POSTs `{"type":"reply.received","id":"rpl_…"}`. Answer 200, then read the reply with `GET /v1/replies/{id}`. Set the route's address with `PUT /v1/replies/endpoint` `{ "url": "https://…" }` and prove it with `POST /v1/replies/endpoint/test`. On localhost, read `GET /v1/replies?since=…` instead. Show every reply to the business; never text back automatically. A reply to a text RelayKit sent for a visit arrives with `word` and `visit`: RelayKit has already texted the answer, so do what the word means (mark the visit confirmed) and send nothing.

## Pitfalls

- **Consent comes first.** The place the app collects a phone number says, next to the field, what texts the person will get. Don't add a send to a flow that never asked.
- **`to` is E.164** — `+` and country code, no spaces.
- **`blocked` is not an error.** It is a 200 with `status: "blocked"` and a `reason`: the recipient opted out, the number isn't a verified tester, a rate limit, or the same words went to the same phone in the last ten minutes (`duplicate_send`). Read the reason, surface it, move on. There is nothing to retry.
- **400 is the request's shape, 422 is what RelayKit cannot send.** A 400 `invalid_data` names the field to fix. A 422 names the code and the reason: `invalid_data` (a missing value), `card_not_sendable`, `reply_route_not_ready`, `template_not_found`, `tone_not_available`, `brand_not_set`, `marketing_consent_required`.
- **`queued` (202) happens only on marketing sends** — RelayKit is holding the text until the recipient's quiet hours end. Nothing else queues.
- **Marketing needs `marketing_consent: true`** on every send in the `marketing` namespace and on `appointments.time-to-rebook`. Without it the request is a 422 `marketing_consent_required`, and that is the app's attestation, not a flag to set blindly.
- **Retries carry an `Idempotency-Key` header** — any string, unique per logical send. The same key inside 24 hours replays the first result instead of sending twice. A blocked result is replayed too — once the cause is fixed (the tester verified, consent recorded), retry with a new key.
- **A test key reaches verified testers only.** A send to anyone else is blocked with `recipient_not_verified`. A live key comes later, from the same workspace, once registration is approved — not from code.
- **A few older messages have one rendering.** Asking for a `tone` on one is a 422 `tone_not_available`; drop the field.
- **A reply can arrive twice** — save replies by `id`. And a reply to a text sent by name never says which text it answers; match it by `from` to the most recent text the app sent that number.

## When a send fails twice

Stop. Re-read `https://relaykit.ai/docs/v1.md`, check whether the key is a test key or a live key and whether the recipient is a verified tester, and ask the person. Don't loop on retries, and don't change the message name or the fields until an error goes away — a 422 names what is missing.

## The rule that doesn't bend

Message wording lives in RelayKit, not in the code. Never write a text message body in application code, never assemble one from strings, and never send through another SMS provider as a fallback. When the words should change, the app's owner changes them in the workspace and the code stays the same.

## Pointers

- `https://relaykit.ai/docs/v1.md` — every request and response shape, plus message history, replies, opt-outs and consent.
- `https://relaykit.ai/corpus.json` — the catalog.
- `https://relaykit.ai/AGENTS.md` — this guide, readable.
- If a RelayKit MCP server is connected, prefer its tools for the catalog and for test sends.
