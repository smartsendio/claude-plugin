---
name: smartsend
description: Use this skill when booking, tracking, or managing shipments through Smart Send. Triggers on requests like "book a shipment", "send this parcel", "ship to <address>", "find a pickup point", "where is order X", "reprint label", "validate these addresses", or any other shipping workflow that involves the Smart Send MCP tools. Covers booking, address validation, service-point selection, templates, tracking lookups, and document retrieval. Do not use for general programming work or for non-Smart-Send shipping platforms.
---

# Smart Send

This skill teaches you how to book, track and manage shipments through the Smart Send MCP server.

## How Smart Send works (read this first)

Smart Send is a shipping aggregator. The user (a webshop, office or production company) has agreements with one or more carriers — PostNord, GLS, Bring, Burd, Budbee — and Smart Send is the layer in between.

Smart Send's core philosophy is that **Smart Send handles the complexity, not the client**. Booking requests look simple from the outside because the complexity is dealt with on Smart Send's side. As a consequence, always include as much data as you have: detailed item descriptions, values, weights, customs information, duty and VAT amounts. The more data Smart Send receives, the better it can handle carrier requirements, customs declarations, and routing decisions on the user's behalf. Never skip optional fields to "keep things simple" — if the data is available, include it.

Three principles matter for everything below:

1. **Every booking request has the same shape.** Sender, receiver, parcels, items, customs info — the *data structure is identical* whether the shipment goes via PostNord home delivery, GLS parcel shop or Bring international. Only three things actually change between carriers:
   - `carrier_code` (e.g. `postnord`, `gls`, `bring`)
   - `service_code` / method (e.g. home delivery, service point)
   - `addons` (e.g. signature, flex delivery, age verification)

   You never need to "learn a new format" per carrier — pick a carrier+service+addons and fill in the same payload.

2. **"Carrier" means agreement carrier.** The carrier you book with is the *agreement* carrier — not necessarily the carrier that physically handles every leg. A PostNord booking from Denmark to Norway might be picked up by PostNord DK, cross the border, and be delivered by Bring in Norway. The booking and tracking remain with PostNord.

3. **"Booking" is a real action with side effects that differ per service.** When you call `book-shipment`, Smart Send transmits the shipment electronically to the carrier, the carrier accepts it, and Smart Send returns unique **tracking numbers for each parcel** plus signed URLs to the shipping documents (label PDFs, customs invoices). Beyond that, the carrier-side effects depend on the chosen service *and* the team's configuration — booking may, for example, request a pickup from the carrier. Usually it does not, and in some cases a booking can later be voided, but **assume in general that a booked shipment cannot be cancelled**. Always confirm with the user before booking.

## Routes vs rates

Two different levels of shipping-option lookup:

- **Routes** (`smartsend://routes` resource): general, static reference — "PostNord offers service X from DK to SE." Does not consider team configuration, pricing, or parcel specifics. Use this for orientation: mapping carrier names to codes, listing what's available in principle.
- **Rates** (`find-delivery-options` tool): live, dynamic lookup. Filters by the team's configuration (excluded services, carrier preferences), calculates prices from the team's pricing setup, estimates delivery windows, considers parcel dimensions and weight. Use this whenever you need to present actionable options to the user.

Always use rates when the answer needs to be actionable. Use routes only for general reference.

## The standard flow

Most shipping tasks follow the same five steps. Skip any step that doesn't apply.

1. **Discover what the team can ship.** Read the `smartsend://routes` resource if you need to map carrier names to codes or list available services/addons. Do this once per session, not per shipment.

2. **Validate addresses.** For anything beyond a single hand-typed address, call `validate-address` first. Catching a bad postal code here is free; catching it after booking costs the carrier's surcharge.

3. **Find delivery options for the specific shipment.** Call `find-delivery-options` with the parties (sender + receiver) and parcels (weight, dimensions). It returns only the services/addons that are actually valid for this shipment given the team's carrier accounts. Present these to the user, ideally ranked, and let them pick.

4. **(If the user picked a service-point service)** Call `search-service-points` near the receiver address and let the user choose one. Pass the chosen service point's identifier as `parties.pickup` when booking. If no specific service point is provided, Smart Send routes to the nearest service point to the receiver's address.

5. **Book.** Call `book-shipment` with the same parties + parcels, plus the chosen `carrier_code`, `service_code` and `addons`. The response contains the tracking numbers and document URLs. Surface both to the user.

## Templates

When the user ships the same kind of parcel repeatedly (e.g. a webshop with a standard 1 kg box on GLS service point), use `create-shipment-template` to save the recurring bits — carrier, service, addons, default parcel dimensions, default item details. Future bookings only need to provide the receiver and any per-shipment overrides.

List templates with `smartsend://shipment-templates`; fetch a single one by URI if you need its full contents.

## Common scenarios

### Single shipment ("ship this parcel to Anna")

1. Validate the address.
2. `find-delivery-options` with one parcel.
3. Show the user the cheapest/fastest 2–3 options. If the user has a clear default, pick it.
4. Confirm explicitly: *"I'm about to book a 2 kg parcel with PostNord home delivery to Anna at <address> — confirm?"*
5. `book-shipment`. Surface tracking number + label URL.

### Bulk booking from a list

1. Parse the list into structured shipments (one per row).
2. Run all addresses through `validate-address` in batches of up to 100. Stop and ask the user how to handle invalid ones — never fabricate fixes.
3. If a template fits, use it; otherwise resolve service per shipment with `find-delivery-options`.
4. Summarise: *"<N> shipments ready: <N1> PostNord home, <N2> GLS service point. Total estimated cost: <X>. Book all?"*
5. On confirmation, book in one `book-shipment` call (it accepts multiple shipments).

### International / customs shipments

International shipments need customs data on the receiver and on each item:
- **Incoterms** (DAP, DDP, ...) determine who pays customs duties. DAP = receiver pays, DDP = sender pays.
- **Receiver identifiers**: `vat`, `eori`, `gb_eori` (UK), `voec` (Norway) — whichever the destination requires.
- **Items**: `sku`, `description`, `quantity`, value (in minor units), currency, origin country, and HS/tariff codes when available.
- **Content type**: commercial goods, returned goods, gift, documents, etc.

`find-delivery-options` will filter out services the receiver country isn't eligible for. If required customs fields are missing, ask the user before booking — the carrier will reject the shipment otherwise.

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

- **Never fabricate addresses, names, emails or phone numbers.** The Smart Send tools say so explicitly in their schemas. Missing data → ask the user or omit the field; do not guess.
- **Always confirm before `book-shipment`.** Booking transmits to the carrier, generates tracking numbers, and may trigger further side effects depending on the service and team config (e.g. requesting a pickup). Some bookings can be voided afterwards, but treat cancellation as *not guaranteed* and get explicit user confirmation up front.
- **Use ISO 3166-1 alpha-2 country codes** (`DK`, `SE`, `NO`, `FI`, `DE`, ...). Smart Send rejects everything else.
- **Weight in kg, dimensions in cm** by default. Override only if the user explicitly works in other units.
- **Cost and delivery window are nullable.** `find-delivery-options` returns `null` for either when the team hasn't uploaded rate tables or when no standard delivery window is known. Don't invent values — say "estimate not available" instead.
- **One team at a time.** All Smart Send data is team-scoped. If the user mentions a different team, ask them to switch before continuing.
