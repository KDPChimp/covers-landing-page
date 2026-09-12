# n8n → Airtable: create the order on payment

Base: **KDPChimp Database 2.0** `appuAp8Kb6iGqSejj`

| Table | ID |
|---|---|
| 01 Cart Purchases | `tblWxpI4G5juBEORv` |
| 02 Line Items | `tblpkBXw6eOkpjJuV` |
| 03 Orders | `tbldPl3mx8lFopILW` |
| 04 Customers | `tblo0Q0OlRP6bPLMS` |
| 06 Products | `tblhFhilJximUkDeN` |

The goal is that a Stripe or PayPal payment produces **exactly the same record
shape** WooCommerce produces today, so every downstream automation, SLA formula
and interface keeps working untouched.

---

## Workflow A — Stripe

**1. Trigger — Webhook**
`POST /webhook/covers-stripe`. In Stripe → Developers → Webhooks, send
`checkout.session.completed` to that URL. Verify the signature in the next node
(n8n has a Stripe Trigger node that does this for you — prefer it over a raw
webhook).

**2. Normalise** (Set node) — map into one shape both workflows share:

| Field | Stripe path |
|---|---|
| `email` | `customer_details.email` |
| `fullName` | `customer_details.name` |
| `amount` | `amount_total / 100` |
| `currency` | `currency` (uppercase it) |
| `paymentRef` | `id` (the `cs_...` session id) |
| `paymentMethod` | `"Stripe"` |
| `promo` | `total_details.breakdown.discounts[0].discount.coupon.name` |
| `discount` | `total_details.amount_discount / 100` |
| `bookTitle` | `custom_fields[0].text.value` |

**3. Find or create the customer** — Airtable *Search* on `04 Customers`,
formula `{cleanemail} = LOWER("{{$json.email}}")`.

- Found → keep `id`.
- Not found → Airtable *Create*: `Email`, `First Name`, `Last Name`
  (split `fullName` on the first space), `Customer Display Name`.

> Search on `cleanemail`, not `Email` — it's the normalised formula field and it
> stops `Jo@x.com` creating a duplicate of `jo@x.com`.

**4. Create the cart purchase** — `01 Cart Purchases`:

```
Customer ID          → [customerRecordId]
Cart Placed at       → now (ISO date)
Total Paid           → amount
Currency             → currency
Payment Method       → paymentMethod
Promo Code Used      → promo
Total Discount       → discount
WooCommerce ID       → paymentRef      ← reuse this column as the payment ref
Acquisition Source   → "Covers landing page"
```

**5. Create the order** — `03 Orders`:

```
Linked Cart Purchase ID → [cartRecordId]
Linked Customer ID      → [customerRecordId]
Linked Product ID       → [record id of "Professional Book Cover Design" in 06 Products]
Project Name            → bookTitle  (or "" — Cara fills it from the brief)
External Status         → your existing "new order" choice
Internal Status         → your existing "awaiting brief" choice
Created Date (Woo)      → now
```

> Look the product record id up once by name rather than hardcoding it, so a
> rename doesn't silently break the workflow. The SLA lookups
> (`SLA (Plan)`, `SLA (Design)`) resolve automatically off this link — that's
> why linking the product matters more than any other field here.

**6. Create the line item** — `02 Line Items`:

```
Link to Cart ID   → [cartRecordId]
Link to Orders ID → [orderRecordId]
Quantity          → 1
Woo Product Name  → "Professional Book Cover Design"
```

**7. Kick off your existing flow** — whatever WooCommerce currently triggers
(Drive folder creation, the welcome email, `Run n8n Automation 01`). Point this
branch at the same sub-workflow rather than copying its nodes.

**8. Notify** — a message to you and Cara with the customer name, title and
order id, so a paid order is never sitting unseen.

---

## Workflow B — PayPal

Same nodes from step 2 onward. Only the trigger and the mapping change.

Webhook event: **`CHECKOUT.ORDER.APPROVED`** (or `PAYMENT.CAPTURE.COMPLETED` if
you capture separately). Set it up in PayPal Developer → Apps & Credentials →
your app → Webhooks.

| Field | PayPal path |
|---|---|
| `email` | `resource.payer.email_address` |
| `fullName` | `resource.payer.name.given_name` + `surname` |
| `amount` | `resource.purchase_units[0].amount.value` |
| `currency` | `resource.purchase_units[0].amount.currency_code` |
| `paymentRef` | `resource.id` |
| `paymentMethod` | `"PayPal"` |

Verify the webhook signature with PayPal's `verify-webhook-signature` endpoint
before touching Airtable. An unverified payment webhook is an open door to
anyone who can guess your URL.

---

## Workflow C — attach the brief

The brief form is submitted separately, so it needs matching to the payment.

1. Trigger on the form (Tally/Fillout webhook, or Airtable trigger on
   `20 File Upload Submission` / `Book Cover Form Submissions`).
2. Match by the `ref` value passed through `thanks.html`: search
   `01 Cart Purchases` where `WooCommerce ID = ref`, follow to its order.
3. Fall back to matching on email if `ref` is empty — someone will always find
   the form link a different way.
4. Update `03 Orders`: `Project Name`, `Form 2 Submitted at`, and flip
   `Internal Status` to your "ready to assign" choice.

---

## Two things worth getting right

**Idempotency.** Stripe and PayPal both retry webhooks. Before creating
anything, search `01 Cart Purchases` for `WooCommerce ID = paymentRef` and stop
if it exists — otherwise a retry bills Lynze and Cara for a second project that
doesn't exist.

**Refunds.** Add a small second workflow on `charge.refunded` /
`PAYMENT.CAPTURE.REFUNDED` that finds the cart by `paymentRef` and sets
`Status (refunded?)` and the order's `Refund Issued?`. Cheap now, painful to
reconcile later.

---

## Test order

1. Stripe test mode → complete a checkout with `4242 4242 4242 4242`.
2. Confirm: one customer, one cart, one order, one line item — and the order's
   `Design Due At` populated (proves the product link resolved).
3. Re-fire the same webhook from Stripe's dashboard and confirm **nothing new
   is created**. That's the idempotency check.
4. Delete the test records.
