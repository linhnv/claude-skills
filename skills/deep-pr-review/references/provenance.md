# Provenance and measured limits

## What this is

A behavioural reconstruction. It was built by reading the output of a strong
commercial AI code reviewer across ~20 real pull requests — the shape of its
summaries, the shape of its inline comments, which findings it raised and, just
as informative, which it stayed silent about — and then writing down the method
that would produce that behaviour.

It is **not** anyone's actual system prompt. No proprietary text was copied. The
one thing taken verbatim from the product is nothing more than what it prints in
every public PR comment, and it was left out of the skill entirely.

## How well it works

Nine blind trials. Procedure, per trial:

1. Fetch the reviewer's findings for the PR and lock them in a file, unread.
2. Check out the exact commit the reviewer reviewed, so every finding it raised
   is findable and none has already been fixed.
3. Review with this skill only, and write the findings to a second file.
4. Reveal and score.

| Finding class | Matched |
|---|---|
| Runtime code defects | 12 / 20 |
| Repository-standard violations | 4 / 8 |
| Test-quality gaps | 0 / 2 |
| **Total** | **16 / 30** |

Per-trial: 0/3, 1/1, 2/5, 3/4, 3/4, 3/4, 3/4, 0/3, 1/2. The improvement over
the run is real but partly reflects the reviewer learning the specific
repositories; do not read 16/30 as a stable rate on a fresh codebase.

It also produced roughly a dozen findings the commercial reviewer did not
raise — among them an index migration that would lock a hot table for the length
of a deploy, a coalesced trigger that was silently dropped on an enqueue
failure, and two whole branches of new code with no test. Those are unverified:
nobody confirmed them, so count them as leads, not wins.

## Known blind spots

- **Test quality: 0 for 2.** Loose mock expectations and captured-but-unasserted
  values were read, written down in the working notes, and dismissed as "too
  weak" both times. The filter below is calibrated for runtime defects and
  misfires here; the skill now says so explicitly, but the fix is unproven.
- **Skipped steps on small diffs.** Three of the misses were rules already in
  the method — read the callers, walk every standards clause, enumerate the
  whole enum — skipped because the diff "looked self-contained". More rules will
  not fix that.
- **One wrong finding shipped.** A P1 was published whose mechanism did not
  exist, because a `git grep | head` hid the migration that had replaced the
  constraint the finding rested on — and the same partial read caused a correct
  finding to be deleted. Truncated evidence is the most expensive mistake in
  this method.

## Changes after the trials (unmeasured)

The nine trials above were run against version 0.1.0, which reviewed one repository in
isolation. Three passes were added afterwards from real review sessions on a multi-repo
product (a Rails API with a web front end, an iOS app that cannot be force-updated, and an
export service), plus one conduct section. **None of them has been through a blind trial.**
Treat the hit rate above as measured only for the parts of the method that produced it.

- **Pass 1.5 (consumers).** Added because the two most consequential findings in those
  sessions were both in a companion repository, not in the diff: an unguarded dereference of
  a field the API had just made nullable, and an iOS save path whose one-line delete rule
  settled a disputed BLOCKER. Neither is visible from the API repository.
- **Pass 1.6 (stacks and epics).** Added because reviewing a stacked PR against the trunk
  reports the parent's work as the child's, and because half a mechanism in an open sibling
  reads as a defect until you name the ship unit.
- **Stale head.** Added after a measurement was published in a draft against a head the
  author had already replaced with the fix. Same class as the truncated-read failure that
  cost a wrong P1 in trial 8, and cheaper to avoid: re-read the head sha before writing.
- **Composition.** Added to settle the overlap with a host review workflow. The split is
  deliberate and was chosen by the user of this plugin: the output contract stays here
  (confidence score, mermaid diagram, inline `suggestion` threads, one review not N
  comments), and conduct - whose turn it is, existing threads, language, and never posting
  until asked - belongs to the host workflow.

### Second batch: extracted from a private review workflow (also unmeasured)

The rules below were lifted from a repository-specific review workflow that had accumulated
them from real incidents, and rewritten to drop everything product-specific. The incidents
behind them are real; the wording here is anonymised, and none of these has been through a
blind trial either.

- **Pass 1 items 6 and 7** - confirm the PR is open, fetch status separately from the
  description, and verify what the description claims.
- **Pass 1.5 items 6 to 8** - compare how both sides compare a value; check whether the
  consumer re-reads after writing; search for consumers you cannot clone.
- **Pass 1.7** - read the decision record before calling anything a gap.
- **Pass 2** - the "work that grows with the data" class, the "repair at the call site"
  class, and the rule to verify by executing rather than remembering.
- **Pass 3** - scope discipline, findings the consumer already makes unreachable, and the
  ban on asserting a gate whose tool is not enabled.
- **B3** - never publish local test results or individual commit shas.
- **Conduct** - establish whose turn it is from the review-request timeline.
- **The findings log** - the mechanism only: when to file an entry, how to title it, and the
  rules that keep the file greppable. The entries themselves stay in the repository that
  produced them, which is the whole point of the split.

## Honest framing

Use it as a second pass **alongside** an automated reviewer, not instead of one.
It catches a bit over half of what a good commercial reviewer catches, and it
catches some things that reviewer does not. It has also been wrong in public.
