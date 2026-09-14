# Send Orders Workflow

An [n8n](https://n8n.io) workflow that pulls orders from an API, normalises and prices them, logs each one to Google Sheets, renders a branded HTML order confirmation, converts it to PDF with Gotenberg, and emails it to the customer via Gmail — one order at a time.

![Workflow overview](docs/workflow.png)

---

## What it does

| # | Node | Type | Role |
|---|------|------|------|
| 01 | `When clicking 'Execute workflow'` | Manual Trigger | Starts the run manually |
| 02 | `02 - Get Orders` | HTTP Request | `GET https://dummyjson.com/carts` — demo order source |
| 03 | `03 - Split Orders` | Split Out | Splits the `carts` array into one item per order |
| — | `Loop Over Items` | Split In Batches | Processes orders **one at a time** (the `loop` output feeds node 04; `Send a message` loops back) |
| 04 | `04 - Normalize Order` | Set | Keeps only the fields that matter and adds `currency: EUR`, `status: NEW` |
| 05 | `05 - Calculate Order` | Code | Computes `subtotal`, `discount`, `VAT` (19 %) and `finalTotal` |
| 06 | `Append row in sheet` | Google Sheets | Appends the order as a row to the tracking spreadsheet |
| 07 | `Code in JavaScript` | Code | Builds a full standalone HTML invoice/order confirmation |
| 08 | `Convert to File` | Convert to File | Turns `invoice_html` into a binary file named `index.html` |
| 09 | `HTTP Request` | HTTP Request | `POST` to Gotenberg (`/forms/chromium/convert/html`) → PDF binary |
| 10 | `Send a message` | Gmail | Emails the order confirmation with the PDF attached, then returns to the loop |

### Flow

```
Manual Trigger
   └─> Get Orders (dummyjson /carts)
         └─> Split Orders (carts[])
               └─> Loop Over Items ──loop──> Normalize ─> Calculate ─> Google Sheets
                          ▲                                                 │
                          │                                                 ▼
                          │                                     Generate invoice HTML
                          │                                                 │
                          │                                                 ▼
                          │                                          Convert to File
                          │                                                 │
                          │                                                 ▼
                          │                                      Gotenberg → PDF
                          │                                                 │
                          └────────────────── Gmail send ◄──────────────────┘
```

The `done` branch of `Loop Over Items` is intentionally empty — the workflow simply ends once every order has been processed.

---

## Pricing logic (node 05)

```js
const subtotal        = $json.total;             // pre-discount amount from the API
const discountedTotal = $json.discountedTotal;

const discount   = subtotal - discountedTotal;
const VAT        = discountedTotal * 0.19;       // 19 % VAT, hardcoded
const finalTotal = discountedTotal + VAT;
```

All four values are merged into the item alongside the normalised fields.

---

## Requirements

- **n8n** (self-hosted or cloud) — the workflow uses `executionOrder: v1` and `binaryMode: separate`.
- **Gotenberg** reachable at `http://host.docker.internal:3000` for HTML → PDF conversion:
  ```bash
  docker run --rm -p 3000:3000 gotenberg/gotenberg:8
  ```
  If n8n is not running in Docker, change the URL in the `HTTP Request` node to `http://localhost:3000`.
- **Google Sheets OAuth2 credential** with write access to the target spreadsheet.
- **Gmail OAuth2 credential** for sending.

---

## Setup

1. **Import** — in n8n: *Workflows → Import from File* → select `Send orders workflow.json`.
2. **Reconnect credentials** — the exported credential IDs (`Google Sheets account`, `Gmail account`) will not exist in your instance. Open the `Append row in sheet` and `Send a message` nodes and re-select your own credentials.
3. **Point at your spreadsheet** — `Append row in sheet` targets the demo sheet `AI Order Management Demo` (`gid=0`). Replace it with your own document and sheet. Expected header row:

   ```
   Order ID | Customer ID | Products | Quantity | Subtotal | Discount | VAT | Total | Currency | Status | Created At
   ```

4. **Start Gotenberg** (see above) and confirm the URL in the `HTTP Request` node.
5. **Set the recipient** — `Send a message` currently sends to a hardcoded address (`achref.tirari.97@gmail.com`). Change it to the real customer address, e.g. an expression resolving the customer's email.
6. **Run** — click *Execute workflow*.

---

## Customising

| Want to change… | Where |
|---|---|
| Order source | `02 - Get Orders` URL (swap dummyjson for your real orders endpoint) |
| Fields carried through | `04 - Normalize Order` assignments |
| VAT rate / currency | `05 - Calculate Order` (`0.19`) and `04 - Normalize Order` (`EUR`) |
| Invoice look & company details | `Code in JavaScript` — the HTML/CSS template and the `YOUR COMPANY` block |
| Sheet columns | `Append row in sheet` column mapping |
| Email subject / body | `Send a message` |

---

## Known rough edges

These are real issues in the current export — worth fixing before using it for anything beyond a demo.

- **The invoice template reads field names the pipeline never produces.** Node 07 expects `order_id`, `customer_id`, `subtotal`, `vat`, `total_quantity`, `products`, while nodes 04/05 produce `id`, `userId`, `subtotal`, `VAT`, `totalQuantity`. Result: the customer ID renders as `N/A`, VAT and quantity render as `0.00`/`0`, and the product table falls back to the "Order contains 0 item(s)" placeholder. Align the names in either the Set node or the template.
- **The Sheets `Total` column receives `$json.total`**, which is the *pre-VAT, pre-discount* amount from the API — not `finalTotal`. Likewise `Discount` receives `discountedTotal` (the discounted amount) rather than `discount` (the saving). Remap if you want the sheet to match the computed figures.
- **The Gmail attachment binding is empty** (`attachmentsBinary: [{}]`). Set the binary property name (the one produced by the Gotenberg `HTTP Request` node, normally `data`) or the email will go out without the PDF.
- **`totalQuantity` is assigned twice** in `04 - Normalize Order` — harmless, but the duplicate can be removed.
- **The recipient address is hardcoded**, so every order confirmation goes to the same mailbox.
- The email body template has no line breaks between sentences (`…Currency }}Best regards,`), so it renders as one run-on paragraph.

---

## Files

- `Send orders workflow.json` — the exported n8n workflow (importable as-is).
- `docs/workflow.png` — canvas screenshot used above.
