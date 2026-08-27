# Goals, open questions, and build order

## What this tool is trying to become

Today it answers "what do we keep on this configuration?" — you move inputs and read results.
The direction of travel is the reverse: **you state a target and it tells you the inputs that
hit it.** The price solver and order-mix weighting are both steps in that direction. The cliff
detector is a third thing: findings the tool surfaces on its own rather than waiting to be
asked.

A secondary goal is confidence. Several inputs are guesses today and the UI is honest about
which. Every guess retired makes every output more usable.

---

## Verify — blocking real decisions

These are not code work. Until they are done, the tool models an assumed business rather than
the actual one.

- [ ] **Amazon apparel referral tier boundaries** — Seller Central. The $20.00/$20.01 boundary
      is worth $1.73 a unit; worth knowing it is exactly where we think.
- [ ] **eBay final value fee for apparel** — the actual account rate, not the published one.
- [ ] **TikTok Shop platform fee** — 8% is a guess and the lowest-confidence input in the model.
      The entire TikTok column rests on it.
- [ ] **Real retail prices for G5000, G18000, G18500, G18600** — four of five prices in the
      model are invented. See the price solver below; these two items are the same project.
- [ ] **Whether $9.63 is standard or Printful Growth plan pricing** — moves every number if it
      is Growth.

---

## Features, in build order

### 1. Cliff detector

Scan the quantity ladder and flag any `q` where adding a unit reduces profit. Render as a
callout.

Three of these were found by hand (4 tees, 2 fleece, and the Amazon $20.01 fee boundary). The
detector turns that into something the tool reports across every product × channel combination
automatically. It is small — roughly 30 lines — and it is what makes finding #2 in
[MODEL.md](MODEL.md) actionable rather than a footnote.

Note: the existing `#cliff` div holds the Amazon *fee* tier warning. Different feature, name
the new one differently.

### 2. Price solver

Enter a target margin and an order size, get the retail price required. The reverse of the
current flow.

**This is the feature that sets the fleece prices**, which is why it comes before order-mix
weighting. The solver has to scan rather than solve algebraically: the free shipping threshold
makes margin discontinuous in price, so there can be a range of prices with no solution and a
jump across it.

### 3. Order mix weighting

Percentage of orders at 1 / 2 / 3 / 4+ units, output one blended margin. Turns the threshold
question from a guess into arithmetic.

Deliberately after the solver: every blended number it produces would otherwise be built on
four placeholder prices.

### 4. Price ladder grid

Retail price down, order size across, net per unit in the cells, color-coded against target.
Would make the Amazon $20.01 cliff and the threshold cliff visible in one glance instead of
requiring a sweep.

### 5. Return rate and CAC inputs

Current margins are pre-marketing and pre-returns, which means they are not the margins. This
is the largest single gap between the model and reality.

### 6. Everything else

- Printful Growth plan toggle with break-even volume
- Affiliate attribution rate — not every Shopify order pays 15%, and assuming they all do is
  the pessimistic case baked into every current number
- Mixed-cart modeling — Printful re-rates tee + hoodie orders at the higher first-item rate
- CSV export

---

## Known limitations

Ordered by how likely they are to bite.

- **No saved scenario library.** Persistence is the URL hash and nothing else. Losing a link
  loses the scenario.
- **Full re-render on every keystroke.** Fine at this size. First thing to fix if it lags —
  and only then.
- **`syncShipRates()` uses a `lastCat` sentinel** to decide when to reset rates on product
  switch. Slightly fragile; worth a cleaner approach.
- **Fee overrides (`S.fo`) store strings, not numbers**, coerced at read time.
- **`shipCost()` reads `S.sfOverride` / `S.saOverride`, which nothing ever sets.** Dead code.

## Privacy, if it ever matters

Repo and site are public, and so are the costs and margins. Deliberate — costs are not
competitive today. If that changes: Cloudflare Pages (account already exists) with Cloudflare
Access in front. Free up to 50 users, real identity-based auth. A client-side password prompt
is not worth building; the password ships to the browser.
