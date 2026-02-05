# Smart Send — Shipping Knowledge

This skill provides domain knowledge for working with the Smart Send shipping platform. Read this when handling any shipping, logistics, or parcel booking task that uses Smart Send tools.

## Smart Send Philosophy

Smart Send's core philosophy is that **Smart Send handles the complexity, not the client**. This is why booking requests may appear simple from the outside — the complexity is dealt with on Smart Send's side. As a consequence, you should always include as much data as possible in every request: detailed item descriptions, values, weights, customs information, duty and VAT amounts. The more data Smart Send receives, the better it can handle carrier requirements, customs declarations, and routing decisions on the user's behalf.

Never skip optional fields to "keep things simple." If the data is available, include it.

## Core Concepts

### Carriers

A **carrier** in Smart Send refers to the **agreement carrier** — the carrier with whom the booking is registered. This is not necessarily the carrier that physically handles every leg of the journey. The agreement carrier might:

- Use a different partner for pickup at the sender's location
- Hand the parcel over to another carrier at a border crossing
- Rely on a local carrier for last-mile delivery in the destination country

For example, a shipment booked with PostNord from Denmark to Norway might be picked up by PostNord DK, cross the border, and be delivered by Bring in Norway. The booking and tracking remain with PostNord (the agreement carrier), but the physical handling involves multiple parties.

Common carriers in Smart Send include PostNord, GLS, Bring, DAO, and others depending on the team's agreements.

### Services

A **service** is a specific shipping method offered by a carrier. Each carrier offers multiple services with different characteristics: delivery to door, delivery to service point, express delivery, economy, etc. Services have carrier-specific codes (e.g., `postnord:mypack_collect` for PostNord service point delivery).

### Addons

**Addons** are extra capabilities that can be enabled for a given service, often at additional cost. Examples include:

- Age verification at delivery
- Signature required on delivery
- Insurance for high-value shipments
- Flex delivery (receiver chooses delivery time)
- Notification preferences (SMS, email)

Not all addons are available for all services. The `find-delivery-options` tool returns which addons are available for each service option.

### Routes vs Rates

These are two different levels of shipping option lookup:

**Routes** (via the `smartsend://shipping-options` resource) represent the general availability of services from origin to destination. This is static reference data — "PostNord offers service X from DK to SE." Routes do not consider the team's configuration, pricing, or parcel specifics. Use routes for general orientation of what's possible.

**Rates** (via the `find-delivery-options` tool) are a live, dynamic lookup. Rates:

- Filter available options based on the team's configuration (excluded services, carrier preferences)
- Calculate prices based on the team's pricing setup (cost price, markups, custom rules)
- Estimate delivery windows based on when the parcel will be shipped (defaulting to now)
- Consider parcel dimensions and weight to exclude irrelevant services (e.g., letter services won't show for a bicycle)
- May include CO2 emission estimates in the future

Always use rates (`find-delivery-options`) when you need to present actionable options to the user. Use routes (`smartsend://shipping-options`) only for general reference.

### Service Points

Service points are physical pickup/drop-off locations — parcel shops (like a Netto or 7-Eleven), self-service lockers, or carrier depots. When a service delivers to a service point rather than the receiver's door, you need to help the user choose one. The `search-service-points` tool finds nearby options with opening hours, which is critical for the user's decision.

If a service point service is selected but no specific service point is provided in the booking, Smart Send will automatically route to the nearest service point to the receiver's address.

### Addresses and Validation

The `validate-address` tool checks addresses for format, postal code validity, and required fields per country. However, note that **booking itself also validates addresses** — so standalone validation is primarily useful for pre-checking addresses in bulk before a batch booking, or when the user wants to verify an address outside a booking flow.

### International Shipping

For cross-border shipments, additional data becomes important:

- **Incoterms** (DAP, DDP, etc.) determine who pays customs duties. DAP = receiver pays, DDP = sender pays.
- **Item details** are essential for customs declarations: description, quantity, value, origin country, tariff codes
- **Regulatory identifiers**: VAT numbers, EORI numbers (EU), VOEC numbers (Norway), GB EORI (UK)
- **Content type**: commercial goods, returned goods, gift, documents, etc.

Always include these when shipping internationally. Smart Send uses them to generate customs documentation automatically.

## Response Formatting Tips

When presenting delivery options, format them as a clear comparison with price, delivery time, and service type. Highlight the recommended option if the user has expressed a preference for speed, cost, or sustainability.

When reporting booking results, always include the tracking number and reference so the user can match results to their orders.

When showing tracking status, present events in reverse chronological order (most recent first) with clear timestamps and locations.
