---
name: prove-it
description: "Use before saying any task is done, fixed, verified, passing or shipped, and before merging work that several agents built. Every claim gets an artifact (a path, an exit code, a status, a screenshot, a quoted line) or it is NOT PROVEN, and each rule names the real failure that produced it: reviews that ran once, suites that collected zero tests, a fix measured locally and broken in production, a screenshot that dropped touch emulation."
---

# Prove it: the verification gate

**Target: bugs in the delivered result, not honesty about it.** A completion check catches
"we said done and it isn't". It does not catch "we did it and it's wrong". Large runs fail
the second way, in the seams no single agent was responsible for.

## 1. One agent reviews the whole diff

In a fan-out every agent verifies its own shard and nothing verifies the integration. The
seams are where the bugs are: two agents editing the same file, incompatible assumptions
across a contract, a replaced code path left live beside its replacement, an import nobody
updated.

End every run with a review of the **complete aggregate diff** by an agent that wrote none
of it, hunting correctness bugs rather than completion. Give it the diff and the original
ask, never the implementers' reports. Use a review command or a reviewer subagent with a fixed brief instead of improvising a prompt.

**Loop it until it comes back clean.** One review is the floor, not the ceiling. Every fix
round gets its own review of the new diff (the same reviewer is fine: it still wrote none
of it), and the work merges only when a review returns no P0 to P2. Anything left at P3 is
fixed or named in the report. Measured 2026-09-26 on a production app's mobile header: four
reviews across the fix rounds, three of which found P1 or P2 defects the builder had
missed, the last one a regression that the previous fix round itself introduced. A review
that runs once, on the first diff, cannot see a defect a later round adds.

## 2. Run it, and prove the tests ran

Build the app and run the real suite **after the whole run**, not per shard. Zero tests
collected is a FAILURE, not a pass (the tests-ran skill in this pack enforces it).

A per-agent green says the new thing works. It never says the old things still work.
Regressions are exactly the landmines that surface weeks later.

**No shard may close a run red because the break belongs to another shard.** Measured: four
parallel agents each correctly diagnosed the same two failing tests, each correctly
attributed them to a sibling's change, and all four declined to fix them as out of
ownership. The run shipped red with every member knowing. Reconciliation is the aggregate
phase's job and it must be somebody's.

## 3. Count the sweep

If the ask was "every place X appears", state the number found, the number changed, and
reconcile the gap by name. "Updated the call sites" with no count is how 9 of 12 ships.
If the run capped coverage (top-N, sampling, no-retry), say what was dropped: silent
truncation reads as full coverage when it was not.

## 4. Verify against the stranger's surface

Deployed URL not source. Shipped bundle not commit. The branch that deploys, not the
branch you are on. The public record, not the confirmation page. "I could not reach it" is
NOT PROVEN, never a pass. For anything irreversible or outward-facing, use three verifiers
with distinct lenses and require a majority.

**A tool result is a report too, and an interrupted command may already have run.**
Measured 2026-10-20 (a production app): a `git push origin origin/main:production` ran 4
seconds into its tool call, the operator interrupted 12 seconds later, and the harness answered
"rejected". The session then told the operator and a peer that nothing was pushed, and a peer found
the deploy an hour later in GitHub's activity log. Before saying a side effect did NOT
happen, read the remote it would have changed (`git reflog show refs/remotes/origin/<branch>`,
`/api/health`, the provider's list), never the tool's status line.

**Name the auth state and the viewport, or the verdict is not a verdict.** "Verified"
hides which surface you stood on. Measured 2026-08-23: a 16-item UI wave shipped reported
as verified when every browser check in it had been signed out, because the logged-out
tool was picked once and never revisited as the task grew. Several of those items only
manifest signed in, so the pass could not have caught them even in principle. Write
"verified signed out at 390px" or "verified signed in on production", per item, and list
the items checked in neither state.

**Check every state, both ways.** Recurrence 2026-10-16 in the opposite direction: three
public SEO pages were screenshotted signed in only and offered as checked; signed out, the
header differed and a free tool page read "0 of 5 free generations used today, Sign
up for unlimited" where signed in read "Unlimited generations". Any surface with more than
one rendered state (auth, plan tier, empty or full, new or returning, viewport) gets one
looked-at screenshot per state. A signed-out pass never closes a signed-in item, and a
signed-in pass never closes a signed-out one. The standing order: zero assumptions when validating.

**An occluded window silently suspends requestAnimationFrame.** Measured 2026-08-23: a
production morph animation probed through browser automation read as frozen at every
sample while the identical code visibly animated on dev, because the automation window
sat on another Space: `visibilityState: "hidden"`, zero rAF frames in 1500ms, while
screenshots kept capturing normally. Forty minutes went to bisecting builds before the
window state was checked. Any animation or rAF-dependent assertion must name
`document.visibilityState` alongside viewport and auth state, and a zero-rAF probe is
the first check when motion "does not work" through automation.

**A screenshot is evidence only of the state it was captured in.** Measured 2026-10-18 on a news site: a
Chromium `fullPage` screenshot drops touch emulation for the shot, `(pointer: coarse)` turns false, and every
44px touch target renders at its desktop size. A judge swarm then confirmed "footer links too small to tap" on
14 of 47 findings, while the live links measured 44px tall in both engines. Phone evidence is captured by
scrolling the real viewport, and the capture asserts `matchMedia("(pointer: coarse)")` before a phone tile is
believed. The same holds for auth: a tile proves its state only if the session was checked at the end of the
capture, not the start.

**Coverage words are counts.** "Every page" means the number checked out of the number that exist, with
anything sampled, capped or skipped named next to it. Same day: a swarm reported as covering every page had
judged 34 of 281, at most 12 screens a page, and skipped 1,729 of 1,952 Safari tiles behind an engine cutoff
labelled "measured" that nobody had measured. A tool that judges coverage must refuse to write its verdict
unless judged plus skipped equals the total.

**A local green is not evidence about production, and neither is staging.** Same day: a layout-shift fix measured
CLS 0.0309 locally under two different throttling profiles and 0.794 on production, and
was reported fixed on the strength of the local number. The defect depended on data
arriving after layout rather than on being slow, so throttling could not reproduce it. If
the claim is about production, the measurement is taken on production. Recurrence
2026-09-29 (a production app): a "speed" lane shipped five pushes on staging acceptance runs while
production server latency sat at p50 about 400ms and p90 about 1.2s for two weeks, unread; staging
answered robots.txt in 0.08s against production's 0.35s (different container placement, DB
distance, load). The founder found it by hand. A speed claim reads production's own Workers wall-time
percentiles before and after, never staging's.

## 5. Look at the artifact yourself, and fix the class

**The orchestrator opens the evidence, never just the report.** Read the screenshot, the
diff and the output before accepting "fixed". Measured 2026-09-26: round 1 reported the
header fixed and its own after-shot still showed the band. Same page, same state, before
and after, side by side.

**When the ask is to truly prevent something, land a guard for the class, not a fix for
the instance.** A sweep test that fails on any new occurrence anywhere in the tree, proven
red first per the verification discipline below. Measured the same day: beyond the
reported header, a sweep found 12 more bars positioned the same way; the review had named
9 of them.

**A builder's side effect beyond the ask is reverted, not shipped.** A mobile fix that
also restyled desktop went back to the original look on desktop. The bar rises; the scope does not widen.

## 6. Wire the swarm peer to peer; the orchestrator is not a relay

Measured 2026-09-27 on a production app's launch: 9 agents looping through the lead sent roughly
45 messages in two hours, most restating a report already received, and every fix waited
for the lead to read, retype and forward it. The order that followed: be far more efficient.

- **Name the loop in every brief, at spawn.** Who sends findings to whom, who reruns, and
  what done means: code is clean at P0 to P2 with the gates green; QA is zero findings at
  every severity.
- **Agents talk to each other directly.** A builder sends its whole diff to the reviewer,
  the reviewer returns findings to the builder, and QA routes each finding to its owner and
  runs the next pass itself once the work goes quiet.
- **One message up per unit of work.** Either ship-ready (PR, sha, gate exit codes,
  verdict, evidence paths) or blocked on a decision only a human can make. Also escalate a
  finding that survives 2 fix rounds. No interim status, no idle summaries.
- **A merged PR's branch is dead.** Before any push to a PR branch, check
  `gh pr view <n> --json state`; a follow-up to a merged PR is a new branch off
  origin/main and a new PR. Measured 2026-09-28: three builders pushed reviewed fixes to
  already-merged branches in one night, and none of those commits would have shipped.
- **Scope stays with the builder.** Fix anything in the class you were given and list
  what you left; do not ask the lead whether it is in scope.
- **Cutting out the relay never cuts out the look.** The orchestrator still opens the final
  evidence itself (section 5) and still owns the aggregate review (section 1).
- **The human hears results and decisions only.** Batch pending gates into one numbered list they can answer in one message; never narrate an agent's duplicate report.

## 7. The verification discipline

- **A build that strips types is NOT a typecheck.** Run `tsc --noEmit` (or `npm run check`) as its own step. NEVER grep its output for a subset of filenames; read it whole or assert the exit code. Never silence an error with `as`, a widened type, or `@ts-ignore`.
- **A check that returns nothing is BROKEN until proven otherwise.** Before trusting "0 matches", confirm the same check finds a known-present case. Assert what must NOT have changed alongside what must.
- **A number you did not PRODUCE this turn is a quote, not a measurement.** Re-run it or label it inherited.
- **A test or a guard is not evidence until it has FAILED.** Break what it protects, `diff` the break LANDED (a no-op edit reads as a fake guard), go red, restore, `diff` identical. A suite green after a substantive edit means no test covers it, and a guard that fails OPEN fails into your most confident answer.
- **Verify the stranger's surface**: deployed URL not source, shipped bundle not commit, public record not confirmation page. "Could not reach it" is NOT PROVEN.
- **A DELTA CANNOT SEE AN ALWAYS-BROKEN PAGE.** Assert an ABSOLUTE floor on a SCHEDULE: status, title, an h1, a body minimum, one sitemap listing each.

## Verdict

`PROVEN <artifact>` or `NOT PROVEN <the check that failed>` per claim, plus what you left
unchecked. An artifact is a path, a status code, an exit code, a screenshot, or a quoted
line of output. "Verified", "should work", "no issues found" with nothing attached is a FAIL.

**NOT PROVEN must name a check that failed.** Doubt you cannot state concretely is noise.

**Judge the work, not your memory of a past incident.** Prior postmortems tell you where to
look. They are never evidence about the artifact in front of you.

---

From the operating rules dotcomjack.com is built on: one founder, many Claude Code sessions, a few dozen products. Each rule above names the failure that produced it. https://dotcomjack.com/skills#kit
