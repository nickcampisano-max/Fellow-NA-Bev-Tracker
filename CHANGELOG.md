# Fellow NA Bev Variance Tracker Changelog

All notable changes to Fellow Osteria's NA Bev Weekly Variance Tracker are documented here.

This is a standalone instance, cloned from the Clever Koi — Central Phoenix build. It has its own
file, its own `localStorage` key (`fellow_bev_tracker_v1`), its own hosting, and no data connection
to the Clever Koi tracker. Changes to one do not reach the other; ports between them are manual and
are recorded here.

---

## [v1.4] — 2026-09-16

### Week table trimmed to 9 columns
Cut **Sold to guests**, its **$** (guest revenue), and **Profit**. Twelve columns forced a horizontal scroll on the one table that gets checked weekly, and a scroll means half the table never gets read. Profit per item was the least useful of the three — the question this app answers is what left without a sale, not the margin on what did sell. All three are still computed and still appear in the Period Summary email export.

### Items split into "Where it's going" and "Everything else"
The nine items that carry the NA Bev number — Coke, Diet Coke, Sprite, Root Beer, Club Soda, Ginger Beer, Espresso, Lemonade, Whole Milk — render first, above a divider. The other 18 collapse behind a one-line subtotal carrying its own Purchased / Comped / Comped $ / Expected on hand / Expected on hand ($), plus a **"n flagged"** badge if anything inside tripped an alert. Nothing is hidden: the collapsed group still sums into Total and still raises its flags. Collapsed state is per-session, not saved, so the app always opens showing nine rows.

Cranberry is deliberately absent — it is coded to bar cost and is not on the NA Bev inventory count, so it is not an NA Bev item at all. Whole milk stays on the focus list despite having no BOH recipe use: with one use (4oz per coffee drink) it is the cleanest available detector for coffee drinks made and never rung.

### Period Summary restructured to match the weekly table
Same split, same reason: the nine focus items read first under a "Where it's going" header, the
other 18 fold behind a one-line subtotal with a toggle. Sorting by missing-cost is preserved
*within* each group, so the worst offender in each block still rises to the top of its own section
rather than being buried by the split. A Total row was added (the weekly table already had one).

The Period Summary keeps its own expand/collapse state rather than sharing the week table's —
expanding one to chase a syrup shouldn't silently unfold the other on a view you're about to glance
at. Purchased, Accounted and Product missing are all serving-equivalents by the time they reach this
table, so unlike the week view's raw Sold column they're dimensionally consistent and safe to sum in
the subtotal and total rows.

### New: Cost rate card
Reproduces the P&L's own arithmetic — **Cost % = (Purchases + EOP Adjustment) ÷ Gross Sales**, where EOP Adjustment = opening inventory − closing inventory — and shows the two numbers that survive inventory timing:

- **Purchase rate** (purchases ÷ sales) — immune to when a delivery lands relative to a count.
- **Inventory change** — flagged when the swing is worth 2+ points of sales.

Built after reconciling Fellow's P7–P9 EOP sheet, which showed the headline cost % is not trustworthy alone. P7 read 29.32% and P8 read 39.48%, but P7 built $245 of inventory that P8 then drew back down; paired, the two periods are 34.81%. The apparent ten-point jump was a stocking swing, not an operational change. The three-period run rate is 36.42%. All three periods reproduce to the cent through this card's formula.

Requires one typed input: **gross NA Bev sales for the period**. Blank leaves every percentage as an em dash rather than a number divided by zero.

### New: Transfers out card
Per-item log of product that left NA Bev inventory but was consumed elsewhere — kitchen club soda for tempura (7.2 bottles per batch), bar espresso for espresso martinis (2oz each). Units are entered in each item's native unit, because that is what somebody can actually count. The card computes the dollar value and the **points off the NA Bev cost %**, producing "cost read X%, transfers were Y points, true NA Bev cost Z%" with the arithmetic shown rather than asserted.

This does not change COGS — it attributes part of it, and says so: the note states plainly that it is not a credit until the receiving department's cost rises by the same dollars.

### Partial-period guard (both new cards)
Mid-period, weeks 3 and 4 are empty — and an empty week is not a week with zero sales, though the running on-hand treats it as one. Closing inventory reads high, and the cost % would report a large inventory build every Monday regardless of what actually happened. Cost %, COGS and inventory change are therefore withheld until all four weeks have data, with the reason stated on the card. Purchase rate shows throughout, because it never depends on a count.

### Verification
`verify5.mjs` — 23 assertions, all passing. Header and every body row are exactly 9 columns; the focus and collapsed groups partition the tracked items with no overlap and no omission; Total equals visible rows plus the collapsed subtotal in both expanded and collapsed states; purchases reconcile to the MarginEdge reports ($424.48 computed vs $424.15 reported, a $0.33 gap from per-unit costing rather than invoice amounts); P7, P8 and P9 each reproduce their published cost % to within 0.02 points; the partial guard both holds and lifts correctly; no JS errors.

### Known gaps
- Bar club soda usage is still unmeasured — the single largest unquantified transfer.
- Pellegrino's $1.596 unit cost is unverified against a recent invoice; nothing was delivered in the sample window.
- Whether anything sits in GL 5250 that never lands on a count sheet (packets, drip coffee) is unresolved. A full-period purchase report would settle it.

## [v1.3] — 2026-09-16

### Fixed
- **A logged count didn't reach its own week's Expected-on-hand cell.** `expectedOnHand()` already
  resets the running balance to a logged count, so a count typed into a week flowed into every later
  week, Period Summary, Compare Periods and the next period's beginning inventory — every place
  except the row it was typed into, which stayed frozen on the projection and made the Count field
  look like a dead placeholder. The cell, its dollar twin and the week's Total row now show the
  confirmed count, tagged "✓ confirmed by count"; the breakdown keeps the projection arithmetic and
  adds the counted figure and the variance beneath it rather than replacing one with the other.
  Display-only by design: `computeRow()` stays pure and `expectedOnHand()` keeps owning the
  carry-forward, so a count is never applied twice. The guard tests for blank / undefined / NaN
  rather than truthiness, so a counted **zero** (item ran out) registers as a real count.
- **The "Check fridge" badge was backwards on a confirmed row.** It tested raw dollar magnitude,
  which is the right question for an unverified projection ("is enough product riding on this to be
  worth a look?") and the wrong one for a row somebody has already counted. A perfectly accurate
  count of a large number triggered the warning purely because the number was large, while an
  obviously wrong low count silenced it. Confirmed rows now test the variance between the count and
  the projection; uncounted rows keep the original magnitude test.

### Added
- **A separate `countVarianceThreshold`, default $10**, with its own Settings field. Without it the
  badge fix above would have silently disabled the badge rather than correcting it: the existing $50
  threshold is scaled to standing inventory, and $50 of *variance* is 52 Cokes, 45 club sodas or
  4 lb of espresso. A 16-can miss — $15.41, a real problem — would never have fired.

### Verification
13 assertions in-browser: uncounted rows unchanged; a perfect count displays itself and raises no
badge (the old test fired here); a 16-can shortage flags where the old $50 test was silent; a
counted zero registers instead of reverting to the projection; clearing the field restores the
projection; the count still re-bases the following week; the Total row follows whatever each cell
actually shows. The first run failed 7 of 13 and the code was correct — the fixtures had been
written against the hosted page's loaded data instead of a fresh one.

---

## [v1.2] — 2026-09-15

### Fixed
- **Three items had been renamed away from their MarginEdge spellings**, silently breaking purchase
  matching for Acqua Panna, Pellegrino and Tuscany Blend. The failure is invisible: the rows simply
  don't import, purchases come up short, and the shortfall reads as extra missing product. Caught by
  asserting matched purchases against the report's own stated total — $424.48 computed against
  $424.15 stated, rounding only — and by checking for unmatched purchase rows, now zero.

---

## [v1.1] — 2026-09-15

### Fixed
- **Toast sales rows could never match a tracked item.** MarginEdge names the products ("Soda, Coke
  12oz Can"), Toast names the menu ("Coca-Cola"). `matchItemByName`'s substring fallback only fires
  when the tracked name sits *inside* the row name, which never happens in that direction, so every
  sales row would have landed in "unmatched". Items now carry an `aliases` list of POS spellings.
- **A Toast row could only draw down one tracked item.** The recipe lookup kept the last match and
  discarded the rest, so a Latte pulled espresso *or* milk but never both, and a Mocha Latte
  silently ignored two of its three ingredients.

### Added
- **Recipes, so composed drinks draw down what they actually consume.** Confirmed by Nick: espresso
  is 2 fl oz — one 0.63 oz shot, per MarginEdge's own shot recipe — in any drink that calls for it;
  all milks 4 oz; all syrups including chai 1 oz; matcha 7 g (0.247 oz); Mocha Latte adds 1 oz
  chocolate sauce; Arnold Palmer is 2 oz tropical iced tea + 4 oz lemonade; Shirley Temple is one
  Sprite can, its grenadine being coded Alcohol rather than NA bev.
- Milk, almond and oat portions corrected from 6 oz to 4; chai from 2 oz to 1.

### Why it matters
Toast rang **3 Sprites** in P10 week 1. Eight cans left the shelf — three sold and five inside
Shirley Temples. Unmapped, that reads as five cans of shrink in one week on one item. Same shape as
the espresso question, just visible sooner.

### Verification
Both real Toast exports map with zero unmatched rows except Orange Juice, which is not a tracked
item.

---

## Data loaded — P10 weeks 1 and 2

Loaded from Fellow's own exports: Toast product mix for Aug 31–Sep 6 and Sep 7–13, and the matching
MarginEdge purchase reports ($282.71 and $141.44).

### What it shows
```
Net NA bev sales                    $1,337.15
Purchases at cost                     $424.48   →  31.8% of sales
Product actually consumed at cost     $203.48   →  15.2% of sales
                                    ─────────
Difference onto the shelf             $221.00
```
Cost of goods taken from invoices — which is what period-to-period reporting does without a closing
count — reads **31.8%** where consumption is **15.2%**. Roughly half the reported NA bev cost in
these two weeks is an inventory build, not product leaving the building. Opening inventory was
$986.22; projected closing is $1,207.23.

The specific offenders: **144 Diet Cokes purchased against 49 sold, and 72 club sodas against 25** —
five to seven weeks of supply on two items.

Caveat: the closing figure is a projection, not a count. True COGS is opening + purchases − closing,
so if the week-4 count lands well below $1,207 the gap is real loss and this reading changes. The
inventory build itself is not a projection — it comes straight from invoices and Toast.

### Espresso baseline
Two weeks of café sales consumed **1.54 lb** of the opening 10 lb — 15 shots in week 1, 24 in week 2
— with nothing delivered until the 9/15 Passport order. **Expected on hand: 8.46 lb.** Whatever the
scale says below that is the bar's draw, measured rather than argued.

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
- **Portion sizes were estimates for the teas, drip coffee, matcha and syrups**, carried over from
  the old cost sheet rather than confirmed by Fellow. Partly settled in v1.1 — milks, syrups, matcha
  and espresso are now confirmed. **Still estimates:** Tuscany drip assumes 30 bags per case at 64 oz
  per bag with a 10 oz pour (Passport's guide doesn't state bags per case); hot teas assume 1 tsp ≈
  0.07 oz of leaf; iced teas assume 384 brewed oz per 3.5 oz bag at a 6 oz pour. Correct these in
  Settings as they're confirmed — they change unit cost and expected-on-hand, not the logic.
- ~~**No recipe layer yet.**~~ Built in v1.1.
- **Alternative milks can't be separated from whole.** Toast sells a "Latte" without saying which
  milk, so every cafe drink draws whole milk. Almond and oat will read as unaccounted while whole
  reads over-accounted. If Toast captures the alt-milk upcharge as a modifier, a modifier-level
  export could split them.
- **Jarritos, and the cafe ingredients, have no menu price.** Jarritos is a cocktail ingredient and
  never sells on its own; the milks, syrups, matcha, chocolate and espresso are cafe ingredients.
  They stay in reconciliation on purpose — their draw is exactly what needs to be visible — but they
  produce no comps and no profit figure until the recipe layer exists.
- **Tea, English Breakfast has no menu price** and appears in no Toast export. Either it sells under
  another name or it isn't on the menu.
- **Orange Juice sells but isn't tracked** — it appears in Toast and in no inventory count.
- **Passport ships free only at 25 lb of coffee and/or tea.** The 8/25 order cleared it; the 9/15
  order was 3 lb and carried $20.00 of freight — $6.67 a pound on top of $11.30. Coffee and tea
  count together toward the threshold, so consolidating the bi-weekly order could remove it.

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
