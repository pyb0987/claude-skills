# Structure & performance checklist (Phase 1b)

The reviewer walks this list **item by item** and reports only what the diff **creates or makes
worse**. Pre-existing smells the diff doesn't touch are OUT_OF_SCOPE.
Why a named list and not "check the design": agents follow what is literally written. A reviewer
given general principles plus "real problems only" raised 0 findings on an obvious duplicate,
while a named list with definitions cut accumulated smells from 12.6 to 0.2 across three releases
(Kim Min-tae, ["AI 에이전트는 적힌 것을 지킨다"](https://medium.com/@ibare/what-ai-coding-agents-follow-05322c76c4b0), Medium 2026-11).

## A. Code smells (Fowler, *Refactoring* 2nd ed., ch. 3)

| # | Smell | Flag when |
|---|---|---|
| 1 | Mysterious name | the name doesn't say what it does or how to use it |
| 2 | Duplicated code | the same code shape appears in more than one place |
| 3 | Long function | one function does several jobs |
| 4 | Long parameter list | many parameters, often passed through unchanged |
| 5 | Global data | data anyone can read and change from anywhere |
| 6 | Mutable data | the same data is changed in several places |
| 7 | Divergent change | one module changes for several unrelated reasons |
| 8 | Shotgun surgery | one change needs small edits in many modules |
| 9 | Feature envy | a function uses another module's data more than its own |
| 10 | Data clumps | values that always travel together aren't one type |
| 11 | Primitive obsession | a domain concept is a raw string/number instead of a type |
| 12 | Repeated switches | the same condition (`kind === X`) branches in several places |
| 13 | Loops | a loop that reads better as a map/filter pipeline |
| 14 | Lazy element | a structure that does almost nothing |
| 15 | Speculative generality | an extension point nothing needs yet |
| 16 | Temporary field | a field that is only filled in some cases |
| 17 | Message chains | `a.b().c().d()` navigation |
| 18 | Middle man | a class that only forwards calls |
| 19 | Insider trading | modules swap internal data too freely |
| 20 | Large class | a class or module that does too much |
| 21 | Alternative classes, different interfaces | same role, different method shapes |
| 22 | Data class | data here, the behavior on it somewhere else |
| 23 | Refused bequest | a subclass ignores most of what it inherits |
| 24 | Comments | a comment that explains confusing code instead of fixing it |

## B. Project-specific

Add your own recurring patterns to `local.md` (section "Extra smells"); the reviewer gets both.
Good candidates are the smells your reviews keep finding by hand, e.g. "a new enum value needs a
branch in more than one file".

## C. Performance

| # | Pattern | How to prove it (execution where possible) |
|---|---|---|
| P1 | N+1: a query inside a loop or per-field resolver | count queries in an integration test (ORM query log, e.g. prisma `$on('query')`) for n=1 vs n=10 |
| P2 | Unbounded `findMany` / list with no `take` on a growing table | cite the table and the caller that can list it all |
| P3 | `await` inside a loop where calls are independent | cite the loop; show they share no state |
| P4 | New `where` / `orderBy` on a column with no index | cite the query + the schema's index list |
| P5 | Work repeated per request that could run once (parse, compile, fetch config) | cite the call site and its call frequency |
| P6 | Full-list re-render or re-fetch on a small change (FE) | cite the dependency array / query key |

## Finding format

Instead of a failure scenario, a structure finding carries a **change scenario**:
the realistic next change and every place it would have to touch.

```
{id, kind:"structure"|"perf", smell:"8 Shotgun surgery", file, line,
 sites:["a.ts:40","b.ts:88","c.ts:12"],
 scenario:"adding the next POS vendor edits all three sites",
 severity}
```

No `sites` → not a finding. "Could be cleaner" is still not a finding.
