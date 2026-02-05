---
description: Book multiple shipments in batch with Smart Send
---

# Book Multiple Shipments

You are helping the user book multiple shipments via Smart Send.

**Step 1 — Get shipment data:**
- If the user provided shipment details in their message, extract them
- If the user mentions a file (Excel, CSV, etc.), read it and extract the shipment data
- If no source is clear, ask the user how they'd like to provide the shipment details

Extract receiver information and any parcel details for each shipment. Where parcel details are missing, assume a single parcel with unknown dimensions and weight.

**Step 2 — Determine delivery options:**
- Check if individual shipments have a specified carrier/service. If so, use those for those shipments.
- For shipments without a specified delivery method (most likely all of them), use `find-delivery-options` with the first shipment's data to determine available options. Present these options to the user once and apply the chosen option to all shipments that don't have an individual specification.

**Step 3 — Confirm and book:**
Present a summary table of all shipments (receiver, service, parcel count) and ask for confirmation before calling `book-shipments`. Report results with tracking numbers. Flag any failures clearly so they can be retried.

Always include as much data as possible in each booking request — item descriptions, values, weights, duty information — Smart Send handles the complexity on its end and benefits from rich data.
