# Fleet Allocation Engine — Max-Min Fairness Simulator

An interactive, single-file prototype simulating **max-min fairness** — a publicly known allocation algorithm — applied to distributing a limited vehicle supply across dealers with different demand levels.

**[Open `index.html`](./index.html) in any browser — no install, no server, no dependencies.**

## What it does

- Takes user input: any number of dealers, each with a requested vehicle count, plus total available supply
- Runs the max-min fairness (water-filling) algorithm client-side
- Renders the final allocation as a bar chart against each dealer's original demand
- Shows a full round-by-round trace of the algorithm's logic — which dealers got capped each round, at what water level, and why
- Includes a step-through replay control so you can watch the allocation build up round by round rather than just seeing the final answer

## Why I built this

Allocation problems like this show up constantly in B2B distribution — vehicles to dealers, inventory to retail locations, leads to sales reps, support tickets to agents. The "fair" answer is rarely as simple as an even split, and being able to actually simulate an allocation algorithm — not just describe it in a slide — is a useful way to pressure-test whether a proposed allocation logic will hold up before it goes anywhere near a real ops team.

## How it works

See [`ALGORITHM.md`](./ALGORITHM.md) for the full explanation of the max-min fairness logic, a worked example, and the scoping decisions behind this V1 (e.g., unweighted only — see the doc for why, and what a weighted V2 would change).

## Try it

The tool loads with a 5-dealer example pre-filled. Hit **Run allocation** to see it work, or edit the dealers/demand/supply first. Use **Next round / Prev round / Play** under the bar chart to step through how the allocation was built up.
