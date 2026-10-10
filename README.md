# Binary Search Algorithm: a template you can write from memory, the three bugs that break it, and which variant fits which problem

Binary search is the algorithm people feel safest about and get wrong most often. Jon Bentley ran it as an exercise in his programming course for professional programmers and found that around 90% of them, given two hours, produced code with a bug in it. That was in *Programming Pearls*, and it wasn't a joke about beginners.

The most famous casualty is the JDK itself. Java's `Arrays.binarySearch` used `mid = (low + high) / 2` for about nine years. Joshua Bloch wrote it up on the Google Research blog in 2006 under the headline "Nearly All Binary Searches and Mergesorts are Broken." The fix is one line. The failure only shows up on arrays longer than 2^30 entries, which is exactly why nobody caught it.

So the interesting question isn't "what is binary search." You probably know the idea: cut the search range in half, repeat. The interesting question is why an algorithm with four moving parts generates so many incorrect implementations, and what the correct set of habits looks like.

## What the algorithm is actually doing

Binary search answers a yes/no question repeatedly until the answer stops flipping. On a sorted array it's "is the target to the right of `mid`?" On something with no array at all it might be "can we ship these packages in 14 days?" The array is an implementation detail. The real precondition is monotonicity: if something is feasible at position `i`, it has to be feasible for everything after it (or before it).

That distinction matters more than it sounds. Almost every "I know binary search, why can't I solve this problem" moment comes from not recognizing that a problem is monotonic. "Minimum eating speed" has no sorted array in it, and it's still binary search.

Cost: O(log n) time, O(1) extra space in the iterative form. A range of one billion elements takes about 30 comparisons. A linear scan takes up to a billion. The gap is why interviewers keep asking.

## The eight lines most problems are built on

This is the exact-match template, closed interval, both ends live:

python
def search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if nums[mid] == target:
            return mid
        if nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1


Four decisions are packed in there, and each one has a right answer that depends on the other three:

1. `mid = lo + (hi - lo) // 2` instead of `(lo + hi) // 2`
2. `while lo <= hi` instead of `lo < hi`
3. `lo = mid + 1` and `hi = mid - 1` instead of `lo = mid` / `hi = mid`
4. `return -1` after the loop, because "not found" is only knowable once the range is empty

Change one without changing the others and you get an infinite loop, an off-by-one, or a wrong answer on the edge case of a single element.

## The three bugs that survive code review

**Integer overflow in the midpoint.** In Java, C++, or any language with fixed-width ints, `lo + hi` overflows once the indices get large enough. `lo + (hi - lo) // 2` never overflows because `hi - lo` is bounded by the array length. Python integers are arbitrary precision so it can't overflow there, but write it the safe way anyway. Interviewers notice, and you'll carry the habit into languages where it matters.

**Loop guard mismatched to the interval.** `while lo <= hi` belongs with a closed interval `[lo, hi]`, where `hi` starts at `len(nums) - 1` and both endpoints are candidates. `while lo < hi` belongs with a half-open interval `[lo, hi)`, where `hi` starts at `len(nums)` and the loop exits when the range holds one element. Mixing the guard from one convention with the initialisation from the other is the single most common way people write binary search that silently returns `-1` for values that are present.

The guard is not a style choice. It follows from the invariant. In the closed version, when `lo == hi` there's still one unprobed element sitting inside the range, and skipping it means declaring the target absent without ever comparing it. That's why the guard has to be `<=`.

**Pointers that don't move past `mid`.** You already compared `nums[mid]` and rejected it. It cannot stay in the candidate range. Writing `lo = mid` on the branch where the target is to the right leaves `mid` in play, and on a two-element range `lo` stops changing entirely. The loop spins until the judge times out.

A quick reference for the conventions:

| Template | Interval | Init | Guard | Update | Returns |
| --- | --- | --- | --- | --- | --- |
| Exact match | closed `[lo, hi]` | `hi = n - 1` | `lo <= hi` | `mid ± 1` | index or `-1` |
| First `>=` (lower bound) | half-open `[lo, hi)` | `hi = n` | `lo < hi` | `hi = mid` | `lo` |
| Predicate / first true | half-open `[lo, hi]` | `hi = upper` | `lo < hi` | `hi = mid` | `lo` |

## Trace it once by hand and the edge cases stop being scary

Take `nums = [-1, 0, 3, 5, 9, 12]`. Search for `9`, which is present:

| Step | lo | hi | mid | nums[mid] | Decision |
| --- | --- | --- | --- | --- | --- |
| 1 | 0 | 5 | 2 | 3 | 3 < 9, discard left half: `lo = 3` |
| 2 | 3 | 5 | 4 | 9 | Match, return 4 |

Now search for `2`, which isn't:

| Step | lo | hi | mid | nums[mid] | Decision |
| --- | --- | --- | --- | --- | --- |
| 1 | 0 | 5 | 2 | 3 | 3 > 2, discard right half: `hi = 1` |
| 2 | 0 | 1 | 0 | -1 | -1 < 2, `lo = 1` |
| 3 | 1 | 1 | 1 | 0 | 0 < 2, `lo = 2` |
| 4 | 2 | 1 | — | — | `lo > hi`, range empty, return `-1` |

Four iterations on six elements. Note step 3: the loop did enter with `lo == hi` and did examine that last element. That's the entire reason the guard is `<=`. If you delete that iteration you'd exit with `lo == 1` and an unexamined `0` sitting in the range.

## The variants that actually show up in interviews

**Lower bound / upper bound.** Which index does the target go in, and what are its first and last positions? This is LeetCode 34 and 35 territory, and it's where the closed-interval template stops being the right tool. The cleaner version uses a half-open range and never returns early:

python
def lower_bound(nums, target):
    lo, hi = 0, len(nums)          # [lo, hi)
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo


Notice `hi = mid`, not `hi = mid - 1`. When `nums[mid] >= target`, `mid` is still a candidate answer, so it stays in the range.

**Binary search on the answer.** This is the variant that trips people up because the input isn't sorted and there may be no array at all. Koko eating bananas, ship packages within D days, split array largest sum. Write the predicate first, then binary search for the smallest value where it turns true:

python
def first_true(lo, hi, feasible):
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if feasible(mid):
            hi = mid          # mid works, look for something smaller
        else:
            lo = mid + 1      # mid fails, everything below fails too
    return lo


The whole difficulty migrates into `feasible`. Once that function is right and monotone, the search is mechanical. Which is also the trap: if `feasible` isn't monotone across the search range, no amount of careful pointer arithmetic saves you.

**Rotated sorted arrays.** Even after rotation, one of the two halves split by `mid` is still fully sorted. Check which one, then ask whether the target falls inside its range. One extra branch on top of the standard loop.

**Real questions from real companies.** Google, Meta, and Amazon all ask binary search variants, but rarely the textbook version. The common shapes are "find a boundary," "search a rotated array," and "binary search is one piece of a harder problem" — the pattern inside a pattern. That second-order recognition is what separates people who know binary search from people who can use it under pressure.

Sidenote: you don't have to hand-roll any of this in production. Python has `bisect.bisect_left`, C++ has `std::lower_bound`, Java has `Arrays.binarySearch`, Go has `sort.Search`. Knowing the library exists is worth something; knowing what it does to your `lo` pointer is worth more in an interview.

## When binary search is the wrong tool

- **The data isn't sorted and you can't afford to sort it.** A hash map gives you O(1) lookup instead. If you're answering one query, sorting at O(n log n) to save one O(n) scan is a net loss.
- **You need more than one match.** Binary search finds a boundary, not a set. If the answer is "all elements between 10 and 50," you still need to scan between the two boundaries.
- **The container has no random access.** Linked lists, streams, and most iterators can't jump to `mid` in O(1), so the halving buys you nothing.
- **n is small.** A linear scan over twelve elements wins on both constants and readability.

## Getting the pattern into your fingers

Reading about the loop guard is not the same as writing it correctly at minute 35 of an interview. The gap between "I understand this" and "I can produce the boundary variant under pressure" is closed by reptition on problems chosen to stress exactly those decisions.

This is where a pattern-oriented platform earns its place. AlgoMonster was built by ex-Google engineers around the observation that interviews recycle a few dozen underlying patterns, and it organizes its curriculum that way rather than as a problem dump. Binary search isn't a single lesson there — it's a pattern with a ladder of variants, from plain exact match, to boundary searches, to using binary search as one component inside a medium or hard problem. The lesson explains why the template works, hands you the reusable version, then drills variations so you stop relearning the pointer updates each time.

A few details worth knowing before you build a study plan around it:

- **48 patterns, 231 lessons and problems, roughly 700 interactive diagrams.** Each pattern is taught the same way: theory, then a code template, then three to five problems with a twist.
- **Templates in Python, Java, JavaScript, C++, Go, and Haskell.** Useful if your target company interviews in something other than Python.
- **Speedrun mode** cycles timed problems with a running clock, which is closer to interview pressure than untimed practice.
- **Decision flowcharts** walk you through categorisation questions ("is the input sorted?", "are you looking for a contiguous subarray?"). For binary search variants, that's the part people fumble at 9pm before an onsite.
- **Company-tagged question sets** for Google, Meta, and Amazon, refreshed monthly.
- **An AI assistant** for hints and concept explanations, unlimited on the paid plans.
- **It's text and diagrams. There is no video.** That's deliberate, and it's also the most common complaint. If you learn by watching someone else code, this is the wrong format.
- **Problem volume is small compared to LeetCode** (231 versus several thousand), and system design coverage is introductory. Most people use it for patterns and LeetCode for volume.

👉 [See how the binary search pattern ladder is structured on AlgoMonster](https://algo.monster?fpr=li69)

## AlgoMonster plans and what each one costs

Pricing on this platform rotates with promotions, and the numbers below reflect the current promotional pricing with the list price alongside. The free tier is real and doesn't need a card, so there's no reason to guess.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems, enough to judge the teaching style | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems; unlimited AI Assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; company questions updated monthly; member discount on live bootcamps; bonus Foundation and Build-an-LLM courses | $99/year (list $198), shown as about $8.25/month billed annually | Annual, cancel or upgrade anytime | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates, and an internal referral to a top tech company for qualified candidates | $189 one-time (list $378) | One payment | [Get Pro Lifetime access](https://algo.monster?fpr=li69) |

There's no monthly plan and no team or enterprise tier. It's annual or lifetime, individual only. Promotions change the headline numbers, so the checkout page is the authority on what you'll actually pay today — if a sale is live, it's already reflected there.

## Which plan actually makes sense

If you're prepping for a loop that's four to eight weeks out, the yearly plan is the sensible one. You'll finish the binary search ladder and the rest of the curriculum inside that window, and paying $189 to own content you'll be done with before the subscription renews is money for a future you haven't committed to.

Lifetime starts making sense if you're the kind of engineer who interviews every two or three years, either chasing a promotion band or testing the market. At the current promotional pricing, the lifetime plan is roughly two years of annual renewals. Past that point it's just cheaper, and the content updates keep landing without you paying again.

The free tier is worth an hour regardless. What you're evaluating isn't the marketing copy, it's whether the text-and-diagram format holds your attention for 40 minutes. If it doesn't, no pricing makes the platform right, and you should go find a video-first alternative.

## Questions people ask about binary search

**Why is it O(log n)?** The candidate range halves every iteration. Starting from n, after k steps you have n/2^k elements left. The loop ends when that's 1, so k = log₂n. A billion elements: about 30 comparisons.

**Does the array have to be sorted?** Something has to be monotone. Sorted is the common case, but a problem like "minimum capacity to ship packages in D days" has no array and is still a valid binary search, because feasibility flips exactly once across the range of capacities.

**Why `lo + (hi - lo) // 2` instead of `(lo + hi) // 2`?** Overflow in fixed-width integer languages, and a well-known historical bug in Java's standard library. Python's integers don't overflow, but the habit transfers to every other language, and interviewers read it as a signal.

**Which template should I memorise?** One, and you should understand a second. Memorise the closed-interval exact-match version because it's the least error-prone for "find this value." Understand the half-open predicate version because it generalises to boundaries, insertion points, and search-on-answer. Those two cover nearly everything.

**Can I pass with binary search alone?** No, and it isn't the interesting question. Binary search appears in interviews as a boundary search, as a rotation variant, or buried inside a harder problem. The useful skill is recognising the shape fast enough to reach for the right template before the clock runs out.

👉 [Learn the binary search pattern properly instead of re-deriving the loop guard](https://algo.monster?fpr=li69)
