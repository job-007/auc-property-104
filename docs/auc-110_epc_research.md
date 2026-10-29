# AUC-110: Add EPC report to Google Drive, research

Researched on 29 Oct 2026. The live requests were made from Make (temporary scenario, since deleted)
for the property in the ticket video: 13 Chester Court, Haverhill, CB9 8LT.

## What the ticket video shows

1. Take the postcode from the GHL opportunity (CB9 8LT).
2. GOV.UK "Find an energy certificate", search by postcode.
3. Open the certificate for the matching address (13, Chester Court: rating D, valid until
   24 Nov 2029, certificate number 8696-9156-1929-4226-6913).
4. Print, then save as PDF (4 pages), and put the PDF in the deal folder.

## Findings

| Question | Result |
|---|---|
| Old EPC API (`epc.opendatacommunities.org`, used by the existing "EPC Register API KEY" connection) | **Retired.** Returns `301 Moved Permanently`. This is also why Julian's PropertyData/EPC chain failed with "redirect destination is not allowed" on 26 Oct. The old connection can be deleted. |
| New official API | `https://api.get-energy-performance-data.communities.gov.uk/api/domestic/search` (beta). Free; register at get-energy-performance-data.communities.gov.uk and use the bearer token from "My account". Parameters: `postcode`, `address`, `uprn`, and others. Returns `certificateNumber`, `addressLine1-4`, `postcode`, `postTown`, `currentEnergyEfficiencyBand`, `registrationDate`, `uprn`. It returns 404 when there is no certificate. There is no expiry field: an EPC is valid for 10 years from `registrationDate`. Rate limit: 6000 requests per 5 minutes. |
| GOV.UK search page (`find-energy-certificate.service.gov.uk/find-a-certificate/search-by-postcode?postcode=CB9+8LT`) | **Works from Make with no key** (HTTP 200). Each result row has `href="/energy-certificate/<certificate number>"`, the address text, the rating, "Valid until", and an "Expired" tag when it applies. 14 results for CB9 8LT, matching the video. |
| Certificate page | `https://find-energy-certificate.service.gov.uk/energy-certificate/<certificate number>`, with a print view at `?print=true` (HTTP 200, about 38 KB of HTML, 8 inline SVGs for the rating graphs, no images). GOV.UK offers **no PDF download**, only browser print. |
| URL to PDF in Make | Native Make apps include CloudConvert **"Capture a Website"** (URL to PDF, returns the file), PDF.co, pdf4me "Convert URL to PDF", and APITemplate.io. All of them render with a real browser, so the inline SVG rating graphs appear the same as in the video. |
| Free alternative (HTML to Google Doc to PDF via Drive) | Possible with the existing Drive connection, but Google Docs import drops inline SVG. The rating graphs, which are the most important part, would be missing. **Not recommended.** |

## Recommended design: new scenario "Deal Staker - AUC-110", run in parallel from AUC-102

1. Webhook `deal-staker-auc-110` (with `x-make-apikey`). AUC-102 sends `opportunity_id`,
   `opportunity_name`, `postcode` and `deal_folder_id`, only when a new deal folder is created.
2. HTTP GET to the new EPC API: `/api/domestic/search?postcode=<postcode>`, using a Bearer token
   connection.
3. Pick the certificate:
   - `addressLine1` (plus line 2) contains the house number or name from the opportunity name;
   - the postcode is an exact match;
   - if several remain, take the newest `registrationDate`.
   - Flats ("Flat 2, 13 …") need the flat number to match too. If no single match is found, skip
     the PDF (do not guess) and optionally post a note.
4. CloudConvert "Capture a Website":
   - `url`: `https://find-energy-certificate.service.gov.uk/energy-certificate/<certificateNumber>?print=true`
   - `output_format`: `pdf`
   - `downloadFile`: yes
5. Google Drive "Upload a File" into the deal folder as `EPC - <opportunity name>.pdf`.

Fallback for step 2: parse the GOV.UK search page with a regex on
`/energy-certificate/(\d{4}-\d{4}-\d{4}-\d{4}-\d{4})">\s*([^<]+)`. It needs no key, but it breaks if
GOV.UK changes its HTML.

## Needed from the business

1. A free account on get-energy-performance-data.communities.gov.uk. Then create a Make "API Key"
   connection:
   - header `Authorization`, value `Bearer <token>`
   - origin `https://api.get-energy-performance-data.communities.gov.uk`
2. A CloudConvert account, plus a Make CloudConvert connection. Alternatively PDF.co or pdf4me.
3. Decide whether to include **expired** certificates. The ticket says "if this is available". The
   suggested default is to save the newest certificate even if it has expired, and mark it
   "(expired)" in the file name.
4. Confirm the file name: `EPC - <opportunity name>.pdf` in the deal folder.
