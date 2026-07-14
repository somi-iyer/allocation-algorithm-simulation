# Max-Min Fairness — How This Prototype Works

## The problem it solves

An OEM (or distributor) has a fixed supply of vehicles and a set of dealers, each requesting a certain number of units. Total demand often exceeds supply. The naive approaches — first-come-first-served, or a flat percentage cut for everyone — both break down in practice:

- **First-come-first-served** rewards whoever submits their order fastest, not whoever needs it most, and can leave small dealers with nothing.
- **Flat percentage cut** (e.g., "everyone gets 60% of what they asked for") sounds fair but isn't: a dealer who over-requested gets padded, while a dealer who requested conservatively gets under-served relative to their actual need — and it can allocate more to a dealer than they even asked for isn't possible, but it wastes supply on dealers who didn't need their full cut, with no mechanism to redirect that surplus to dealers who could have used it.

**Max-min fairness** solves this with one guiding principle: *maximize the minimum allocation any dealer receives, without wasting supply on dealers who don't need more than they asked for.*

## The algorithm (water-filling)

Think of supply as water being poured into a set of buckets (dealers), where each bucket has a maximum height (its demand). The rule: raise the water level equally across all buckets simultaneously. The moment a bucket hits its own maximum height (its demand is fully met), it stops rising and is removed from the pool — but the water level keeps rising for everyone else, using the supply that would have overflowed that bucket.

**Formally, each round (unweighted):**

1. Take the dealers who still have unmet demand ("active" dealers).
2. Compute the *water level* = remaining supply ÷ number of active dealers.
3. Any active dealer whose remaining demand is **less than or equal to** the water level gets capped — allocate them their full remaining demand, and remove them from the active pool.
4. Subtract what was just allocated from the remaining supply.
5. Repeat with the smaller active pool and the smaller remaining supply, until either:
   - No dealer gets capped in a round → everyone left gets an equal share of what remains, and the algorithm terminates, or
   - Supply runs out, or
   - Every dealer's demand has been met (any leftover supply is reported as unallocated surplus).

## V2: weighted max-min fairness

The unweighted version above treats every dealer's claim on supply as identical. In practice, dealers often have a legitimate reason for an unequal claim — historical sell-through rate, contract tier, dealership size. **V2 generalizes the algorithm with a per-dealer weight**, toggleable in the tool.

The water-filling intuition still holds, but instead of every bucket rising at the same rate, buckets rise at a rate proportional to their weight — a dealer with weight 2 fills twice as fast as a dealer with weight 1, for as long as both are still active.

**Formally, each round (weighted):**

1. Take the active dealers and sum their weights: `activeWeightSum`.
2. Compute `t` = remaining supply ÷ `activeWeightSum` — this is the water level *per unit of weight*, not per dealer.
3. Each active dealer's tentative allocation this round is `t × their own weight`.
4. Any dealer whose remaining demand is **less than or equal to** their tentative allocation gets capped at their remaining demand and removed from the pool; the difference between their tentative allocation and what they actually needed flows back into the pool for the next round.
5. Repeat until no one gets capped in a round (remaining active dealers each receive `t × weight`) or supply/demand is exhausted.

Setting every weight to 1 makes `activeWeightSum` equal to the dealer count and collapses this exactly back to the V1 algorithm — the weighted version is a strict generalization, not a different algorithm.

**Worked example (weighted):** Same five dealers and 100-unit supply as below, but now with weights A=1.5, B=1, C=2, D=0.5, E=1 (Dealer C, a larger dealership, has twice the claim of Dealer B or E; Dealer D, a smaller satellite location, has half). Result: A=25, B=16.67, C=33.33, D=8.33, E=16.67 — every unit of weight is treated equally, so C consistently receives roughly double what B or E get, and double what D gets four times over, for as long as all four remain in the active pool together.

## Why this is "fair" in a formal sense

An allocation is max-min fair if you cannot increase any dealer's allocation without decreasing the allocation of some other dealer who already has an equal or smaller share. Practically, this means:

- No dealer with unmet demand is left worse off just so a dealer who already has more can get even more.
- Small dealers are never crowded out by large dealers — supply that a large, capped dealer doesn't need is systematically redirected to dealers who still need more.
- The algorithm is deterministic and auditable — the same inputs always produce the same allocation, and every allocation decision traces back to a specific round and water level.

## Worked example

Supply: 100 vehicles. Dealers: A=40, B=25, C=60, D=15, E=30 (total demand = 170, so supply is constrained).

- **Round 1:** Water level = 100 / 5 = 20. Dealers D (demand 15) is below 20 → capped, allocated 15 in full. Remaining supply = 85. Active pool shrinks to {A, B, C, E}.
- **Round 2:** Water level = 85 / 4 = 21.25. Dealer B (demand 25, remaining 25) is above the level, stays active. No dealer is at or below 21.25... check each: A=40, B=25, C=60, E=30 — none capped. So round 2 is the terminal round: everyone remaining gets 21.25.

Final allocation: A=21.25, B=21.25, C=21.25, D=15, E=21.25 — every dealer either got their full request or an equal share with every other unmet dealer, and no supply went unused.

## Complexity

Each round removes at least one dealer from the active pool, so the algorithm terminates in at most *n* rounds for *n* dealers — O(n²) in the worst case with the naive re-scan-every-round approach used in this prototype (fine for the dealer-network scale this is designed for; a sorted-demand approach can bring this to O(n log n) if needed at larger scale).

## Scope

- **V1** shipped unweighted only — every dealer treated as having equal claim to supply. That was a deliberate scope cut to get a correct, explainable baseline shipped first, not an oversight — see the [PRD collection](../prd-collection) for how this kind of scoping decision gets made and written up.
- **V2** adds the weighted generalization described above, exposed as an explicit on/off toggle in the tool rather than replacing V1's behavior — unweighted stays the default so the simpler model is still available and the two are easy to compare side by side.
- **Not yet in scope:** dynamic/time-varying weights (e.g., a weight that decays if a dealer under-sells their last allocation), and multi-resource allocation (allocating several vehicle trims/models simultaneously with cross-model constraints). Both are reasonable V3 candidates if this moved toward a real tool.
