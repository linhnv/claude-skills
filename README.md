# claude-skills

Claude Code skills I use. One skill per directory under `skills/`.

## `deep-pr-review`

A defect-hunting review pass for a single pull request. It is deliberately
narrow: it looks for bugs, repository-standard violations and missing tests,
and it reports them in a fixed shape — severity, a three-to-five word title,
then mechanism → consequence → fix. It does not praise, summarise the diff, or
comment on taste.

What it does differently from asking for "a code review":

- **Reads the whole repository, not the diff.** Callers and callees of every
  changed symbol are part of the review; a change is only correct relative to
  the paths that reach it.
- **Walks the repo's own written standards clause by clause.** `CLAUDE.md`,
  `.cursorrules`, ADRs. A rule with five clauses is five checks, and a rule you
  extracted but never applied is a rule you missed.
- **Filters hard.** Every finding must name the state, the actors and the wrong
  outcome. No hedging, no duplicates, no style notes.
- **Judges tests by a different bar.** For a test or convention finding the
  question is not "what breaks in production" but "what wrong change would slip
  through that this test exists to stop".

### Install

```bash
git clone git@github.com:linhnv/claude-skills.git
cp -R claude-skills/skills/deep-pr-review ~/.claude/skills/
```

Then in Claude Code:

```
/deep-pr-review 1234        # a PR number
/deep-pr-review             # the current diff
```

To keep it updated in place, symlink instead of copying:

```bash
ln -s "$PWD/claude-skills/skills/deep-pr-review" ~/.claude/skills/deep-pr-review
```

### How well it works

It was measured, not assumed: nine blind trials against the findings of a
commercial AI code reviewer on real pull requests, scored after the fact.
**16 of 30** findings matched — 12/20 runtime defects, 4/8 standards
violations, 0/2 test-quality gaps. It also produced about a dozen findings the
reference reviewer did not raise, and once published a finding whose mechanism
did not exist.

`skills/deep-pr-review/references/provenance.md` has the full method, the
per-trial scores and the known blind spots. Read it before trusting the skill
with anything that matters.

Use it as a second pass alongside an automated reviewer, not instead of one.

## License

MIT — see `LICENSE`.
