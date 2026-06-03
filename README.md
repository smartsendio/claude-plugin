# Smart Send for Claude

A Claude Code plugin that gives Claude a comprehensive reference for managing shipping via Smart Send.

Smart Send is a shipping platform that connects you to carriers like PostNord, GLS, Bring, Burd and Budbee. This Claude plugin wires Claude directly into your Smart Send account so it can find delivery options, book shipments, look up service points, troubleshoot deliveries and keep templates tidy — all from a chat.

## Who is this intended for

Anyone interested in booking shipments or tracking parcels via AI.

## What you can ask Claude to do

Once installed, Claude can act as a shipping assistant inside your daily workflow. Some examples:

- *"Book a 2 kg parcel to this customer with PostNord home delivery."*
- *"Find me the three closest GLS parcel shops to postal code 8000 in Denmark."*
- *"Where is order #1042? It was supposed to arrive yesterday."*
- *"Validate these 40 addresses from my CSV before I import them."*
- *"Make a shipment template for our standard 1 kg box with GLS."*
- *"Print labels for every order that came in this morning."*

## Requirements

Before installing the plugin, make sure you have an active [Smart Send account](https://www.smartsend.io/dashboard).

## Installation

You need [Claude Code](https://code.claude.com/docs/en/setup) installed and authenticated.

From inside Claude Code, run:

```text
/plugin marketplace add smartsendio/claude-plugin
/plugin install smartsend@smartsend
/reload-plugins
```

On first use, Claude will prompt you to sign in to Smart Send via OAuth — log in with your existing account and approve access. No API keys to configure.

## What's in the plugin

The plugin bundles two things:

1. **An MCP connection** to the hosted Smart Send MCP server at `https://mcp.smartsend.io`. This gives Claude tools, resources and prompts for talking to your Smart Send account.
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
| `track-shipment` | Investigate a shipment's tracking events, compare expected vs. actual timeline, and identify the parcels that are stuck or late. |

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
