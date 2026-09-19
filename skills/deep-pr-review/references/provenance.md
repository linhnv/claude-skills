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

## Honest framing

Use it as a second pass **alongside** an automated reviewer, not instead of one.
It catches a bit over half of what a good commercial reviewer catches, and it
catches some things that reviewer does not. It has also been wrong in public.
