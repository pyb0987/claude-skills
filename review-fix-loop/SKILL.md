---
name: review-fix-loop
description: "Review a PR or branch with an independent subagent, adversarially verify each finding, fix confirmed ones with a separate fixer agent (test-first), then re-review with a fresh reviewer until clean or capped. Use when the user says 'review and fix this PR', 'review-fix loop', '/review-fix-loop <PR>', or wants a reviewed-and-fixed PR without doing the back-and-forth by hand. On a teammate's PR it switches to feedback mode: posts verified review comments, then re-checks them after the teammate pushes ('review his PR', 're-review after his changes'). Watch mode polls a PR (default every 30m, max 5 checks) for approvals, comments or pushes ('watch my PR', 'check if they approved')."
---

# Review → Verify → Fix → Re-review

Input: a PR as `owner/repo#N` or a URL (or a branch), plus the local worktree it lives in.
Output: fix commits on the PR branch (local until the user approves a push) and a
round-by-round report of findings: confirmed / refuted / out-of-scope / fixed.

## Why it is shaped this way (keep these — each one is load-bearing)

| Rule | Evidence |
|---|---|
| Reviewer ≠ fixer ≠ orchestrator; every reviewer starts with **fresh context** | Self-repair is bottlenecked by the quality of the critique, not the repair, and a stronger/separate critic is what moves it (Olausson et al., ICLR 2024). Models rate their own output higher (self-preference bias, NeurIPS 2024). |
| Every finding is **verified adversarially before anyone fixes it** | Frontier reviewers sit at ~0.72–0.78 precision — about 1 flag in 4 is wrong. A refute-or-promote gate is the precision lever (arXiv 2604.19049). |
| Verification is **by execution** wherever possible (a failing test, a repro) | Feedback grounded in execution beats feedback from introspection; intrinsic self-correction without external signal is unreliable (Huang et al. 2023; Kamoi et al. survey 2024). |
| Re-review reads the **whole diff**, not only the fix commits | A cross-model critic flagged errors in ~16% of fixes a same-family reviewer had approved (arXiv 2604.19049): fixes introduce new bugs. |
| Hard cap of 3 rounds; stop on the first round with 0 confirmed findings | Self-repair gains are modest once cost is counted and vary a lot by task (Olausson et al.). Loops past that mostly churn. |
| Structure/perf is a **separate lane** with a named checklist, its own evidence standard (citation, not a failing test) and its own fixer | The bug lane's rules ("real problems only", failure scenario, refute by test, no refactors) filter design out by construction: 2% of 90 review findings were structural. A named list with definitions, not general principles, is what moved smells (12.6 → 0.2); cleanup run as its own task fixed 0.68 smells per new one vs 0.16 when mixed with feature work (Kim Min-tae, Medium 2026-11 — `smells.md`). |

Known ceiling: every agent here is the same model family, so self-preference is reduced
(fresh context, adversarial role, executable checks) but not removed. If a different-family
reviewer is available (e.g. a Codex CLI), use it for round-2+ reviews.

## Phase 0 — Setup (orchestrator, you)

1. Resolve the PR: `gh pr view <N> -R <owner/repo> --json headRefName,baseRefName,url,body,author,headRefOid`.
   A bare number resolves against the **cwd's** repo, which is usually the wrong one: ask which repo,
   and never guess. Every later `gh` call takes the same `-R`.
   **Mode:** if `author.login` ≠ `gh api user -q .login`, use **Teammate mode** (end of this file):
   you never commit to their branch. Phases 1–2 are shared; Teammate mode replaces 3–5.
2. Find or create the **edit worktree** for the head branch (follow the repo's worktree rules —
   never edit the shared checkout). `git fetch` the base first. Teammate mode only needs a
   detached worktree at the PR head (`git fetch origin pull/<N>/head`) to run tests in.
3. Write the review packet to the scratchpad: `git diff origin/<base>...HEAD`, the PR body,
   and the test command for the touched area. Agents read the packet; they do not get your
   conversation or your opinion of the code.
   **Spec grounding.** Add the PR body's reviewer-focus section (the author's own questions) and
   the spec the PR implements (ticket, design doc, acceptance criteria) if one is linked. If
   [`local.md`](./local.md) exists beside this file, follow its grounding steps too: it holds the
   project-specific lookups (and any overrides of the defaults below).
   No spec found → say so in the report and review without one. Don't guess one.
4. Record the baseline: run the touched area's tests once; save pass/fail counts.

## Phase 1 — Review (subagent, read-only, fresh)

Spawn one `general-purpose` agent (inherit the session model — this is judgment, not collection).
Prompt essentials:

- The packet path. "You did not write this code. Assume it has bugs; find them."
- Scope = correctness, data loss, security, concurrency, error paths, and **behavior the PR
  description claims but the code does not deliver**. Style/naming only if it hides a bug.
- With spec grounding: answer each of the author's reviewer-focus questions from the spec
  (cite the criterion id or doc §), name the acceptance criteria the diff leaves uncovered, and
  flag spec details the code no longer matches because it took a different shape
  (a spec fix, not a code bug: put it in the report, not in a blocking comment).
- Each finding must carry: `file:line`, one-sentence defect, a **concrete failure scenario**
  (inputs/state → wrong result), and severity (blocker / major / minor).
- No finding without a failure scenario. "Could be cleaner" is not a finding.
- Return JSON only: `{"findings":[{id,file,line,defect,scenario,severity}]}`.

Round 2+: same prompt, **new agent**, packet = the full current diff vs base. Do not pass
earlier findings (anchoring); dedupe afterwards yourself.

## Phase 1b — Structure & performance review (subagent, read-only, fresh, same message as Phase 1)

A second, separate reviewer, so the bug lane's "failure scenario only" rule doesn't filter it out.
Prompt essentials:

- The packet path plus [`smells.md`](./smells.md) (and `local.md`'s "Extra smells" if it exists). "Go through every item of the checklist against
  this diff, one by one. Report only smells the diff creates or makes worse."
- Do **not** add "only real problems" or "only if it matters": that sentence is what made reviewers
  report nothing. The `sites` requirement is the precision filter instead.
- Each finding uses the `smells.md` format: `kind`, `smell`, `sites` (every location), and a
  **change scenario** (the realistic next change and what it would touch). Perf findings add how
  to prove it (P1–P6).
- Return JSON only: `{"findings":[…]}`.

Round 2+: new agent, full current diff, and one extra instruction: check whether a smell fixed in
an earlier round came back **in a different shape** (agents follow the literal wording — a banned
"temporary field" reappeared as a delimited string). Pass the smell names fixed so far, not the findings.

## Phase 2 — Verify (one subagent per finding, parallel, fresh)

For each finding spawn a verifier in one message (cap ~6; batch the minors into one agent).
Prompt: "Your job is to REFUTE this finding. Read the code. If you can, demonstrate it by
writing a failing test or a repro in the scratchpad, never in the worktree. Verdict:
CONFIRMED (with the failing test or the exact code path), REFUTED (with the line that
disproves it), or OUT_OF_SCOPE (real, but pre-existing or outside what the PR changes)."

Keep only CONFIRMED. OUT_OF_SCOPE goes to the report and is not fixed. Tight scope beats
completeness here.

**Structure findings** (batch them into one verifier): a test can't fail on duplication, so the
evidence is the citation. CONFIRMED only if (1) every listed site exists and does what is claimed,
(2) this diff introduced or worsened it (`git diff origin/<base>...HEAD` touches those lines), and
(3) the named smell's definition really applies. **Perf findings** are verified by execution where
`smells.md` says how (P1 query count n=1 vs n=10); otherwise by citation like structure.

## Phase 3 — Fix (one fixer subagent, edits the worktree)

Spawn one `general-purpose` agent with **only** the confirmed findings and their failing
tests, not the reviewer's prose. Rules for it:

1. Per finding: add the failing test → see it fail → minimal fix → see it pass. One commit per
   finding (`fix: <why>`), so a regression can be bisected to one change.
2. Touch only what the finding needs. No refactors, no drive-by cleanups.
3. If a fix needs a design decision or changes public behavior, stop and return
   `NEEDS_HUMAN` with the question. Do not guess.
4. Run the touched-area tests; report counts vs the baseline.

The orchestrator checks the fixer's claims itself: `git log`, re-run the tests, compare with
the baseline. A fixer report is not proof.

## Phase 3b — Cleanup (separate fixer subagent, after Phase 3)

Confirmed structure/perf findings go to a **new** agent, after the bug fixes land. Mixing
cleanup into feature or bug work is where it doesn't happen. Rules for it:

1. Behavior-preserving only: `refactor: <why>` (or `perf: <why>`), one commit per finding.
   Existing tests stay unchanged and green. A perf fix keeps its P1 query-count test.
2. **Net lines must not grow** for a structure fix (`git diff --shortstat` per commit). Reviewed
   code tends to come out 1.5–1.8× larger, and the cost of the next change follows code size.
   If the only fix adds lines, return `NEEDS_HUMAN` with the trade-off.
3. Changes a public interface or needs a design call → `NEEDS_HUMAN`. No guessing.
4. Skip it when the PR is a hotfix (`base=main`): report the findings, don't refactor production fixes.

## Phase 4 — Re-review and stop rule

Go back to Phase 1 + 1b with new reviewers. Stop when:
- a round produces **0 confirmed** findings → done;
- the **same finding** (same file/defect) is confirmed twice → oscillation, so escalate to the user;
- round 3 is finished → stop and report what is left;
- anything returns `NEEDS_HUMAN`.

## Phase 5 — Report and hand-off

Report per round (then write the Run log line, below): findings raised / confirmed / refuted / out-of-scope / fixed, plus test
counts vs baseline. Re-read the PR body against the final diff: fixes routinely make its
claims false (in one dry run a removed retry gate left "fires once per order" in
the body). Draft the corrected lines, and refuted findings that show a claim was too strong
("idempotent"), for the user to approve. Then **ask** before any outward action: pushing the fix commits,
commenting on the PR, re-requesting reviewers. Out-of-scope findings become one line each
in the report, or a follow-up proposal if they are real work.

## Run log — last step of every run, both modes

Append **one JSON line** to `~/.claude/skills/review-fix-loop/runs.jsonl` (create it if missing),
after the report and after the user has answered the outward-action question. Teammate mode logs
one line per round (per head SHA). The no-change T0 exit logs nothing.
This is the evidence for changing this skill: reviewer precision on our own PRs, whether the
3-round cap ever binds, and where the user had to step in. A rule change should cite lines from it.

```json
{"date":"2026-11-24","pr":"acme/api#1449","mode":"author|teammate","card":"TICKET-153|null",
 "rounds":[{"raised":5,"confirmed":2,"refuted":2,"out_of_scope":1,"fixed":2,
             "by_kind":{"bug":{"raised":3,"confirmed":1},"structure":{"raised":1,"confirmed":1},"perf":{"raised":1,"confirmed":0}},
             "net_lines_cleanup":-14}],
 "tests":{"baseline":"412/0","final":"414/0"},
 "stop":"clean|cap|oscillation|needs_human|user_stop|approve|comments_posted",
 "pushed":false,
 "overrides":["user rejected confirmed finding F3: intended behavior per spec §2"],
 "friction":["bare PR number resolved to wrong repo"]}
```

- `overrides`: every time the user disagreed with a verdict (reopened a REFUTED, dropped a
  CONFIRMED, edited a drafted comment) or corrected the procedure. Quote what they said, in short.
- `friction`: any step that failed, was skipped, or needed a workaround (packet too big, test
  command unknown, `gh` silent failure, verifier repro that didn't run outside the scratchpad).
- `by_kind` is how we see whether the structure lane changes our own mix (the 2% chart was the
  article author's repo, not ours). `net_lines_cleanup` = summed shortstat of Phase 3b commits.
- Empty arrays are fine. Write it with `jq -nc` or `python -c 'import json…'`, never by hand-concatenating strings.

## Model allocation

Reviewer, verifier and fixer all inherit the session model (judgment stages). Only pure
collection (e.g. gathering call sites for the packet) may use `model: sonnet`.

## Teammate mode — the author fixes, you give feedback and re-check

The teammate is the fixer. **GitHub review threads are the state**, so a re-run days later
in a fresh session picks up where the last one stopped. Don't keep a local state file.

**T0 — Load the previous round.**
- Last reviewed SHA: `gh api repos/{o}/{r}/pulls/<N>/reviews --paginate` → your last review's `commit_id`.
- Your open threads: GraphQL `pullRequest.reviewThreads{isResolved path line comments{author{login} body databaseId}}`,
  keep the unresolved ones whose first comment is yours.
- No previous review → round 1: run Phases 1–2, then go to T2.
- Head SHA == last reviewed SHA and **every open thread's latest comment is yours** → say "no change
  since <sha>" in one line and stop. This keeps `/loop` polling cheap. Key it on the latest comment,
  not "replied since my last review": in one run a reply older than my last review went unanswered,
  and round 3 then told the teammate "no reply yet".
- Last reviewed SHA unreachable after fetch (force-push) → skip the incremental diff and only do the full review.

**T1 — Re-check your open threads (round 2+).** One fresh verifier per thread (batch the minors
together), all in one message. Give each one the original comment (including its repro), the teammate's
replies and `git diff <last>..<head>`. It runs the comment's repro at the new head first.
Verdict: FIXED (repro passes or the code path is gone), NOT_FIXED (cite the line), or for a
teammate's pushback, ACCEPT (their reason holds) / REBUT (with evidence). "Fixed" in a reply
is not proof. The repro is. Then run Phases 1–2 on the full current diff with a **new**
reviewer for new findings (don't pass it the old ones), and dedupe against all your threads, resolved ones included: a concern closed as ACCEPT comes back only with new evidence.

**T2 — Draft, ask, post.** Comments use the team's language and tone (`local.md` may set it;
default: the language of the PR). One per confirmed finding:

```
**[blocker|major|minor]** <defect in one line>
- Scenario: <input/state → wrong result>
- Repro: <the verifier's failing test or repro, short>
- Suggestion: <direction only, not a full patch>
```

- The repro in the comment is what T1 runs next round, so it must work outside your scratchpad.
- Structure/perf findings use the same template with `Scenario: <next change → sites it must edit>` and
  `Repro:` replaced by `Sites: <sites>` (perf with a query count: keep `Repro:`). Default severity is
  minor; major only when the change scenario is already planned (in the spec or the next release).
- REFUTED findings are never posted. OUT_OF_SCOPE gets at most one line ("FYI, outside this PR") in the review body.
- Thread replies: FIXED/ACCEPT → short ack + resolve (`resolveReviewThread`). NOT_FIXED/REBUT → reply with the evidence.
- `event`: an open blocker/major → `REQUEST_CHANGES`; only minors → `COMMENT`; nothing open and nothing new → `APPROVE`.
- Show the user the whole draft (counts + every comment) and **ask before posting**. Post the
  review in one call: `gh api repos/{o}/{r}/pulls/<N>/reviews --input review.json` with
  `commit_id`=head, `event`, `body`, `comments:[{path,line,side:"RIGHT",body}]`. A line outside
  the diff can't take an inline comment, so put that one in `body`. Replies go to `.../pulls/<N>/comments/<id>/replies`, where `<id>` is the thread's **first** comment (GitHub rejects replies to replies).
  Re-read the PR afterwards: `gh` writes can fail silently.

**Stop rule.** One round per head SHA, since the teammate's push starts the next one. If the same
thread is NOT_FIXED/REBUT two rounds in a row, stop commenting and tell the user to talk it through.
The loop is done when an `APPROVE` is posted.

**Closing the loop.** Re-run `/review-fix-loop <owner/repo>#<N>` after the teammate pushes, or use
Watch mode (below). End after APPROVE.

## Watch mode — poll a PR until something happens (both modes)

Trigger: "watch this PR", "check every 30m/1h if they approved/updated", or `--watch`.
Default 30 min, **max 5 checks**, then stop and say so. The user may give 1h or another cap.

**State lives in the tick prompt**, not a file: each tick schedules the next one as a one-shot
`CronCreate` (`recurring: false`, minute/hour/day/month pinned to now + interval, off the :00/:30 marks) with
`/review-fix-loop <owner/repo>#<N> --watch <k>/<max> --every <30m|1h> --since <ISO-8601 of the previous check>`.
No recurring job, so nothing to forget to delete; the cap is just "k == max → don't schedule".
Jobs are session-only: if the session closes, the watch dies. Say this once when starting.

**Each tick:**
1. `gh pr view <N> -R <o/r> --json state,headRefOid,reviewDecision,author`. `MERGED`/`CLOSED` → report, stop.
2. Look only at activity by people other than you (`gh api user -q .login`) after `--since`:
   - Your PR: `gh api repos/{o}/{r}/pulls/<N>/reviews --paginate` (`submitted_at`, `state`),
     `.../pulls/<N>/comments --paginate` and `.../issues/<N>/comments --paginate` (`created_at`),
     commits pushed by others (`gh pr view --json commits`, `committedDate`).
   - Teammate's PR: T0's no-change check is the test (new head SHA, or a thread whose latest comment isn't yours).
3. **Nothing new** → one line ("watch k/max: no change since <since>"), schedule tick k+1 unless k == max.
   Mark it a no-op; don't write a run-log line.
4. **Something new** → stop watching (don't schedule) and act:
   - Your PR, `APPROVED` → report who approved; ask whether to merge (never merge on your own; the
     repo's approval rules still apply; `local.md` may point to them).
   - Your PR, `CHANGES_REQUESTED` or new comments/commits → summarize each with its link and ask whether to
     run author mode on the reviewer's points (the reviewer's comments go in the packet as findings, Phase 2 verifies them).
   - Teammate's PR updated → run the normal teammate round (T1–T2). Posting still waits for the user's yes.
   Send a `PushNotification` (if the tool exists) with a one-line summary, since the user is likely away.
5. Hitting the cap with no change → "no activity after <max> checks (<span>)", offer to restart the watch.
