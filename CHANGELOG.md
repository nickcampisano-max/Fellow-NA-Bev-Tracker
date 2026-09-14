# Fellow NA Bev Variance Tracker Changelog

All notable changes to Fellow Osteria's NA Bev Weekly Variance Tracker are documented here.

This is a standalone instance, cloned from the Clever Koi — Central Phoenix build. It has its own
file, its own `localStorage` key (`fellow_bev_tracker_v1`), its own hosting, and no data connection
to the Clever Koi tracker. Changes to one do not reach the other; ports between them are manual and
are recorded here.

---

## [v1.0] — 2026-09-14

Rebuilt on Clever Koi's current `index.html` (its v3.10), then re-applied only what is genuinely
Fellow's. The earlier attempt was built from a written port spec, which described the logic changes
fully but the interface only partially — CK had since dropped the Flag and % of purchases columns,
added Sold to guests / $ / Comped $ / Profit, renamed the alert badge to "Check fridge," and
reordered the Period Summary table. Taking CK's file as the base rather than hand-porting from
screenshots keeps the two interfaces identical and makes the next port a diff instead of a rewrite.

### Added
- **All 29 items, with P10 opening inventory.** Taken from the P9-ending MarginEdge count — the
  actual "Non-Alcoholic Bev Cost (5250)" category — not from the all-shops cost sheet. That sheet is
  shared across concepts, was priced below Passport's current order guide on 11 of 12 matchable
  lines, listed items Fellow doesn't carry (Dr. Pepper, cranberry, coconut milk, cold brew, decaf
  drip), and omitted seven Fellow does (Ginger Beer, Jarritos, and the vanilla / hazelnut /
  pumpkin syrups, stevia, sugar).
- **Period 10 (Aug 31 – Sep 27, 2026)** seeded, with beginning on-hand set from that same count, each
  item in its own native unit.
- **A generalized unit model.** `unitOz` + `pourOz` per item, behind `unitOz()` / `isBulk()` /
  `servingsPerUnit()` / `nativeLabel()`. Previously the only bulk unit the app understood was a
  gallon, with 128 hardcoded in `purchasedServings()`, `servingsToNative()` and `costPerServing()`.
  Central Phoenix never needed more than that because it marks coffee and tea `orderOnly` — they get
  ordered, never reconciled. Fellow can't do that: isolating the bar's espresso draw is the reason
  this instance exists, so espresso and loose tea reconcile by the pound, drip coffee and iced tea by
  the case, milk by the quart, and syrup by the litre. A gallon item with no explicit `unitOz` still
  resolves to 128, so nothing that previously worked changed.
- **Unit name and Oz-per-unit fields in Settings**, so on-screen units read lb / case / qt / btl /
  bag / gal instead of labelling everything "gal."

### Changed
- **Espresso is costed at $12.05/lb landed**, not MarginEdge's $3.77/lb. The 8/25 Passport invoice
  bills Portofino Espresso at $11.30 per 1 lb bag (6 bags, $67.80) plus a $0.75 per-bag packing fee
  that is coded to To-Go & Paper Supplies but is real espresso cost. MarginEdge's $3.77 is exactly
  $11.30 ÷ 3 — the product description carries the text "3LB MINIMUM ORDER," and something read that
  as the pack size. The same bad conversion is baked into MarginEdge's shot recipe, which prices a
  shot at $0.1483 where the correct figure is $0.4745. **This needs fixing on the MarginEdge side;
  the tracker no longer inherits it.**
- **Espresso portioning comes from MarginEdge's own shot recipe** — 0.63 oz of beans yields one 2 fl
  oz shot, so 25.4 shots per pound. That is a confirmed figure, unlike the portion sizes below.
- `VENDOR_SLOTS` emptied. Central Phoenix's cadence (Shamrock three orders a week, Passport one) may
  not be Fellow's, so every vendor falls back to a single generic "Order" slot until confirmed.
- Vendors assigned: Shamrock supplies the sodas, waters, lemonade and milks; Passport supplies the
  coffee, espresso, teas and syrups — everything on Passport's 9/13 order guide.
- Stevia and sugar packets marked `orderOnly`. They sit in the 5250 category and get ordered, but
  they never ring against a ticket, so they can't reconcile.

### Known gaps
- **Portion sizes are estimates for the teas, drip coffee, matcha and syrups**, carried over from the
  old cost sheet rather than confirmed by Fellow: Tuscany drip assumes 30 bags per case at 64 oz per
  bag with a 10 oz pour (Passport's guide doesn't state bags per case); hot teas assume 1 tsp ≈ 0.07
  oz of leaf; iced teas assume 384 brewed oz per 3.5 oz bag at a 6 oz pour; matcha assumes ¼ oz;
  syrups assume 1 oz (chai 2 oz, per the cafe recipe sheet). Correct these in Settings as they're
  confirmed — they change unit cost and expected-on-hand, not the reconciliation logic.
- **No recipe layer yet.** Toast sells composed drinks — Arnold Palmer, Shirley Temple, and twelve
  cafe drinks — while the tracker reconciles purchased ingredients. Until drink sales are exploded
  into ingredients, espresso, milk, syrup, matcha and lemonade will show their full draw as expected
  on hand with nothing counting against it. This is the main piece still missing, and it's what
  actually answers how much espresso the bar is taking.
- **Jarritos, and the cafe ingredients, have no menu price.** Jarritos is a cocktail ingredient and
  never sells on its own; the milks, syrups, matcha, chocolate and espresso are cafe ingredients.
  They stay in reconciliation on purpose — their draw is exactly what needs to be visible — but they
  produce no comps and no profit figure until the recipe layer exists.
- **Tea, English Breakfast has no menu price** and appears in no Toast export. Either it sells under
  another name or it isn't on the menu.

---

## Ported from Clever Koi — 2026-09-14

These are the two logic fixes that came across from CK ahead of the full rebuild, recorded here
because they changed what Fellow's numbers mean.

### Fixed
- **Comps were being double-counted in "Accounted for."** Toast reports "Qty sold" *inclusive* of
  comped transactions — a 100%-discounted comp still rings as a sale at $0 revenue; it is not a void.
  Verified on a real row: Qty sold 52 × $4.50 = Gross $234.00. Had the 24 comped units been excluded
  from Qty sold, Gross would have been built from the 28 paid units ($126.00). It wasn't. So `sold`
  already contains `comped`, and `accountedFor = sold + comped + kitchen` counted them twice —
  inflating Accounted for and pushing Expected on hand *down*, the falsely-reassuring direction.
  Now `accountedFor = sold + kitchen`, with `soldToGuests = sold − comped` carrying the paid-only
  breakdown for revenue and profit.
- **Period-end "product missing" was quadruple-counting.** Period Summary and Compare Periods each
  summed the running balance across all four weeks (`ua += r.unaccounted`), so a pile that sat
  unsold and unpurchased for three weeks got counted three times. Unlike Purchased / Sold / Comped /
  Profit — discrete weekly events that sum correctly — the balance is a *stock* figure. It is now
  read once, at the period's last week, via `expectedOnHand()`: the same running-balance function the
  weekly carry-forward already trusts, so the period view and the week view can never disagree.

### Verification
26 assertions run in-browser against the shipped functions, covering both fixes and the unit model:
comps excluded from Accounted for while still reported as a subset; paid units, guest revenue and
profit derived from `sold − comped`; carry-in chaining across week *and* period boundaries; the
period rollup closing to the chained balance; a physical count re-basing the carry-forward and
recording the difference as the loss; espresso round-tripping 10 lb → 254 shots → 10 lb; lemonade
still computing at exactly 128 ÷ 6; each-count items left unconverted; every item carrying an
opening count. Each assertion also states what a failure would mean on the floor — including the
failure signature for both bugs above, so neither can return silently.
