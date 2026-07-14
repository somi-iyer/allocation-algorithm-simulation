# Max-Min Fairness — How This Prototype Works

## The problem it solves

An OEM (or distributor) has a fixed supply of vehicles and a set of dealers, each requesting a certain number of units. Total demand often exceeds supply. The naive approaches — first-come-first-served, or a flat percentage cut for everyone — both break down in practice:

- **First-come-first-served** rewards whoever submits their order fastest, not whoever needs it most, and can leave small dealers with nothing.
- **Flat percentage cut** (e.g., "everyone gets 60% of what they asked for") sounds fair but isn't: a dealer who over-requested gets padded, while a dealer who requested conservatively gets under-served relative to their actual need — and it can allocate more to a dealer than they even asked for isn't possible, but it wastes supply on dealers who didn't need their full cut, with no mechanism to redirect that surplus to dealers who could have used it.

**Max-min fairness** solves this with one guiding principle: *maximize the minimum allocation any dealer receives, without wasting supply on dealers who don't need more than they asked for.*

## The algorithm (water-filling)

Think of supply as water being poured into a set of buckets (dealers), where each bucket has a maximum height (its demand). The rule: raise the water level equally across all buckets simultaneously. The moment a bucket hits its own maximum height (its demand is fully met), it stops rising and is removed from the pool — but the water level keeps rising for everyone else, using the supply that would have overflowed that bucket.

**Formally, each round:**

1. Take the dealers who still have unmet demand ("active" dealers).
2. Compute the *water level* = remaining supply ÷ number of active dealers.
3. Any active dealer whose remaining demand is **less than or equal to** the water level gets capped — allocate them their full remaining demand, and remove them from the active pool.
4. Subtract what was just allocated from the remaining supply.
5. Repeat with the smaller active pool and the smaller remaining supply, until either:
   - No dealer gets capped in a round → everyone left gets an equal share of what remains, and the algorithm terminates, or
   - Supply runs out, or
   - Every dealer's demand has been met (any leftover supply is reported as unallocated surplus).

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

## Scope of this prototype

This is an **unweighted** implementation — every dealer is treated as having equal claim to supply. A natural V2 extension is weighted max-min fairness (e.g., weighting by dealer size, historical sell-through rate, or contractual tier), which changes step 2 to divide supply proportionally to weight rather than equally. That's a deliberate scope cut for this prototype, not an oversight — see the [PRD collection](../prd-collection) for how this kind of scoping decision gets made and written up.
