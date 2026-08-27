# Printcraft Tees — Unit Economics Dashboard

A single-page calculator that models per-unit and per-order profitability for Printcraft Tees'
Printful print-on-demand catalog across four sales channels.

**Live:** https://ccnickjoseph.github.io/Printcraft-Tees-economics/

- **[docs/MODEL.md](docs/MODEL.md)** — the economics: formulas, data, provenance, findings
- **[docs/ROADMAP.md](docs/ROADMAP.md)** — goals, open questions, what to build next
- **[CLAUDE.md](CLAUDE.md)** — orientation for anyone (or any agent) editing the code

## The question it answers

Print-on-demand margins are thin enough that intuition is unreliable. Shipping is a step
function, marketplace fees are tiered, and affiliate and charity payouts stack on top. Whether
an order makes money depends on the interaction of all of it — and the answer changes at every
order size.

This tool makes that interaction visible: pick a product, pick a channel, and see where every
dollar of the cart goes across 21 order sizes from 1 to 100 units.

## What it is for

**Setting the free shipping threshold.** The largest single lever in the model. The threshold
panel scans carts of 1–8 units, assumes you absorb the full Printful shipping cost, and names
the smallest cart that still clears your target net margin.

**Deciding which channels are worth listing on.** Fees, and whether a channel carries an
affiliate or creator payout, vary enough to reorder the ranking. Modeling one global payout
rate made the marketplaces look far worse than they are — payout is per channel for a reason.

**Pricing products that are not priced yet.** Four of the five retail prices in the model are
placeholders. The tool shows what a given price does; it does not yet solve for the price a
target margin requires (see the roadmap).

**Sanity-checking bulk and promo ideas.** Bulk discounts, 2XL upcharges, and flat-rate
shipping all sound reasonable until you see them at 4 units versus 40.

**Sharing a scenario for approval.** Every input encodes into the URL. You send a link, not a
screenshot, and the recipient sees exactly your configuration with a panel listing what you
changed from the baseline.

## How to use it

Left rail is inputs, right column is output. Change anything and everything re-renders.

- **Product chips** switch between the five blanks. Shipping cost follows the product's
  category automatically (tee vs fleece).
- **Channel chips** switch between Shopify DTC, Amazon, eBay, and TikTok Shop. Each carries its
  own fee structure, payout type, and confidence note.
- **Where every dollar goes** breaks each order size into stacked cost segments. Click a row to
  make it the focus of the headline stats.
- **Chart** plots per-unit profit, per-order profit, or a product comparison across the ladder.
- **Free shipping threshold** is the recommendation panel — it names a dollar figure.
- **Scenario detail** is the full table if you want the raw numbers.
- **Changed from baseline** lists only what differs from the default model. An empty panel means
  you are looking at the baseline.

### Sharing

**Copy link** encodes all state into the URL hash — send that, not the bare URL, or the
recipient loses your configuration. **Copy code** is the same payload as a short string, for
pasting into the load box when a link would get mangled. **Reset** returns to baseline.

## Trust level of the numbers

Read [docs/MODEL.md](docs/MODEL.md) before quoting anything from this tool externally.

- **Confirmed:** blank costs (Notion, 2026-08-27), Printful US shipping rates, the G64000 $20
  retail price, Shopify card processing.
- **Placeholder:** retail prices for G5000, G18000, G18500, G18600. Flagged in the UI.
- **Unverified:** Amazon referral tier boundaries, the eBay final value rate, and the TikTok
  Shop platform fee — the last is the lowest-confidence input in the model.
- **Not modeled:** mixed carts, returns, chargebacks, ad spend, Printful Growth plan pricing.

## Development

Edit `index.html`, commit to `main`, live in about a minute. No build step, no dependencies,
no tests. Open the file directly in a browser to preview.

The repo and the site are public, and so are the costs and margins in them. That was a
deliberate call — costs are not competitive right now. If that changes, the path is Cloudflare
Pages (account exists) with Cloudflare Access in front, which is free up to 50 users and gives
real identity-based auth. A client-side password prompt is not worth building; the password
ships to the browser.
