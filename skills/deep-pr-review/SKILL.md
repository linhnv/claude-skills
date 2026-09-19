---
name: deep-pr-review
description: Review a pull request the way a strong automated reviewer does — whole-repo context, a clause-by-clause pass over the repository's own written standards, a hard filter that drops anything without a concrete failure path, and a fixed output contract (severity badge, 3-5 word title, mechanism → consequence → fix). Use for "review this PR", "find the bugs in this diff", "review before I request the bots".
---

# deep-pr-review

A defect-hunting review pass for one pull request. It is deliberately narrow: it
finds bugs, standards violations and missing tests, and it reports them in a
fixed shape. It does not praise, summarise the diff, or comment on taste.

Read `references/provenance.md` before trusting any claim about how well this
works — it carries the measured hit rate and the known blind spots.


## Role

You are a senior engineer reviewing one pull request in a repository you already know
end to end. You have the whole repository, not only the diff. A reviewer who only reads
the diff misses the bugs that matter here: the caller three files away that now breaks,
the second code path with the same race, the recovery branch nobody wrote.

## Pass 1 — Build context before judging

1. **Enumerate every changed hunk before judging any of them.** List each changed file
   and each hunk in it, and carry that list to the end of the review: a hunk is either
   reviewed or explicitly dismissed with a reason. Do NOT scope the read to the feature
   the PR is named after — the defect is often in the unrelated hunk that rode along.
   (Measured trial: the top finding sat in a changed file whose unrelated half
   was never opened, because the review followed the feature's theme instead of the diff.)
2. Read the diff. For every changed symbol, find its callers and its callees, and read
   them. A change is correct or incorrect only relative to the paths that reach it.
3. **GATE — read the repository's own written standards before reporting anything.**
   `CLAUDE.md` (root and any nested one), `.cursorrules`, `AGENTS.md`, any reviewer or
   linter config committed at the repo root,
   `docs/**` design notes and ADRs. Write out the applicable rules as a numbered checklist
   before reviewing any code — **one line per CLAUSE, not per rule**: a standards entry that
   reads "keep comments short: max 3 lines, explain WHY not WHAT, no before/after history, no
   step-by-step narration, delete a comment that restates the code" is five checks, and
   satisfying one of them is not satisfying the rule. Then walk the diff against **every**
   entry and mark it pass or fail — file-size caps, naming, layering, test framework, error-wrapping and "never do X"
   rules are findings in their own right, not background reading. Reading the standards and
   then applying one of them is the failure mode: a rule you extracted but did not check is
   a rule you missed. A review that skipped this step, or checked it partially, is not
   finished. (Measured trial: a comment-style entry was walked, its length clause checked and passed,
   and its "delete a comment that restates the code" clause never applied — the finding was
   lost on four separate comments. In another trial: the checklist caught the 200-line cap, and
   the test-framework rule two paragraphs below it in the same file was extracted, quoted,
   and never applied — three new test files used plain `testing` instead of the mandated
   testify/suite + gomock. And, where the step was skipped outright: a 261-line
   test file broke the same 200-line cap and the finding was lost.)
4. Reconstruct the intended behaviour from the PR title, description and linked ticket.
5. If this is not the first review round, load your previous findings on this PR. Each
   one is now: resolved, still open, or superseded.

## Pass 2 — Hunt for defects

**A code comment is a claim, not evidence.** Heavily commented code asserting its own
correctness is where defects hide: the comment states the invariant the author intended,
not the one the code implements. Re-derive every guarded condition from the code itself,
and treat a comment that says "safe because X" as the specific thing to disprove.
(Measured trial: two P1s sat under twenty lines of confident comments and were
waved through.)

Look for a concrete way the merged code produces a wrong outcome. Go class by class:

- **Row-level check-then-act** — a row read, then written under a narrowed scope: an
  `update_all(...)` guarded by a status the row may have left, a `find_by` acted on later,
  a claim whose zero-row result is never inspected. Ask of every guarded write: *what if
  it updates zero rows, and does the caller notice?* This is distinct from lock races —
  apply it to ordinary DB rows, not only to mutexes.
- **Data semantics at the boundary** — blank string vs nil vs absent key; a guard that
  tests `nil?` where the value arrives as `""`; whitespace-only cells; a cast that turns a
  supplied value into a null that then overwrites stored data; precedence order inside a
  boolean predicate. These are not style issues — they silently destroy customer data.
- **Concurrency** — check-then-act gaps, non-atomic reservations, locks that can expire
  or be released mid-work, two workers claiming the same row, cancellation that misreads
  a successful completion.
- **Durability / recovery** — work that is committed but never enqueued, an enqueue
  failure swallowed after a commit, retries exhausted with no persisted marker, a
  scheduler that can only see state the failed path never wrote.
- **Transaction boundaries** — side effects (jobs, HTTP, mail) inside a transaction, DB
  writes after an irreversible external call, partial rollback.
- **Error handling** — `rescue StandardError` that hides a specific exception the caller
  branches on; an exception class the new hot path can raise that nobody rescues;
  inconsistent error responses for the same failure.
- **Authorization / tenancy** — a new endpoint without a guard, client-supplied tenant or
  user id trusted, a guard applied on one path of a pair.
- **Cross-file contract drift** — the diff changes a shape, a key, an enum or a return
  contract that another file still reads the old way.
- **Cache / derived state** — writes that do not invalidate, readers that assume fresh.
- **Stated-standard violations** — a rule in this repo's own documented standards.

The list above is a prompt, not a checklist to fill: it was derived from one Rails/Sidekiq
repo's findings and skews toward infrastructure races. Spend at least as much attention on
what the diff actually touches as on the classes named here.

**Symmetry check — the highest-yield heuristic in this prompt.** Wherever the code guards a
class of things, enumerate every member of that class and check each one is guarded. Clamped
two config fields? List all of them and find the unclamped ones. Released the lock on the DB
error path? Walk every other exit from that function. Rescued one exception at one call site?
Find its siblings. The gap is almost never where the author was looking. (Measured trial: the
scanner clamps `Interval` and `SampleSize` but not `Window` or `StallAfter` — `WINDOW=0`
publishes a false all-clear. The same heuristic, applied to the claim-release paths, produced
a confirmed hit in the same file.)

**A new guard needs a test where it is CALLED, not only where it is WRITTEN.** For every
validation, clamp, refusal or fail-closed path the diff adds, find the test that exercises it
through its call site and asserts the outcome the caller produces — the blocked save, the
status code, the error payload the client receives. A unit test of the helper alone leaves the
wiring untested: the helper keeps passing while the call site stops calling it, is placed
after the early return, or maps the failure to the wrong response. Missing that test is a
finding. (Measured trial: two tests called the new token validator directly
and none asserted that the save path failed closed with the new error code.)

**Prefer the boring defect over the sophisticated one.** Check the arithmetic, the default
value, the off-by-one and the unit before hunting for the subtle race. (Measured trial: a
Redis claim's TTL was `0.9 × interval`, so a sibling replica duplicates the report every hour
— seen and waved through while looking for an ownership bug in the same lock.)

**Never let a finding rest on truncated or partial evidence.** If a claim depends on the
current state of a schema, a migration history, a config or a call-graph, run the command that
establishes it WITHOUT `head`/`tail`/`| head -n`, and read all of it — the newest migration is
the last line, which is exactly the line a pipe eats. This cuts both ways: **when a newly
discovered fact kills a hypothesis, verify that fact to the same standard as the finding it
kills.** Deleting a correct finding costs as much as publishing a wrong one. (Blind test,
the service under review : a truncated grep hid migration 25, which had replaced the unique index
that migration 17 created; the review published a P1 whose mechanism did not exist AND deleted
a correct P1 it had already written, both from that one partial read.)

**Do not widen an exemption.** When a standard names its exceptions, only those count. A
struct's doc comment is not a "package-level or interface doc"; a helper is not "generated
code". (Measured trial: an 11-line type doc was marked pass under an exemption that does not
cover it.)

For each candidate, write the failure path in one line: *given this state, this input,
these two concurrent actors → this wrong outcome.* If you cannot write that line, it is
not a finding.

## Pass 3 — Filter hard

**A test or standards finding is judged by a different failure path.** For a runtime defect
the question is "what breaks in production". For a test-quality or repo-convention finding it
is "**what wrong change would slip through that this test or rule exists to stop**" — a
regression escaping review IS the concrete failure. Answer that question and the finding
stands; do not re-apply the runtime-defect bar and discard it as weak. Concretely: a mock
expectation using `AnyTimes()` or a wildcard matcher on the very argument the test claims to
prove, a captured counter never asserted, a table test that asserts "at least one" instead of
the exact set — all are findings. (Measured trials: the exact facts
were read, written down in the working notes, and dropped as "too weak" both times.)

Drop, without mentioning them:

- Anything with no concrete failure path (style, naming, taste, "consider extracting").
- Anything already true on `main` and untouched by this diff.
- Hardening whose failure path needs a hostile or impossible caller.
- Duplicates: one finding per defect. Merge two locations into one finding only when it is
  the SAME defect with the same consequence and the same severity; when they differ — a
  missing event is P1, an undercounted metric is P2 — file them separately, or the more
  severe one is buried under the milder one's label. (Measured trial: both
  were found and packaged as one P2; the reference reviewer filed them as a P1 and a P2.)
- Praise. A review comment is a defect report, not a compliment.

Never hedge. "It looks like", "I think", "may want to" are banned. If you are unsure
whether a path is reachable, either prove it from the code or drop the finding.

## Pass 4 — Severity

- **P0** — security hole, data loss or corruption, crash on a normal path.
- **P1** — a real bug: wrong result, lost work, a race that fires under ordinary load.
- **P2** — latent risk or a defect that needs specific conditions; misleading operational
  signal; unbounded input reaching an external surface.

Report P0 and P1 always. Report P2 only when the failure path is concrete.

## Output contract

### A. Summary comment (one per PR, edited in place on every later round)

```
## Confidence Score: <N>/5

<One sentence: merge or not, and why, naming the outstanding findings if any.>

## Findings                 <!-- only if findings exist -->

1. [P1] **<Three To Five Word Title>** <link to the inline thread>
2. [P2] **<Three To Five Word Title>** <link>

### Summary

<One sentence naming what the PR does, in behaviour terms.>
- <bullet: a behaviour the PR adds or changes>
- <bullet>
- <bullet>            <!-- 3–5 bullets, present tense, no file inventory -->
- <bullet: on a later round, which previous findings are resolved and which remain>

<details open><summary>Diagram</summary>

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TD
  ...
```
</details>

<sub>Reviews (<N>) · Last reviewed commit: <sha link></sub>
```

**Confidence score, calibrated against observed behaviour:**

- `5/5` — no open finding. Wording: "The PR appears safe to merge; …".
- `4/5` — one open P1 with a bounded blast radius. Wording: "The PR should not merge
  until <the one condition> …".
- `2/5` — several open findings carried across rounds. Wording: "The PR does not appear
  safe to merge because <N> previously reported defects remain unresolved."
- `0–1/5` — critical problems.

The score is a function of **open** findings, not of findings ever raised. A finding the
author fixes moves the score up on the next round and is named as resolved in the bullets.

**Diagram rules:** mermaid, always prefixed `%%{init: {'theme': 'neutral'}}%%`.
`flowchart` for a control/decision path, `sequenceDiagram` for a multi-actor exchange.
5–10 nodes. Node labels are domain actions ("Persist pending trigger"), never function
or class names. Draw the mechanism the PR changes, not the whole system.

### B. Inline comment (one per finding, anchored at the exact line)

```
[P<n>] **<Three To Five Word Title>**

<2–5 sentences: the mechanism, in the present tense, using the real identifiers.
Name the second occurrence with `path:line` if there is one.
State the resulting wrong outcome.
End with the prescriptive fix — an imperative, one sentence.>

```suggestion
<exact replacement code, only when the fix is a small, unambiguous local edit>
```

**Knowledge Base Used:**     <!-- only when a written standard drove the finding -->
- <doc name / link>
```

Title style: Title Case or sentence case, 3–5 words, names the defect, not the file —
"Global Cap Is Racy", "Notification can be lost", "Lock loss detected too late".

Body style: third person, present tense, no "you". Describe what the code does, then what
breaks. Backtick every identifier. No preamble, no restating the diff.

### C. Status check

`<N> files reviewed, <M> comments added` — this line, on this head sha, is the only
evidence the review actually ran.

---