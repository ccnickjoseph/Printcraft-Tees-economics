# The economic model

Everything the calculator computes lives in `calc()` at `index.html:429`. This document is the
spec for that function, the provenance of every number feeding it, and what the model has
found so far.

Last data review: **2026-08-27**.

---

## Formulas

Per order of quantity `q`, on one product and one channel:

```
unitPrice   = price, less bulk discount if q >= discountMinQty
sub         = unitPrice * q                       product revenue
free        = sub >= threshold
charged     = 0 if free, else per shipMode        shipping revenue
cart        = sub + charged                       what the customer pays

blank       = (productCost + 2XL uplift) * q
shipCost    = shipFirst + shipAdd * (q - 1)       what Printful bills us
fee         = cart * channelRate% + channelFixed
payout      = sub * affiliateOrCreator%           channel-specific
charity     = sub * charity%                      channel-specific

net         = cart - (blank + shipCost + fee + payout + charity)
margin      = net / cart
perUnit     = net / q
shipPL      = charged - shipCost
```

### Basis rules — the part that is easy to get wrong

- **Channel fees apply to the cart total *including* shipping charged.** You pay processing on
  the shipping you collect.
- **Payout and charity apply to product revenue *excluding* shipping.** You do not pay an
  affiliate a cut of postage.
- **Printful's shipping cost is deducted on every row**, whether or not the customer was
  charged for it. Free shipping does not make shipping free; it moves who pays.
- **Payout and charity assume 100% attribution** — every order on that channel pays them. That
  is the pessimistic case. Real Shopify orders are a mix of affiliate-attributed and direct.
  An attribution-rate input is on the roadmap.

### Shipping charge modes

| Mode | Behavior |
|---|---|
| `pass` (default) | Customer pays exactly what Printful bills. Shipping P&L nets to zero below the threshold. |
| `tiered` | Your own first-item + each-additional rates. |
| `flat` | One fee regardless of quantity. Leaks on larger sub-threshold carts. |

### Quantity ladder

`[1,2,3,4,5,6,7,8,9,10,15,20,25,30,40,50,60,70,80,90,100]` — 21 points. Chart labels are
thinned via `LABEL_Q`; unlabelled points get tick marks.

---

## Data and provenance

### Blank costs — CONFIRMED

Source: Notion "All Blanks & Details", Printful view, read 2026-08-27
(`collection://37cec00e-c0b1-8058-b58f-000bbb781572`).
Cost = blank + one DTG print location, sizes S–XL. 2XL adds $2.

| Style | Product | Cost | Retail | Category | Retail confidence |
|---|---|---|---|---|---|
| G64000 | Softstyle Tee | $9.63 | $20 | tee | **confirmed** against live CFT Shopify store |
| G5000 | Heavy Cotton Tee | $9.44 | $20 | tee | placeholder |
| G18000 | Crewneck Sweatshirt | $19.17 | $45 | fleece | placeholder |
| G18500 | Pullover Hoodie | $22.63 | $50 | fleece | placeholder |
| G18600 | Full-Zip Hoodie | $24.92 | $55 | fleece | placeholder |

Placeholder retail prices are **invented, not live**. The UI flags them with a badge and a
banner. Do not quote a margin on a fleece style externally.

Open: whether $9.63 is standard Printful pricing or Growth plan pricing. Changes every fleece
and tee number if it is the latter.

### Shipping — CONFIRMED

US Printful rates, confirmed by Nick 2026-08-27.

| Category | First item | Each additional |
|---|---|---|
| tee | $4.95 | $2.20 |
| fleece | $8.79 | $2.50 |

**Mixed carts are not modeled.** Printful re-rates an order containing both a tee and a hoodie
using the higher first-item rate, so real mixed orders cost more than either single-product
view shows.

### Channel fees — CONFIDENCE VARIES

| Channel | Rate | Fixed | Payout type | Payout % | Charity % | Confidence |
|---|---|---|---|---|---|---|
| Shopify DTC | 2.9% | $0.30 | affiliate | 15 | 5 | high — standard card processing |
| Amazon | 5 / 10 / 17% tiered | — | none | 0 | 0 | **verify tier boundaries** |
| eBay | 13.25% | $0.30 | none | 0 | 0 | **verify against actual account rate** |
| TikTok Shop | 8% | — | creator | 15 | 0 | **lowest confidence input in the model** |

Amazon tiers: 5% at or under $15, 10% from $15.01 to $20.00, 17% above $20.00. The UI warns
when price > $20 on Amazon.

### Baseline values

Free shipping threshold $70 · target net margin 12% · 2XL uplift $2 · bulk discount 0% at
qty ≥ 25 · shipMode `pass` · payout and charity both on.

Defined in `baseDefaults()` (`index.html:386`). This is what "Changed from baseline" compares
against — changing it changes the meaning of every shared snapshot, so treat it as a decision,
not a tweak.

### Explicitly excluded

Returns, chargebacks, ad spend and CAC, Printful Growth plan pricing, Amazon's minimum referral
fee, mixed-cart re-rating, sales tax, international shipping. All margins here are
pre-marketing.

---

## Findings

Baseline unless stated: G64000, $20, Shopify DTC, $70 threshold, 15% affiliate, 5% charity.

### 1. Profit cliff at 4 tees

| q | 1 | 2 | 3 | **4** | 5 | 6 |
|---|---|---|---|---|---|---|
| net | $5.35 | $11.07 | $16.80 | **$11.31** | $14.90 | $18.49 |
| per unit | $5.35 | $5.54 | $5.60 | **$2.83** | $2.98 | $3.08 |

Crossing $70 stops $11.55 of shipping revenue and starts absorbing the cost. Order profit does
not recover to the 3-unit level until 6 units.

### 2. Raising the threshold relocates the cliff, it does not remove it

Per-unit profit at q = 1…6, sweeping the free shipping threshold:

| Threshold | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| $70 / $80 | 5.35 | 5.54 | 5.60 | **2.83** | 2.98 | 3.08 |
| $85 / $90 / $100 | 5.35 | 5.54 | 5.60 | 5.63 | **2.98** | 3.08 |
| $120 | 5.35 | 5.54 | 5.60 | 5.63 | 5.65 | **3.08** |

Per-unit profit never exceeds ~$5.65 once shipping goes free, at any threshold. A hard
threshold is a step function; moving it buys exactly one more paid-shipping unit. The
structural fixes are charging partial shipping above the threshold, or a price that absorbs
it — not a bigger number in the same box.

### 3. Fleece has the same problem at 2 units

One G18500 hoodie at $50 is under threshold: nets $15.37 at 26.1%. Two is $100, free shipping,
$20.25 at 20.3% — per-unit halves from $15.37 to $10.13.

### 4. Bulk is a 1-to-3 story, not a 1-to-100 story

Per-unit shipping asymptotes at the additional-item rate. Going from 40 units to 100 moves
per-unit profit by about $0.05. The interesting range is entirely in single digits.

### 5. Single-unit orders are strong when the customer pays shipping

The intuition that bulk is always better is wrong here. A 1-unit paid-shipping order beats a
4-unit free-shipping order on per-unit profit by 2x.

### 6. Payout must be per channel

Amazon and eBay carry no affiliate or creator payout. Modeling one global rate made the
marketplaces look far worse than they are. This is already fixed in the model; noted so it is
not "simplified" back.

### 7. The Amazon $20 boundary is worth $1.73 a unit

At $20.00 a single tee nets $7.87 (31.6%). At $20.01 it nets $6.14 (24.6%) — the referral fee
jumps 10% → 17%. You do not recover the $20.00 net until roughly $22. Pricing anything into
$20.01–$22 on Amazon is strictly worse than pricing at $20.00.

---

## Verifying a change to this model

There are no automated tests. Reproduce the arithmetic in a scratch Node script before
believing any new finding — the step functions above make eyeballing unreliable. Ten lines is
enough to transcribe `calc()`.
