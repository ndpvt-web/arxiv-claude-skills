---
name: "deepplanning-benchmarking-long-horizon-agentic"
description: "Solve long-horizon planning tasks with verifiable constraints using the DeepPlanning methodology: proactive information gathering, local constraint reasoning, and global constrained optimization. Use when asked to 'plan a multi-day trip with budget constraints', 'build a shopping optimizer with coupons and sizing', 'create an agent that handles complex multi-step planning', 'design a constraint-satisfaction planner', 'optimize across interdependent decisions with budgets', or 'build a planning benchmark with verifiable solutions'."
---

# DeepPlanning: Long-Horizon Agentic Planning with Verifiable Constraints

This skill enables Claude to tackle complex, long-horizon planning problems that require simultaneously satisfying local constraints (e.g., business hours, seat availability, item sizing) and global optimization objectives (e.g., time budgets, financial budgets, cross-task dependencies). Based on the DeepPlanning benchmark, it teaches a three-layer planning discipline: proactive information acquisition before committing to decisions, fine-grained local constraint checking at each step, and global feasibility verification across the entire solution. The methodology applies to any domain where an agent must gather information from APIs/tools, reason about overlapping constraints, and produce a holistic plan rather than a sequence of locally-greedy choices.

## When to Use

- When the user asks to plan a multi-day trip involving flights, hotels, attractions, restaurants, and budget limits
- When building a shopping cart optimizer that must satisfy sizing, brand preferences, coupon stacking rules, and a total budget
- When designing an agentic system that must call multiple APIs/tools to gather information before making interdependent decisions
- When implementing a constraint-satisfaction planner where local choices propagate constraints to future steps
- When the user needs to evaluate or benchmark an LLM's planning ability on tasks with verifiable correct answers
- When building any multi-step workflow where skipping an information-gathering call leads to invalid downstream decisions
- When optimizing across coupled decisions (e.g., selecting a flight constrains hotel check-in time, which constrains dinner reservations)

## Key Technique

DeepPlanning identifies three capability layers that separate effective planners from failing ones. **Layer 1: Proactive Information Acquisition** -- the agent must recognize what information it lacks and actively query for it before committing. The benchmark's error analysis shows "insufficient search" is a dominant failure mode: agents skip critical API calls (e.g., checking seat availability, querying transit times) and then build plans on false assumptions. The fix is to treat information gathering as a first-class planning phase, not an afterthought.

**Layer 2: Local Constrained Reasoning** -- each individual decision must satisfy both explicit constraints (user preferences like "4-star hotel", "brand X only") and implicit environmental constraints (the hotel is fully booked, the attraction closes at 5 PM, only 2 flight seats remain). Models fail here by treating user requirements as the only constraints, ignoring what the environment actually permits. The solution is to validate every choice against both the user's stated preferences and the retrieved environmental state.

**Layer 3: Global Constrained Optimization** -- individual valid choices can still produce an infeasible plan when combined. A hotel check-in at 3 PM is fine locally, but not if the preceding flight lands at 5 PM. A coupon saves money on item A, but using it there prevents applying it to item B where the savings are larger. This layer requires the agent to verify cross-decision consistency (temporal overlaps, budget summation, dependency chains) and backtrack when local optimality conflicts with global feasibility. The DeepPlanning findings show this is where even frontier models fail most often (101 of 120 travel task errors, 52 of 120 shopping errors involve global optimization failures).

## Step-by-Step Workflow

1. **Decompose the planning problem into decision variables and constraint categories.** Identify what must be decided (e.g., which flights, which hotels, which products), what the local constraints are per decision (availability, hours, sizing, preferences), and what global constraints span across decisions (total budget, time continuity, dependency ordering).

2. **Map required information sources for each decision variable.** Before any planning, list every API call or data query needed. For travel: transit times, flight/train schedules, hotel availability, attraction hours, restaurant ratings. For shopping: product catalogs, user profile (size, address), coupon rules, shipping times. Err on the side of over-gathering -- "insufficient search" is the #1 failure mode.

3. **Execute information-gathering calls in parallel where independent.** Group API calls that don't depend on each other and issue them concurrently. For example, querying flight options and hotel options for the same city can happen in parallel; querying hotels for city B must wait until the arrival time in city B is known from the flight selection. This parallel-then-sequential pattern achieves the best effectiveness-efficiency trade-off.

4. **Build a candidate solution skeleton using gathered information.** Create a draft plan structure (day-by-day itinerary, or cart contents) by selecting the most promising option for each decision variable, respecting local constraints. Do not finalize -- this is a draft.

5. **Validate each local decision against explicit AND implicit constraints.** For every choice in the skeleton, verify: (a) it satisfies stated user preferences, (b) it is actually available/feasible given the environment state retrieved in step 3. Flag any violations. Replace violated choices with the next-best feasible alternative.

6. **Run global constraint verification across the entire solution.** Check cross-decision consistency: Do time windows overlap? Does the total cost exceed budget? Are there dependency violations (arriving after a venue closes, selecting a coupon that conflicts with another)? Sum up all costs, verify temporal chains, and check coupling constraints.

7. **If global verification fails, identify the conflicting decisions and backtrack.** Do not patch locally -- re-examine which combination of decisions is globally infeasible. Consider alternative allocations (e.g., swap the order of two attractions, choose a different coupon assignment, pick a later flight to unlock a cheaper hotel). Re-validate from step 5.

8. **Produce the final plan with explicit verification evidence.** Output the plan with per-decision justification: which constraints each choice satisfies, total budget consumed, time feasibility proof. This makes the solution auditable and debuggable.

9. **If building a benchmark or evaluation system, use solution-centric reverse generation.** Design tasks by starting from a known-feasible solution, then generating constraints that make it the unique optimum. Inject environmental constraints (limited availability, closures) to ensure only one valid solution exists, enabling automated rule-based verification instead of LLM-based scoring.

## Concrete Examples

**Example 1: Multi-Day Travel Planning**

User: "Plan a 3-day trip to Tokyo for 2 people. Budget: $2000 total. Must visit TeamLab, Tsukiji Market, and Meiji Shrine. One person is vegetarian. Prefer 4-star hotels."

Approach:
1. Decompose: Decision variables = flights, hotel (3 nights), daily itineraries (attractions + restaurants + transport). Local constraints = vegetarian dining, 4-star hotel, attraction hours. Global constraints = $2000 budget, no time overlaps, transit feasibility.
2. Gather information: Query flight prices/times, 4-star hotel availability and rates, opening hours for TeamLab/Tsukiji/Meiji, vegetarian restaurants near each attraction, transit times between all location pairs.
3. Parallel calls: Flight search + hotel search + all three attraction detail queries run concurrently. Restaurant searches depend on attraction locations, so run after.
4. Skeleton: Day 1 = Tsukiji (morning market, must be early) + TeamLab (afternoon/evening, open late). Day 2 = Meiji Shrine (morning) + free exploration. Day 3 = departure.
5. Local validation: Check Tsukiji hours (some stalls close by noon -- confirmed feasible for morning). TeamLab requires advance tickets -- check availability. Verify hotel has rooms for 2 on all 3 nights.
6. Global validation: Sum costs: flights ($800) + hotel ($450) + attractions ($120) + meals ($300 estimate) + transport ($100) = $1770. Under budget. Verify no time conflicts: Tsukiji 7-11 AM, transit 40min, TeamLab 1-9 PM -- feasible.
7. No backtracking needed.
8. Output structured itinerary with cost breakdown and time verification.

Output:
```
Day 1: Tsukiji Outer Market (7:00-11:00) → [40min transit] → Lunch at Afuri Ramen Shinjuku
        (vegetarian options, 11:40-12:40) → TeamLab Borderless (13:30-18:00)
Day 2: Meiji Shrine (9:00-11:00) → Harajuku exploration → Ain Soph Ripple (vegetarian, lunch)
        → Shibuya afternoon → Hotel
Day 3: Morning free → Airport departure

Budget verification: Flights $800 + Hotel $450 + Attractions $120 + Dining $280 + Transit $95 = $1,745 / $2,000
Time verification: All transitions verified with transit queries. No overlaps detected.
```

**Example 2: Multi-Product Shopping Optimization**

User: "I need running shoes, a waterproof jacket, and a fitness tracker. Budget: $350. I have a 15% off coupon for electronics and a $20-off-$100 coupon for apparel. Optimize total cost."

Approach:
1. Decompose: 3 products to select. Local constraints = must be running shoes (not casual), waterproof jacket (not water-resistant), fitness tracker (not smartwatch). Global constraints = $350 budget, coupon assignment optimization.
2. Gather: Search products in each category, retrieve user size profile if relevant, get coupon applicability rules (can coupons stack? per-item or per-cart?).
3. Parallel: All three product searches + coupon rule query run concurrently.
4. Skeleton: Best-rated running shoes ($120), top waterproof jacket ($150), best fitness tracker ($110). Total before coupons = $380 (over budget).
5. Local validation: All products meet category requirements. Sizes available. But raw total exceeds budget.
6. Global optimization: Apply 15% off to tracker ($110 → $93.50). Apply $20 off to jacket ($150 → $130, since shoes + jacket > $100 in apparel). New total: $120 + $130 + $93.50 = $343.50. Under budget. BUT -- check alternative: $20-off on shoes instead? $100 + $150 + $93.50 = $343.50. Same. Check if cheaper jacket exists: $130 jacket + $20 off = $110, total = $120 + $110 + $93.50 = $323.50. Better.
7. Backtrack to cheaper jacket option. Re-validate: jacket is still waterproof (confirmed). New total $323.50.
8. Output optimized cart with coupon assignments and savings breakdown.

Output:
```
Optimized Cart:
  1. Nike Pegasus 41 Running Shoes — $120.00
  2. Columbia Watertight II Jacket — $130.00 → $110.00 ($20-off-$100 apparel coupon)
  3. Fitbit Inspire 3 Fitness Tracker — $110.00 → $93.50 (15% electronics coupon)

Total: $323.50 / $350.00 budget (saved $46.50 via coupons)
Coupon assignment verified: no stacking conflicts, both coupons applied to eligible categories.
```

**Example 3: Building a Planning Evaluation Benchmark**

User: "I want to create a benchmark to test if my LLM agent can handle constrained planning. How should I design the tasks?"

Approach:
1. Use solution-centric reverse generation: start from a known valid solution, then build constraints around it.
2. Design a toolbox of domain-specific APIs the agent can call (minimum 8-15 covering search, filter, detail retrieval, and cart/itinerary manipulation).
3. Layer constraints: base skeleton (what must be accomplished) → personalized preferences (explicit local constraints) → environmental constraints (implicit, requiring API calls to discover) → budget caps (global).
4. Ensure exactly one optimal solution by tuning environmental constraints (e.g., only one hotel has availability on the required dates at the required star level).
5. Use rule-based automated verification: check each constraint programmatically rather than using LLM-as-judge.

Output:
```
Benchmark Task Template:
  - Scenario: [domain description with 3-5 subtasks]
  - Explicit constraints: [user-stated preferences, 4-8 per task]
  - Implicit constraints: [discoverable only via API calls, 2-4 per task]
  - Global constraints: [budget cap, time window, dependency ordering]
  - Tools available: [list of 8-15 APIs with input/output schemas]
  - Gold solution: [the unique valid plan, used for automated scoring]
  - Scoring: [rule-based checklist across all constraint categories]
```

## Best Practices

- **Do:** Treat information gathering as a mandatory first phase. Query every relevant API before committing to any decision. The #1 error pattern in DeepPlanning is skipping critical searches.
- **Do:** Validate against implicit constraints (availability, capacity, hours), not just explicit user preferences. A choice that matches preferences but violates environmental reality is still wrong.
- **Do:** Use parallel tool calls for independent queries (e.g., searching flights and hotels simultaneously) to reduce interaction turns without sacrificing accuracy.
- **Do:** Always perform a global feasibility check after assembling the full plan. Local validity does not imply global validity.
- **Avoid:** Greedy local optimization without backtracking. Picking the best option at each step often creates globally infeasible or suboptimal plans.
- **Avoid:** Assuming information not retrieved from an API. If you didn't query seat availability, don't assume seats exist. If you didn't check business hours, don't assume a venue is open.
- **Avoid:** Skipping the cost/time summation step. Many plans fail simply because the agent never adds up the total cost or checks for temporal overlaps.

## Error Handling

| Error Pattern | Cause | Fix |
|---|---|---|
| **Insufficient search** | Agent skips API calls, plans on assumptions | Add a pre-planning checklist: list every unknown, query each one |
| **Tool misuse** | Wrong API called or wrong parameters | Validate API schemas before calls; retry with corrected params |
| **Fact displacement** | Agent retrieves info but uses wrong values later | Anchor each decision to the specific retrieved data point; cite sources |
| **Explicit constraint violation** | User preference ignored | Post-check every choice against the original requirement list |
| **Implicit constraint failure** | Environmental limit missed (sold out, closed) | Always query availability/status; never assume from past data |
| **Global optimization failure** | Valid parts, infeasible whole | Run end-to-end verification: sum budgets, check time chains, validate dependencies |

## Limitations

- This methodology is most effective for planning domains with structured, queryable information sources (APIs, databases). It is less applicable to purely creative or open-ended planning where constraints are subjective.
- The backtracking step assumes a manageable solution space. For combinatorially explosive problems (e.g., 50+ interdependent decisions), heuristic pruning or dedicated solvers may be needed alongside this approach.
- Global optimization verification requires enumerating all cross-decision constraints. If the constraint set is not well-defined upfront, the agent may miss interactions.
- The parallel tool-call strategy depends on the execution environment supporting concurrent API calls. In strictly sequential environments, the efficiency gains disappear.
- Benchmark design via reverse generation produces tasks with unique solutions, which may not reflect real-world ambiguity where multiple good-enough plans exist.

## Reference

**Paper:** [DeepPlanning: Benchmarking Long-Horizon Agentic Planning with Verifiable Constraints](https://arxiv.org/abs/2601.18137v1) (Zhang et al., 2026). Focus on Section 4 (error taxonomy: insufficient search, implicit constraint failures, global optimization breakdowns) and Section 5 (the effectiveness-efficiency frontier showing reasoning models with parallel tool use dominate).