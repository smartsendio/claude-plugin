# Smart Send for Claude

**Who is this for?** This plugin is for small and medium-sized businesses that ship parcels — webshops, offices sending letters and packages, and small production companies handling fulfilment in-house. If you currently book your shipments by hand on carrier portals or your shop admin, Smart Send for Claude lets you do the same work by talking to Claude.

Smart Send is a shipping platform that connects you to carriers like PostNord, GLS, Bring, Burd and Budbee. This Claude plugin wires Claude directly into your Smart Send account so it can find delivery options, book shipments, look up service points, troubleshoot deliveries and keep templates tidy — all from a chat.

## What you can ask Claude to do

Once installed, Claude can act as a shipping assistant inside your daily workflow. Some examples:

- *"Book a 2 kg parcel to this customer with PostNord home delivery."*
- *"Find me the three closest GLS parcel shops to postal code 8000 in Denmark."*
- *"Where is order #1042? It was supposed to arrive yesterday."*
- *"Validate these 40 addresses from my CSV before I import them."*
- *"Make a shipment template for our standard 1 kg box with GLS."*
- *"Print labels for every order that came in this morning."*

## How Smart Send books shipments

Two things are worth understanding before you start:

- **Every booking request has the same shape.** Sender, receiver, parcels, items, customs info — the data structure is identical regardless of carrier. The only things that change per shipment are the `carrier_code`, the `service_code` (the carrier's specific delivery method) and the optional `addons` (signature, flex delivery, age verification, etc.). That means once you've described a shipment once, switching it from PostNord home delivery to GLS service point is a three-field change.
- **Booking is a real action with real side effects, and the side effects differ per service.** When the plugin books a shipment, Smart Send registers it electronically with the carrier and the carrier generates a unique tracking number for each parcel. Shipping documents (labels, customs invoices) are created at the same time and are available as signed download URLs. Depending on the chosen service *and* the team's configuration, booking may also trigger additional carrier-side effects — for example, a pickup being requested from the carrier. Most of the time no pickup is requested, and some bookings can later be voided or simply ignored, but there is no general guarantee that a booked shipment can be cancelled. Treat every booking as final and confirm with the user first.

## Requirements

Before installing the plugin, make sure you have:

1. **A Smart Send account.** If you don't have one yet, sign up at [smartsend.io](https://www.smartsend.io/dashboard). The free plan is enough to try the plugin.
2. **Connected carrier accounts.** Smart Send needs to know which carriers you have agreements with (PostNord, GLS, Bring, Burd, Budbee, etc.). Follow the in-app [onboarding guide](https://docs.smartsend.io/) to connect your carriers — the plugin can only book shipments with carriers you have already configured.
3. **A Smart Send team.** All shipments belong to a team. If you're part of more than one team, make sure your active team is the one you want Claude to work in.

The plugin authenticates with Smart Send via OAuth — the first time you use a tool, Claude will ask you to sign in and grant access. No API keys to copy around.

## Installation

You need [Claude Code](https://code.claude.com/docs/en/setup) installed and authenticated.

From inside Claude Code, run:

```text
/plugin marketplace add smartsendio/claude-plugin
/plugin install smartsend@smartsend
/reload-plugins
```

What each step does:

1. `/plugin marketplace add smartsendio/claude-plugin` — registers this repository as a marketplace so Claude Code can see the plugin. No plugin is installed yet.
2. `/plugin install smartsend@smartsend` — installs the `smartsend` plugin from the `smartsend` marketplace into your user scope (available across all your projects). To pick a different scope (project or local), run `/plugin` instead, open the **Discover** tab and select **smartsend** there.
3. `/reload-plugins` — activates the plugin in your current session without a restart.

On first use, Claude will prompt you to sign in to Smart Send via OAuth — log in with your existing account and approve access. No API keys to configure.

To update later, run `/plugin marketplace update smartsend` followed by `/reload-plugins`. To remove the plugin, run `/plugin uninstall smartsend@smartsend`.

See Claude's [plugin discovery guide](https://code.claude.com/docs/en/discover-plugins) for the full reference.

## What's in the plugin

The plugin bundles two things:

1. **An MCP connection** to the hosted Smart Send MCP server. This gives Claude tools, resources and prompts for talking to your Smart Send account.
2. **A skill** that teaches Claude *how* to use those tools well for everyday shipping work.

### MCP tools

Actions Claude can take on your behalf:

| Tool | What it does |
| --- | --- |
| `find-delivery-options` | Given a shipment (parties + parcels), returns the carriers, services and add-ons your team can actually use, with estimated delivery windows where available. |
| `book-shipment` | Books one or more shipments with the selected carrier. Generates tracking numbers and shipping documents (and prints them, if your team is configured to auto-print). |
| `create-shipment-template` | Saves a reusable template with pre-filled carrier, service, add-ons, parcel dimensions and item details — great for recurring shipments. |
| `search-service-points` | Finds nearby pickup points, parcel shops and lockers for a given carrier and address. |
| `validate-address` | Validates up to 100 addresses against the destination country's rules. Catches bad postal codes before they reach the carrier. |
| `search-shipments` | Looks up previous shipments by reference, receiver email, tracking number, date range or status. |

### MCP resources

Read-only data Claude can pull into context:

| Resource | URI | What it provides |
| --- | --- | --- |
| Who Am I | `smartsend://who-am-i` | Details about the authenticated user and the active Smart Send team. |
| Routes | `smartsend://routes` | All carriers, services and add-ons available to your team — used to map carrier names to codes and discover what you can ship. |
| Service Point | `smartsend://service-points/{carrier}/{country}/{code}` | Full details of a specific pickup point: type, address, coordinates, opening hours. |
| Shipment | `smartsend://shipments/{shipmentUuid}` | Complete shipment data including parcels, tracking and documents. |
| Shipping Documents | `smartsend://shipments/{shipmentUuid}/documents` | Signed URLs to download labels, customs invoices and other documents for one shipment. |
| Tracking | `smartsend://tracking/{trackingCode}` | Current tracking status and event history for one parcel. |
| Shipment Templates | `smartsend://shipment-templates` | List of all saved shipment templates. |
| Shipment Template Detail | `smartsend://shipment-templates/{shipmentTemplateUuid}` | Full content of a specific template. |

### MCP prompts

Guided multi-step workflows you can start with a slash command:

| Prompt | What it walks you through |
| --- | --- |
| `book-shipment` | Collect the address, define the parcels, select the service, handle customs if needed, and complete the booking. |
| `troubleshoot-delivery` | Investigate a shipment's tracking events, compare expected vs. actual timeline, and identify the parcels that are stuck or late. |

### Skill

| Skill | What it covers |
| --- | --- |
| `smartsend` | The whole shipping lifecycle: discovering what the team can ship, validating addresses, finding delivery options for a specific parcel, picking service points, booking (single or bulk), saving and reusing templates, looking up tracking, and reprinting documents. Encodes the Smart Send specifics — generic booking payload, confirm-before-booking, never-fabricate-data — so Claude behaves consistently regardless of carrier. |

## Getting help

- **Smart Send docs**: [docs.smartsend.io](https://docs.smartsend.io/)
- **Sign up / dashboard**: [smartsend.io](https://www.smartsend.io/dashboard)
- **Carrier onboarding**: see the *Carriers* section of the dashboard after signing in.

If a shipping action fails, Claude will surface the carrier's error message — most issues come from missing receiver details or a carrier account that hasn't been fully connected yet.

## License

MIT
