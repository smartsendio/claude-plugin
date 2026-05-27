---
name: smartsend
description: Use this skill when booking, tracking, or managing shipments through Smart Send. Triggers on requests like "book a shipment", "send this parcel", "ship to <address>", "find a pickup point", "where is order X", or "reprint label". Covers booking, service-point selection, templates, tracking lookups, and document retrieval. Do not use for general programming work or for non-Smart-Send shipping platforms.
---

# Smart Send

This skill teaches you how to book, track and manage shipments through the Smart Send MCP server.

## How Smart Send works (read this first)

Smart Send is a shipping aggregator. The user (a webshop, office or production company) has agreements with one or more carriers — PostNord, GLS, Bring, Burd, Budbee — and Smart Send is the layer in between.

Smart Send's core philosophy is that **Smart Send handles the complexity, not the client**. Booking requests look simple from the outside because the complexity is dealt with on Smart Send's side. As a consequence, always include as much data as you have: detailed item descriptions, values, weights, customs information, duty and VAT amounts. The more data Smart Send receives, the better it can handle carrier requirements, customs declarations, and routing decisions on the user's behalf. Never skip optional fields to "keep things simple" — if the data is available, include it.

Two principles matter for everything below:

1. **"Carrier" means agreement carrier.** The carrier you book with is the *agreement* carrier — not necessarily the carrier that physically handles every leg. A PostNord booking from Denmark to Norway might be picked up by PostNord DK, cross the border, and be delivered by Bring in Norway. The booking and tracking remain with PostNord.

2. **"Booking" is a real action with side effects that differ per service.** When you call `book-shipment`, Smart Send transmits the shipment electronically to the carrier, the carrier accepts it, and Smart Send returns unique **tracking numbers for each parcel** plus signed URLs to the shipping documents (label PDFs, customs invoices). Beyond that, the carrier-side effects depend on the chosen service *and* the team's configuration — booking may, for example, request a pickup from the carrier. Usually it does not, and in some cases a booking can later be voided, but **assume in general that a booked shipment cannot be cancelled**. Always confirm with the user before booking.

## Booking flow — decision tree

Pick one of three paths. Do not run extra preflight checks "to be safe" — they cost tool calls and don't change the outcome.

### Path A — service is already known → book directly

If the user (or a template, or a previous tool call) gives you a `carrier_code` and `service_code`, go straight to `book-shipment`. No `find-delivery-options`, no `validate-address`.

`book-shipment` runs the same address and routing validation internally. Calling those tools first is duplicate work.

### Path B — service is unknown → `find-delivery-options` then `book-shipment`

When the user hasn't named a service:

1. Call `find-delivery-options` with the receiver address and parcel details. It returns the services this team can actually book for this shipment, with `carrier_code`, `service_code`, optional addons, estimated cost, and estimated delivery days.
2. Present 2–3 options ranked by relevance. Highlight a recommendation if the user signalled a preference (cheapest, fastest, sustainable).
3. On confirmation, call `book-shipment` reusing the same `parties.receiver.address` block and copying `carrier_code` / `service_code` straight from the chosen option.

### Path C — bulk address cleanup (rare)

Only use `validate-address` for standalone batch jobs — e.g. the user pastes a list of addresses and asks "are any of these wrong?". Never as a preflight before `find-delivery-options` or `book-shipment`; both already validate.

## Field-name cheat sheet

The Smart Send tools share a vocabulary. Use these exact names — do not invent synonyms from other shipping APIs:

| Concept | Field name | Notes |
|---|---|---|
| Address street | `address_lines` | **Array** of 1–2 strings. Not `street`, not `street1`. |
| Postal/ZIP | `postal_code` | Not `zip_code`. |
| Country | `country` | ISO 3166-1 alpha-2 (`DK`, `SE`, `NO`, …). Not `country_code`. |
| Person name | `name_lines` | **Array** of 1–2 strings. Not `name`. |
| Carrier | `carrier_code` | Same name on input and output across all tools. |
| Service | `service_code` | Same name on input and output across all tools. |

## Canonical booking payload

Copy this shape; replace values with real data. Omit any field you don't have — never fabricate.

```json
{
  "shipments": [{
    "reference": "ORDER-1234",
    "delivery": {
      "carrier_code": "postnord",
      "service_code": "home"
    },
    "parties": {
      "receiver": {
        "address": {
          "address_lines": ["Vesterlundvej 16"],
          "postal_code": "6830",
          "city": "Nørre Nebel",
          "country": "DK"
        },
        "contact": {
          "name_lines": ["Jeremias Wolff"],
          "email": "jeremias@example.com",
          "phone": "+4512345678"
        }
      }
    },
    "parcels": [{ "gross_weight": 0.5 }]
  }]
}
```

For service-point services, add `parties.pickup.service_point_code` (from `search-service-points`). For international shipments, see "Customs" below.

## Routes resource

The `smartsend://routes` resource is general, static reference — "PostNord offers service X from DK to SE." Read it at most once per session if you need to map carrier names to codes or list what's available in principle. Do **not** read it before every booking — `find-delivery-options` already filters to what this team can actually use.

## Templates

When the user ships the same kind of parcel repeatedly (e.g. a webshop with a standard 1 kg box on GLS service point), use `create-shipment-template` to save the recurring bits — carrier, service, addons, default parcel dimensions, default item details. Future bookings only need to provide the receiver and any per-shipment overrides.

List templates with `smartsend://shipment-templates`; fetch a single one by URI if you need its full contents.

## Common scenarios

### Single shipment ("ship this parcel to Anna")

1. `find-delivery-options` with the receiver address and one parcel.
2. Show the user the cheapest/fastest 2–3 options.
3. Confirm: *"I'm about to book a 2 kg parcel with PostNord home delivery to Anna at <address> — confirm?"*
4. `book-shipment`. Surface tracking number + label URL.

### Service-point delivery

If the chosen service has `is_pickup: true`:

- Default: omit `parties.pickup` — Smart Send routes to the nearest service point to the receiver.
- If the user wants to pick: call `search-service-points` with `carrier_code` + receiver address, present the closest 3–5, then pass the chosen `code` as `parties.pickup.service_point_code` when booking.

### Bulk booking from a list

1. Parse the list into structured shipments (one per row).
2. If a template fits, use it; otherwise resolve service per shipment with `find-delivery-options`.
3. Summarise: *"<N> shipments ready: <N1> PostNord home, <N2> GLS service point. Total estimated cost: <X>. Book all?"*
4. On confirmation, book in one `book-shipment` call (it accepts up to 100 shipments).

### International / customs shipments

International shipments need customs data on the receiver and on each item:
- **Incoterms** (DAP, DDP, …) determine who pays customs duties. DAP = receiver pays, DDP = sender pays.
- **Receiver identifiers**: `vat`, `eori`, `gb_eori` (UK), `voec` (Norway) — whichever the destination requires.
- **Items**: `sku`, `description`, `quantity`, value (in minor units), currency, origin country, and HS/tariff codes when available.
- **Content type**: commercial goods, returned goods, gift, documents, etc.

`find-delivery-options` filters out services the receiver country isn't eligible for. If required customs fields are missing, ask the user before booking — the carrier will reject the shipment otherwise.

### "Where is my order?"

1. `search-shipments` by the user's reference (order number, receiver email, or tracking number).
2. Read the matching shipment resource for full context (parties, parcels, current status).
3. Read the `tracking` resource per parcel for the event timeline.
4. Summarise in plain language: where the parcel is, what the next expected event is, and whether anything looks stuck. Present events in reverse chronological order (most recent first) with clear timestamps and locations. Use the `track-shipment` MCP prompt if the situation needs diagnosis.

### Reprint a label

1. `search-shipments` to find the shipment.
2. Read the `shipping-documents` resource for that shipment — it returns fresh signed URLs.
3. Hand the URL to the user (or trigger the team's auto-print setup, if relevant).

## Response formatting

When presenting delivery options, format them as a clear comparison with price, delivery time, and service type. Highlight the recommended option if the user has expressed a preference for speed, cost, or sustainability.

When reporting booking results, always include the tracking number and reference so the user can match results to their orders.

## Guardrails

- **Never fabricate addresses, names, emails or phone numbers.** Missing data → ask the user or omit the field; do not guess.
- **Always confirm before `book-shipment`.** Booking transmits to the carrier, generates tracking numbers, and may trigger further side effects depending on the service and team config (e.g. requesting a pickup). Some bookings can be voided afterwards, but treat cancellation as *not guaranteed* and get explicit user confirmation up front.
- **Use ISO 3166-1 alpha-2 country codes** (`DK`, `SE`, `NO`, `FI`, `DE`, …). Smart Send rejects everything else.
- **Weight in kg, dimensions in cm** by default. Override only if the user explicitly works in other units.
- **Cost and delivery window are nullable.** `find-delivery-options` returns `null` for either when the team hasn't uploaded rate tables or when no standard delivery window is known. Don't invent values — say "estimate not available" instead.
- **One team at a time.** All Smart Send data is team-scoped. If the user mentions a different team, ask them to switch before continuing.
