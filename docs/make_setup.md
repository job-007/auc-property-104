# Make.com ↔ Claude routine setup

How the deal folder automation fits together. The routine prompt is in `routine/PROMPT.md` and the
JSON Make receives is defined in `routine/make_payload.schema.json`.

```
GHL card → UNDER ANALYSIS
   │
   ▼
Make scenario A ── POST /fire ──▶ Claude routine ── PropertyEngine connector
                                       │
                                       ▼
Make scenario B ◀── POST JSON ── (photo URLs, details, comparables)
   │
   ├─ download each photo URL → Google Drive /Photos
   ├─ floor plan URL → Google Drive /Floor Plan
   ├─ comparables → CSV → Google Drive /Comparables
   └─ update GHL "Google Drive link" + Slack message
```

## 1. Claude side (one-off)

1. Add the PropertyEngine connector at https://claude.ai/customize/connectors
   (URL `https://propertyengine.co.uk/mcp`, "Sign in now", "Use Claude's published identity"),
   and sign in with the team email and the 6-digit code.
2. Create the routine at https://claude.ai/code/routines:
   - **Prompt**: the text from `routine/PROMPT.md`.
   - **Repository**: this repository (the prompt reads the schema from it).
   - **Connectors**: PropertyEngine only. Remove the rest.
   - **Environment**: one whose network access allows your Make webhook host
     (for example `hook.eu1.make.com`; check the region in your webhook URL).
   - **Environment variables**: `MAKE_WEBHOOK_URL` (from scenario B) and `MAKE_WEBHOOK_SECRET`
     (a long random string you also put in scenario B's filter).
   - **Trigger**: API. Copy the **Fire URL**, click **Generate token** and copy the token
     straight into Make. It is shown once.

## 2. Make scenario B: receive the bundle (build this first)

1. **Webhooks → Custom webhook**. Copy its URL into the routine's `MAKE_WEBHOOK_URL`.
   Turn on "Get request headers".
2. **Filter**: header `x-auc-secret` equals `MAKE_WEBHOOK_SECRET`. Drop anything else.
3. **Router** on `status`:
   - `not_found` / `error` → Slack message with `errors[]` and `claude_session_url`. Stop.
   - `ok` / `partial` → continue.
4. **Google Drive → Create a folder** named from `details.address_line` and `details.postcode`,
   then the subfolders Photos, Floor Plan and Comparables.
5. **Iterator** on `photos[]` → **HTTP → Get a file** (`url`) → **Google Drive → Upload a file**
   into Photos, file name `{{if(index < 10; "0" + index; index)}}.jpg`.
6. If `floorplan_urls[]` is not empty: Iterator → Get a file → Upload into Floor Plan.
7. **Array aggregator** on `comparables[]` → **CSV → Create CSV** → Upload `comparables.csv`
   into Comparables.
8. **GHL → Update opportunity** (`opportunity_id`) custom field "Google Drive link".
9. **Slack** message with the folder link, plus `errors[]` if `status` is `partial`.

Download the photos as soon as the webhook arrives: discovery will tell us whether the URLs
expire.

## 3. Make scenario A: start the routine

1. Trigger: the GHL "opportunity stage changed → UNDER ANALYSIS" webhook.
2. **HTTP → Make a request**:
   - Method `POST`, URL = the routine's Fire URL
     (`https://api.anthropic.com/v1/claude_code/routines/trig_.../fire`)
   - Headers:
     - `Authorization: Bearer <routine token>`
     - `anthropic-beta: experimental-cc-routine-2026-04-01`
     - `anthropic-version: 2023-06-01`
     - `Content-Type: application/json`
   - Body type Raw, JSON:
     ```json
     { "text": "opportunity_id={{opp.id}}; propertyengine_url={{opp.pe_link}}; address={{opp.address}}; postcode={{opp.postcode}}" }
     ```
   - Parse response: yes. The reply is `{ "type": "routine_fire", "claude_code_session_id": ..., "claude_code_session_url": ... }`,
     not the property data. That arrives in scenario B a few minutes later.
3. Optional: store `opportunity_id` + `claude_code_session_url` in a Make data store, and have a
   scheduled check that alerts Slack when no bundle has arrived after 15 minutes.

## Limits

- The `/fire` API is in research preview; the beta header may change (older versions keep working
  for a while).
- 30 API fires per hour per routine. Each run uses the Claude Pro plan's allowance.
- A run can occasionally stop to ask for approval; open `claude_code_session_url` to answer it.
