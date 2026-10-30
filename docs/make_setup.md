# Deal Staker: Make ↔ Claude routine setup

Make folder "Deal Staker" (team 2348752, zone eu1). The routine prompt is `routine/PROMPT.md`; the JSON
the routine sends to Part 2 is defined in `routine/make_payload.schema.json` (example:
`routine/sample_bundle.json`).

```
GHL card → UNDER ANALYSIS
   │  webhook ghl-deal-stacker-under-analysis (x-make-apikey)
   ▼
Deal Staker - AUC-102 (8165648)
   search Drive → no folder yet → create deal folder + Photos/Videos/Floorplan/Comparables
   → GHL: write Drive link → HTTP POST to Part 1 (opportunity + folder IDs)
   │  webhook deal-staker-auc-104-part-1 (x-make-apikey)
   ▼
Deal Staker - AUC-104 - Part 1 (8255125)
   → HTTP POST to the Claude routine Fire URL (returns at once with a session link)
   ▼
Claude routine "AUC deal bundle" (trig_01RLU5DXVkTJqrNGAAJ5f6Kx) ── PropertyEngine connector (get_property)
   │  POST JSON with x-make-apikey
   ▼  webhook deal-staker-auc-104-part-2
Deal Staker - AUC-104 - Part 2 (8255121)
   router:
   ├─ each photo URL → Download a file → Drive: Photos/01-exterior.jpg …
   ├─ each floor plan URL → Download a file → Drive: Floorplan/floorplan-1.png
   ├─ PropertyData sold prices (postcode, 0.5 mi, 12 months, same type) → freehold rows only
   │    → Drive: Comparables/comparables.csv
   └─ details → Drive: deal folder/PropertyEngine summary.txt
```

| Scenario | ID | Webhook | Webhook URL |
|---|---|---|---|
| Deal Staker - AUC-102 | 8165648 | ghl-deal-stacker-under-analysis (3972455) | `https://hook.eu1.make.com/t6w6i964milync86m0zcrgp4lsju90fj` |
| Deal Staker - AUC-104 - Part 1 | 8255125 | deal-staker-auc-104-part-1 (4001785) | `https://hook.eu1.make.com/detj1a328lm2ahwpu1dg8ps57cjiyvvq` |
| Deal Staker - AUC-104 - Part 2 | 8255121 | deal-staker-auc-104-part-2 (4001786) | `https://hook.eu1.make.com/4goqouy6g32cvpusniiziwyn3rvvnjj3` |

All three webhooks require the header `x-make-apikey` with the value of the Make key
"ghl-webhook-key" (the one GHL already sends).

## One-off setup

### 1. Make: two API key connections (Make → Connections → Add, type "API Key")

| Connection name | Key name | Key value | Placement | Allowed service origin |
|---|---|---|---|---|
| Make webhook key | `x-make-apikey` | the ghl-webhook-key value | Header | `https://hook.eu1.make.com` |
| Claude routine token | `Authorization` | `Bearer <routine token>` | Header | `https://api.anthropic.com` |

Then select them in:

- AUC-102 → module "HTTP to Deal Staker - AUC-104 - Part 1" → **Make webhook key**
- AUC-104 Part 1 → module "Start Claude routine (PropertyEngine)" → **Claude routine token**

### 2. GHL: send the property links

In the GHL workflow's webhook action to AUC-102, the Custom Data must include:

| Key | Value (GHL merge field) |
|---|---|
| `opportunity_id` | opportunity ID (already there) |
| `opportunity_name` | opportunity name (already there) |
| `postcode` | postcode (already there) |
| `propertyengine_link` | the opportunity's "Property Engine Link" field (already there) |
| `rightmove_link` | the opportunity's "Rightmove Link" field (already there) |

AUC-102 passes these to Part 1, which sends them to the routine (`propertyengine_url`, and
`rightmove_url` as `portal_url`). Without either link the routine falls back to an address +
postcode search, which only finds properties currently on the market.

### 3. Claude routine (claude.ai/code/routines → New)

- **Prompt**: everything below the line in `routine/PROMPT.md`.
- **Connectors**: PropertyEngine MCP only.
- **Environment**: network access Custom, allowed domains `hook.eu1.make.com` (keep the package
  manager defaults ticked). Environment variables:
  ```
  MAKE_WEBHOOK_URL=https://hook.eu1.make.com/4goqouy6g32cvpusniiziwyn3rvvnjj3
  MAKE_WEBHOOK_API_KEY=<the ghl-webhook-key value>
  ```
- **Trigger**: API → copy the Fire URL, Generate token (shown once).
- Put the Fire URL into AUC-104 Part 1's HTTP module (replace `REPLACE_WITH_ROUTINE_TRIGGER_ID`), and the
  token into the "Claude routine token" connection.

### 4. Test, in this order

1. Routine → **Run now** with:
   `opportunity_id=test-1; propertyengine_url=https://propertyengine.co.uk/property/VsBsijE3n; deal_folder_id=<id>; photos_folder_id=<id>; floorplan_folder_id=<id>; comparables_folder_id=<id>`
   using the IDs of a test deal folder. Part 2 must be switched on (or "Run once") to receive it.
2. Turn on Part 1 and Part 2, then move a real card to UNDER ANALYSIS.

## Notes

- AUC-102 only calls Part 1 when it creates a new folder; moving a card again just refreshes the GHL link.
- The routine API is in research preview: 30 fires per hour per routine, and each run uses the Claude
  plan's allowance. A run can occasionally pause for approval; open its session link to answer.
- Photo URLs are on `media.propertyengine.co.uk` (small size). Download them as soon as the bundle
  arrives.
- Known AUC-102 issues still open: GHL can fire twice at once (turn on sequential processing), and the
  folder search uses the raw opportunity name while the folder is created with `/` and `\` replaced.

## AUC-109: Deal Stacker (runs in parallel with AUC-104)

`Deal Staker - AUC-109` (8260214), webhook `deal-staker-auc-109` (4003439,
`https://hook.eu1.make.com/a8gljbgqqonwodwucp4rwm3yu398l6ey`, `x-make-apikey`). Called by AUC-102
right after the AUC-104 call, only when a new deal folder is created.

1. Google Drive (Anastasiya's connection) copies the Master
   (`1Pdow8Q2bjEtN4lNbq-KzBAyDnjoZOn8u9niE_T9Q0aY`) into the deal folder as
   `Stacker - <opportunity name>`.
2. Google Sheets (Julian's connection) writes, on tab `" Numbers Overview All Options "` (the tab
   name has a leading and trailing space): C10 = `guide_price`, C44 = `new_market_value`,
   F69 = `beds_potential`, F72 = `lha_weekly`. Each cell is skipped when GHL sends no value.
3. In parallel with steps 1–2, Google Drive copies "Sample Schedules"
   (`1gUafgwbDwn174LwVqeP-JqrXLfbvS4ICt9uNmgAlmpQ`) into the deal folder as `SoW - <opportunity name>`.

## AUC-110: EPC report PDF (runs in parallel with AUC-104 and AUC-109)

```
AUC-102 (new folder only) → webhook deal-staker-auc-110-part-1
Deal Staker - AUC-110 - Part 1 (8277679)
   EPC API search by postcode (connection "EPC API (new)")
   → keep certificates whose address appears in the opportunity name → newest first
   → HTTP POST to the "AUC EPC PDF" routine Fire URL (certificate print link, file name, folder ID)
Claude routine "AUC EPC PDF" (trig_01JMcrrgYCCvummVnfmYcfdf, routine/epc_pdf/PROMPT.md) → headless Chromium → PDF
   │  POST JSON (pdf_base64) with x-make-apikey
   ▼  webhook deal-staker-auc-110-part-2
Deal Staker - AUC-110 - Part 2 (8277682) → Drive: deal folder/EPC - <opportunity name>.pdf
```

| Scenario | ID | Webhook | Webhook URL |
|---|---|---|---|
| Deal Staker - AUC-110 - Part 1 | 8277679 | deal-staker-auc-110-part-1 (4008987) | `https://hook.eu1.make.com/2kjesi8ika276qpnf18ddanvj35htb62` |
| Deal Staker - AUC-110 - Part 2 | 8277682 | deal-staker-auc-110-part-2 (4008988) | `https://hook.eu1.make.com/ni6vmr73jq4pco40wuhti1af4d3kabzd` |

- Matching: the certificate's address (commas removed) must appear as whole words in the opportunity
  name, so "13 Chester Court" matches "TEST 13 Chester Court, Haverhill" but "3 Chester Court" does
  not. No match, no postcode or no certificate means no PDF.
- A certificate registered more than 10 years ago is still saved, with " (expired)" in the file name.
- Setup: create the routine (no connectors, environment "AUC routine" with
  `find-energy-certificate.service.gov.uk` added to the allowed domains and
  `MAKE_EPC_WEBHOOK_URL=https://hook.eu1.make.com/ni6vmr73jq4pco40wuhti1af4d3kabzd` added), then put
  its Fire URL and a new "Claude EPC routine token" connection into Part 1's last module, and turn
  Part 1 on.
