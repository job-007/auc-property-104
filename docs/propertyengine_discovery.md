# PropertyEngine MCP discovery

Run on 29 Oct 2026 through the claude.ai PropertyEngine connector, against
https://propertyengine.co.uk/property/VsBsijE3n (15 Filbert Close, Gloucester, GL4 5EX).

## Answers to the spec's open questions

| Question | Answer |
|---|---|
| Plan / workspace | The account's default workspace is not on a plan with the connector. Every call must pass `workspaceId: "4qM_6RG0D5"` ("Anastasiya Chakuryna's Workspace"). |
| Do tools accept a PropertyEngine URL or ID? | Yes. The ID is the last segment of the PropertyEngine URL and goes straight into `get_property` / `get_property_comparables`. `lookup_property` takes a Rightmove or Zoopla link (`portalLink`), not an address. Address-only lookups go through `search_properties` with `areas: [{"area": "<postcode>"}]`. |
| Where do photo URLs come from? | `get_property` → `photos[]`: `{ index (0-based), room (EXTERIOR / KITCHEN / BATHROOM / LIVING / UNKNOWN), url }`. 20 photos for the test property. `get_property_photos` returns images for the model to look at, at most 8, and is not useful for Make. |
| Floor plans? | Separate: `get_property` → `floorplans[]`, a list of URLs (one PNG for the test property). |
| URL hosts | Photos: `https://media.propertyengine.co.uk/photo/<id>/<index>/small/<key>`. Floor plans: `https://media.propertyengine.co.uk/<hash>.png`. |
| Are the URLs public / do they expire? | **Not yet confirmed.** This cloud session's network policy blocks `media.propertyengine.co.uk`. The URLs contain no signature or expiry parameter, which suggests they are public. To confirm, open one in a private browser window. Also try `large` or `medium` in place of `small` for a bigger image. |
| Comparables parameters | `radiusMeters` (200–5000, default 1609 = 1 mile), `soldSince` (default `now-24M`), `include` (`sold` / `forSale` / `rental`), `limitPerSection` (max 25, default 15 sold), `propertyTypes`, `minBedrooms`/`maxBedrooms`, floor area and condition filters. Results are deliberately not filtered to the subject's type or bedrooms. |
| Comparable fields | `distanceMeters`, `address`, `postcode`, `price`, `dateOfTransfer`, `propertyType`, `bedrooms`, `tenure`, `totalFloorAreaSqm`, `pricePerSqm`, `conditionScore`, `requiresModernisation`, `recentlyModernised`, `wentUnderOfferDate`, `propertyEngineUrl`. Bedrooms and floor area are `null` for some rows. |
| OAuth / refresh tokens | Handled by claude.ai for the connector; nothing to manage on our side. |
| Rate limit | Not documented; not hit during discovery. |

## `get_property` fields used

`id`, `propertyEngineUrl`, `formattedAddress`, `postcode`, `askingPrice`, `askingPriceQualifier`,
`bedrooms`, `bathrooms`, `addressType`, `tenure`, `totalFloorAreaSqm`, `conditionScore`,
`addedDate`, `soldSTCLetAgreed`, `removed`, `lastSoldPrice`, `lastSoldPriceDate`, `description`
(HTML), `epc.currentEnergyRating`, `epc.potentialEnergyRating`, `agent.name`,
`agent.phoneNumber`, `rightmoveLink`, `zooplaLink`, `askingPriceVsArea.averageSoldPrice`,
`photos[]`, `floorplans[]`.

Not provided: receptions, council tax band (sometimes in the description text), floor area in
sqft (computed from sqm).

`get_property` also returns `listingHistory[]` (Land Registry sales, previous listings, price
changes) and `planningApplications[]`, which are not passed to Make yet.
