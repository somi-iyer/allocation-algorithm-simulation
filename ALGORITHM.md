# Max-Min Fairness - Water filling algorithm

## Intention of this exercise
- Use a publicly available algorithm and create a quick working prototype using AI ( just a free-tier of LLM) which proves my ability to quickly seek feedback and test a concept without any dependency on Dev Team
- Learn how to create professional .md files  
## The problem it solves

An OEM (or distributor) has a fixed supply of vehicles and a set of dealers, each requesting a certain number of units. Total demand often exceeds supply. The naive approaches - first-come-first-served, or a flat percentage cut for everyone - both break down in practice:

- **First-come-first-served** rewards whoever submits their order fastest, not whoever needs it most, and can leave small dealers with nothing.
- **Flat percentage cut** (e.g., "everyone gets 60% of what they asked for") sounds fair but isn't: a dealer who over-requested gets padded, while a dealer who requested conservatively gets under-served relative to their actual need - and it can allocate more to a dealer than they even asked for isn't possible, but it wastes supply on dealers who didn't need their full cut, with no mechanism to redirect that surplus to dealers who could have used it.

**Max-min fairness** solves this with one guiding principle: *maximize the minimum allocation any dealer receives, without wasting supply on dealers who don't need more than they asked for.*

## The algorithm (water-filling)

Think of supply as water being poured into a set of buckets (dealers), where each bucket has a maximum height (its demand). The rule: raise the water level equally across all buckets simultaneously. The moment a bucket hits its own maximum height (its demand is fully met), it stops rising and is removed from the pool - but the water level keeps rising for everyone else, using the supply that would have overflowed that bucket.

**Formally, each round (unweighted):**

1. Take the dealers who still have unmet demand ("active" dealers).
2. Compute the *water level* = remaining supply ÷ number of active dealers.
3. Any active dealer whose remaining demand is **less than or equal to** the water level gets capped - allocate them their full remaining demand, and remove them from the active pool.
4. Subtract what was just allocated from the remaining supply.
5. Repeat with the smaller active pool and the smaller remaining supply, until either:
   - No dealer gets capped in a round → everyone left gets an equal share of what remains, and the algorithm terminates, or
   - Supply runs out, or
   - Every dealer's demand has been met (any leftover supply is reported as unallocated surplus).

The result is each dealer's **fair share**, which can be fractional (e.g., 21.25). Because vehicles can't be split, a final reconciliation step converts fair shares to whole vehicles - see [Whole-vehicle allocation](#whole-vehicle-allocation-v21) below.

## V2: weighted max-min fairness

The unweighted version above treats every dealer's claim on supply as identical. In practice, dealers often have a legitimate reason for an unequal claim -historical sell-through rate, contract tier, dealership size. **V2 generalizes the algorithm with a per-dealer weight**, toggleable in the tool.

The water-filling intuition still holds, but instead of every bucket rising at the same rate, buckets rise at a rate proportional to their weight - a dealer with weight 2 fills twice as fast as a dealer with weight 1, for as long as both are still active.

**Formally, each round (weighted):**

1. Take the active dealers and sum their weights: `activeWeightSum`.
2. Compute `t` = remaining supply ÷ `activeWeightSum` - this is the water level *per unit of weight*, not per dealer.
3. Each active dealer's tentative allocation this round is `t × their own weight`.
4. Any dealer whose remaining demand is **less than or equal to** their tentative allocation gets capped at their remaining demand and removed from the pool; the difference between their tentative allocation and what they actually needed flows back into the pool for the next round.
5. Repeat until no one gets capped in a round (remaining active dealers each receive `t × weight`) or supply/demand is exhausted.

Setting every weight to 1 makes `activeWeightSum` equal to the dealer count and collapses this exactly back to the V1 algorithm - the weighted version is a strict generalization, not a different algorithm.

**Worked example (weighted):** Same five dealers and 100-unit supply as below, but now with weights A=1.5, B=1, C=2, D=0.5, E=1 (Dealer C, a larger dealership, has twice the claim of Dealer B or E; Dealer D, a smaller satellite location, has half). Fair share: A=25, B=16.67, C=33.33, D=8.33, E=16.67 — every unit of weight is treated equally, so C consistently receives roughly double what B or E get, and four times what D gets. After [whole-vehicle reconciliation](#whole-vehicle-allocation-v21) the final allocation is A=25, B=17, C=33, D=8, E=17.

## Whole-vehicle allocation (V2.1)

Water-filling produces a real-valued **fair share** (e.g., 21.25). A vehicle can't be split between dealers, so V2.1 adds a final **reconciliation** step using the largest-remainder method (also known as Hamilton's method):

1. Floor every dealer's fair share to a whole number.
2. Count the whole vehicles still unplaced: total fair share minus the sum of the floors.
3. Give one extra vehicle at a time to the dealers with the **largest fractional remainder**.
4. If remainders tie, the dealer with the **largest unmet demand** (demand minus floored share) wins; if that also ties, input order decides. This keeps results deterministic.

Guarantees:

- The total allocated never changes - only the split between dealers does.
- No dealer receives more than they requested (a dealer only gets an extra vehicle if their floored share is still below their demand).
- Every dealer's final allocation is within 1 vehicle of their fair share.
- Weights stay decimal (they are ratios); demand and total supply must be whole numbers, which the tool now validates.

**Walk-through (unweighted example below):** fair shares are A=21.25, B=21.25, C=21.25, D=15, E=21.25 (total 100). Floors are 21, 21, 21, 15, 21 (total 99), so 1 vehicle is still unplaced. A, B, C and E tie on a 0.25 remainder, so the tie-break applies: unmet demand after flooring is A=19, B=4, C=39, E=9, and C has the most. Final: A=21, B=21, C=22, D=15, E=21.

**Walk-through (weighted example above):** fair shares are A=25, B=16.67, C=33.33, D=8.33, E=16.67 (total 100). Floors are 25, 16, 33, 8, 16 (total 98), so 2 vehicles are unplaced. The largest remainders are B and E (about 0.67 each), ahead of C and D (about 0.33), so B and E each get one. Final: A=25, B=17, C=33, D=8, E=17.

In the tool, the replay gets one extra step after the last round: intermediate steps show fair shares (which may be fractional), and the final step shows whole vehicles. The trace lists each flooring decision and every extra vehicle awarded.

## Why this is "fair" in a formal sense

An allocation is max-min fair if you cannot increase any dealer's allocation without decreasing the allocation of some other dealer who already has an equal or smaller share. This holds exactly for the fair share produced by water-filling. After whole-vehicle reconciliation it holds to within 1 vehicle per dealer: with indivisible units, a perfectly equal split isn't always possible (e.g., 10 vehicles across 3 equal dealers is 4/3/3), so the tool guarantees the closest whole-number outcome rather than exact equality. Practically, this means:

- No dealer with unmet demand is left worse off just so a dealer who already has more can get even more.
- Small dealers are never crowded out by large dealers - supply that a large, capped dealer doesn't need is systematically redirected to dealers who still need more.
- The algorithm is deterministic and auditable - the same inputs always produce the same allocation, and every allocation decision traces back to a specific round and water level (and, for whole-vehicle rounding, to a specific remainder and tie-break).

## Worked example

Supply: 100 vehicles. Dealers: A=40, B=25, C=60, D=15, E=30 (total demand = 170, so supply is constrained).

- **Round 1:** Water level = 100 / 5 = 20. Dealer D (demand 15) is below 20 → capped, allocated 15 in full. Remaining supply = 85. Active pool shrinks to {A, B, C, E}.
- **Round 2:** Water level = 85 / 4 = 21.25. Check each remaining dealer's demand against it: A=40, B=25, C=60, E=30 - all are above 21.25, so none is capped. Round 2 is therefore the terminal round: everyone remaining gets 21.25.
- **Whole-vehicle reconciliation:** floor each share (21, 21, 21, 15, 21 = 99), leaving 1 vehicle to place. A, B, C and E tie on a 0.25 remainder, so the dealer with the largest unmet demand wins: C (39 unmet).

Fair share: A=21.25, B=21.25, C=21.25, D=15, E=21.25 - every dealer either got their full request or an equal share with every other unmet dealer, and no supply went unused. Vehicles can't be split, so this is converted to whole vehicles: A=21, B=21, C=22, D=15, E=21 (the one leftover vehicle goes to C, the tied dealer with the largest unmet demand).

## Scope

- **V1** shipped unweighted only - every dealer treated as having equal claim to supply. That was a deliberate scope cut to get a correct, explainable baseline shipped first, not an oversight - see the [PRD collection](../prd-collection) for how this kind of scoping decision gets made and written up.
- **V2** adds the weighted generalization described above, exposed as an explicit on/off toggle in the tool rather than replacing V1's behavior - unweighted stays the default so the simpler model is still available and the two are easy to compare side by side.
- **V2.1** makes allocations whole vehicles via largest-remainder reconciliation. It is a bug fix: V1 and V2 could output fractional allocations like 16.5 or 21.25, which don't match reality since a vehicle can't be split between dealers. Water-filling itself is unchanged; the fix is a final step layered on top, and demand/supply inputs are now validated as whole numbers.
- **Not yet in scope:** alternative tie-break rules for rounding (e.g., favoring the higher-weight dealer), minimum-order or lot-size constraints (e.g., vehicles that ship in multiples), dynamic/time-varying weights (e.g., a weight that decays if a dealer under-sells their last allocation), and multi-resource allocation (allocating several vehicle trims/models simultaneously with cross-model constraints). Both are reasonable V3 candidates if this moved toward a real tool.
