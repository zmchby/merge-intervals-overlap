# Merge Intervals: A LeetCode 56 Walkthrough, the Overlap Rule People Get Wrong, and the Pattern Behind Insert Interval and Meeting Rooms

"Merge intervals" is one of those problems that looks like a five-line answer and still eats 20 minutes in an interview. The core idea is short — sort by start time, then sweep — but the details are where candidates lose points: whether `[1,2]` and `[2,3]` count as overlapping, whether you used `max()` when extending, whether touching intervals collapse into one.

This walkthrough covers the full solution for LeetCode 56, the edge cases interviewers actually probe, the three related problems that reuse the same mental model, and where structured pattern practice (AlgoMonster, in this case) fits if grinding random problems isn't producing results.

## What the problem is really asking

You get an array where each element is a two-value interval, `intervals[i] = [start, end]`. Return a set of non-overlapping intervals that covers every range in the input.


Input:  intervals = [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]


`[1,3]` and `[2,6]` overlap between 2 and 3, so they collapse into `[1,6]`. The other two intervals don't touch anything and pass through unchanged.

The second sample is the one that matters more than it looks:


Input:  intervals = [[1,4],[4,5]]
Output: [[1,5]]


Those intervals share exactly one point: 4. LeetCode treats that as an overlap, so they merge. That single sample decides your comparison operator — `start <= currentEnd` instead of `start < currentEnd` — and it's the most common silent bug in submitted solutions.

Constraints are small enough that you don't need to be clever about memory: up to 10⁴ intervals, each with `0 <= start <= end <= 10⁴`. That also tells you the input isn't guaranteed to be sorted, which is the whole reason the solution starts with a sort.

## Why sorting by start makes this a one-pass problem

Sort the intervals by their start value and an important property falls out: any intervals that can be merged are now next to each other. You never have to look backward, and you never have to compare a new interval against more than the most recent merged one.

Walk through `[[1,3],[8,10],[2,6],[15,18],[5,7]]`:

1. Sort by start: `[[1,3],[2,6],[5,7],[8,10],[15,18]]`
2. Seed the result with the first interval: `merged = [[1,3]]`
3. `[2,6]`: is `2 <= 3`? Yes → extend to `[1, max(3,6)] = [1,6]`
4. `[5,7]`: is `5 <= 6`? Yes → extend to `[1, max(6,7)] = [1,7]`
5. `[8,10]`: is `8 <= 7`? No → push `[1,7]`, start fresh with `[8,10]`
6. `[15,18]`: is `15 <= 10`? No → push `[8,10]`, start `[15,18]`
7. Loop ends, push the final interval.

Result: `[[1,7],[8,10],[15,18]]`. Five intervals became three.

### The transitive overlap trap

Step 4 is the part that separates a correct solution from a plausible-looking one. `[1,3]` doesn't overlap `[5,7]` at all — 3 < 5. But `[2,6]` sits between them and bridges the gap, so the true merged range is `[1,7]`.

If you compare each new interval only against the *original* interval that started the current merge, you'll produce `[[1,6],[8,10],...]` and drop the `[5,7]` merge entirely. Comparing against the **current merged end** (which grows as you go) is what makes the transitive case work. That's the invariant worth saying out loud in an interview:

> The last interval in the result is the fully merged representation of everything seen so far.

## The code

Python, in full:

python
class Solution:
    def merge(self, intervals: List[List[int]]) -> List[List[int]]:
        if not intervals:
            return []

        intervals.sort(key=lambda x: x[0])
        merged = [intervals[0]]

        for start, end in intervals[1:]:
            if start <= merged[-1][1]:
                merged[-1][1] = max(merged[-1][1], end)
            else:
                merged.append([start, end])

        return merged


Two lines carry the logic. `start <= merged[-1][1]` is the overlap check (equality included, per sample 2). `max(merged[-1][1], end)` handles nesting — `[1,10]` followed by `[2,3]` should stay `[1,10]`, and assigning `end` directly would shrink it to `[1,3]`.

One practical note that rarely comes up in interviews but shows up in code review: `merged = [intervals[0]]` puts a reference to the input's first interval into your output, so extending it mutates the caller's array. Seed with `intervals[0][:]` if the input needs to stay intact.

Java and C++ versions are the same shape — sort with a comparator on `start`, iterate from index 1, mutate the tail of the result list. In JavaScript, `intervals.sort((a, b) => a[0] - b[0])` rather than the default sort, which compares intervals as strings and will quietly misorder them.

### Other accepted approaches, and when they matter

The sort-and-scan version is what interviewers expect. Two alternatives show up in published explanations:

- **Event sweep line.** Turn each interval into a start event (+1) and an end event (−1) in a sorted map. Sweep boundaries in order; when the active count returns to zero, you've closed a merged interval. Useful if you're already thinking in terms of "how many things are active right now," which is the same reasoning behind Meeting Rooms II.
- **Greedy "farthest end" array.** Record the farthest end per start position, then scan positions left to right. It's bounded by the coordinate range, so it only makes sense when coordinates are small — as they are here (≤ 10⁴). It's a nice demonstration that you understood the problem, not an upgrade you'd ship.

The brute-force baseline is worth mentioning before the optimal: repeatedly scan for overlapping pairs, merge one pair, restart. That's O(n²) worst case with a chain of overlapping intervals. Starting there and explaining why sorting eliminates the repeated passes is the standard interview arc.

## Complexity

- **Time:** O(n log n), dominated by the sort. The merge pass is O(n) — one comparison per interval, no nested loops.
- **Space:** O(n) for the output in the worst case (no overlaps, every interval survives). Ignoring the output, the algorithm uses O(1) extra space beyond the sort.

Worth knowing: people do push back on the O(1) claim, and that argument shows up in AlgoMonster's own discussion threads on this problem, where a commenter points out that once you allocate a result array, the "constant extra space" framing only holds if you exclude the output. If you say "O(1) excluding the result," nobody can argue with you.

## Edge cases to run before you say you're done

- **Empty input** — return `[]` before touching `intervals[0]`.
- **Single interval** — the loop body never runs; the seed is already the answer.
- **Touching intervals** — `[[1,4],[4,5]]` → `[[1,5]]`. Confirms your `<=`.
- **Nested intervals** — `[[1,10],[2,3]]` → `[[1,10]]` if you used `max`.
- **Duplicates** — `[[1,4],[1,4]]` → `[[1,4]]`.
- **Fully disjoint** — `[[1,2],[4,5]]` → unchanged; tests that you're not merging on `start > end` boundaries alone.
- **Unsorted input** — the four-interval sample above is unsorted on purpose.

The bug list is short and predictable: strict `<` in the overlap check, forgetting `max`, comparing against the original interval instead of the grown merged end, appending the previous interval too early (so the last one never lands), and assuming the input is sorted because it "looks" sorted in the example.

## The same pattern, six more problems

Merge intervals is a template, not a one-off. Once you own sort-by-start plus overlap handling, these become variations rather than new problems:

| Problem | What changes | The move |
| --- | --- | --- |
| Insert Interval (57) | Input is already sorted; you're adding one interval | Skip, merge the overlap, append the rest — O(n) possible without re-sorting |
| Non-overlapping Intervals (435) | You need the *minimum removals* | Greedy on end times, not starts |
| Meeting Rooms (252) / Meeting Rooms II (253) | Detect any conflict / count concurrent rooms | Sort by start and compare against a running end, or use a min-heap of end times |
| Interval List Intersections (986) | Two sorted interval lists | Two pointers, advance whichever ends first |
| Remove Covered Intervals (1288) | Count intervals not contained in another | Sort by start ascending, end descending |
| Minimum Number of Arrows (452) | Count points hitting every interval | Same greedy-by-end instinct as 435 |

Notice how much 57 leans on 56. Because the input is already sorted, re-sorting is wasted work, and a linear pass is achievable — a point raised repeatedly in the platform's discussion thread on that problem, where people post binary-search plus two-pointer variants to get there. If you can explain why 56 needs a sort and 57 doesn't, you can explain the difference between a template and its application, which is exactly what interviewers are listening for.

## How to actually present this in an interview

The coding is maybe 40% of the signal. The rest is the sequence:

1. **Clarify overlap semantics first.** Do `[1,2]` and `[2,3]` merge? The answer changes your comparison operator, and asking shows you know it's a decision, not a detail.
2. **Confirm whether the input is sorted.** If it is, skipping the sort is a real complexity win.
3. **State the brute force and its cost** — O(n²), repeated passes.
4. **Name the insight:** sorted by start, mergeable intervals are adjacent, so one pass suffices.
5. **Write the code**, then say the invariant out loud: *the last merged interval always covers everything processed so far.*
6. **Walk the touching and nested cases manually.** Don't wait for the interviewer to ask.

If you can do those six things under pressure, you have the pattern. The trouble is that most people learn it by accidentally hitting the problem, failing the transitive case, and reading the answer — which doesn't generalize to 57, 435, or 253.

## Where structured pattern practice fits

If your situation is "I can solve merge intervals today but couldn't derive it cold tomorrow," the gap isn't effort, it's missing structure. AlgoMonster is one of the platforms built specifically around that gap: it organizes interview prep by pattern rather than by problem count, and intervals is one of the patterns with its own entry.

A few things about it that are verifiable and relevant here:

- The LeetCode 56 write-up on AlgoMonster walks the solution in explicit steps (sort, seed, extend with `max`, append the final interval) and includes an example walkthrough — the same structure as this article, which is a reasonable sign that the editorial is teaching process rather than just dumping code.
- The platform's decision flowchart has an interval branch: given a problem description, it routes you toward "sorting plus interval scan" versus other families (binary search on sorted input, hash tables, and so on). That's the generalization step most people skip.
- Its discussion forum has active threads on both Merge Intervals and Insert Interval where users argue about space complexity and post O(n) alternatives — useful if you want to see how other candidates reason about the same code.
- It's subscription-free in structure: no monthly plan, just yearly or a one-time lifetime purchase.

👉 [Check current AlgoMonster plan prices and the active sale](https://algo.monster?fpr=li69)

### Plans and pricing

Pricing on the site moves with whatever promotion is running, so treat the numbers below as what the official pricing page displays rather than a permanent figure. Third-party coupon blogs quote different totals ($99/year, $189 lifetime, $459, and so on) — that spread is a good reason to check the checkout page itself instead of trusting any single review.

| Plan | Price | Billing | What you get | Get it |
| --- | --- | --- | --- | --- |
| Free | $0 | No billing | Limited access to a subset of lessons and problems — enough to test the teaching style | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | $69.99/year (shown as about $5.84/month billed annually) | Recurring, cancel or upgrade anytime | All patterns, lessons and problems; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; company questions updated monthly; unlimited Speedrun and AI assistant; 12 months of access; member discount on live bootcamps | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro – Lifetime | $89 | One-time payment | Everything in the annual plan, plus lifetime updates and permanent access to future courses | [Buy Pro Lifetime once](https://algo.monster?fpr=li69) |

The gap between the two paid plans is the actual decision. Yearly is roughly $70; lifetime is $89. If the lifetime price still holds at checkout, the break-even against a single year is about four months of use — which makes lifetime the obvious pick unless you're certain you'll finish in one sitting and never return. Interview prep has a way of recurring every two or three years.

There's a promotion-code field at checkout, and third-party sites circulate codes claiming extra percentages off on top of the sale price. Those codes change and expire without notice, so enter one if you have it, then judge the final number on the payment screen rather than on any blog's table.

### What it won't do

Worth saying plainly, because most reviews of this platform hedge: the content is text and diagrams, not video. There are no live mock interviews, and the system design material is introductory — it won't carry a senior-level system design loop on its own. The problem count is also far smaller than LeetCode's, which is the point (patterns over volume) but is still a real tradeoff if what you want is a giant bank to grind.

The honest split is: use a platform like this to learn *why* merge intervals generalizes to insert interval and meeting rooms, then use LeetCode for volume and speed on company-tagged sets. Neither fills the other's role.

👉 [See what's included before you pay](https://algo.monster?fpr=li69)

## FAQ

**Does merge intervals require sorting?**
For LeetCode 56, yes — the input isn't guaranteed sorted, and sorting is what makes mergeable intervals adjacent. If the input arrives sorted (as in Insert Interval), the sort is unnecessary and a linear pass is enough.

**What's the time complexity?**
O(n log n), dominated by the sort; the merge pass itself is O(n).

**Why use `max` when extending?**
Because an interval can be fully contained in the current merged range. `[1,10]` followed by `[2,3]` stays `[1,10]` — plain assignment would produce `[1,3]`.

**Do `[1,2]` and `[2,3]` merge?**
In LeetCode 56, yes. They share the point 2, and sample 2 (`[[1,4],[4,5]]` → `[[1,5]]`) confirms it. Always confirm this with your interviewer anyway, since some problem variants define it differently.

**Is it worth paying for pattern-based prep just for intervals?**
No — not for one problem. The case for a paid platform is the whole interval family plus the other 40-something patterns, and the fact that you'll interview again. If you're two weeks out from a loop, the free tier plus consistent LeetCode practice is the better allocation of time and money.

**Is there a monthly option?**
Not currently. The plans shown are yearly and one-time lifetime.

The short version of this whole article: sort by start, keep one growing merged interval, use `<=` and `max`, and remember that the last result entry is entirely merged for everything you've seen. Say the invariant out loud and merge intervals stops being a trick question.
