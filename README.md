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

```
/plugin marketplace add linhnv/claude-skills
/plugin install deep-pr-review@claude-skills
```

The repo is private, so the first command needs git access to it (an authenticated
`gh` or an SSH key already works).

Then, in any session:

```
/deep-pr-review 1234        # a PR number
/deep-pr-review             # the current diff
```

Cost: about 210 tokens always-on, ~10.5k when the skill actually fires.

<details>
<summary>Without the plugin system</summary>

```bash
git clone git@github.com:linhnv/claude-skills.git
ln -s "$PWD/claude-skills/skills/deep-pr-review" ~/.claude/skills/deep-pr-review
```

Do not do both — a personal copy in `~/.claude/skills/` and the installed plugin
are two separate registrations of the same skill.
</details>

### How well it works

It was measured, not assumed: nine blind trials against the findings of a
commercial AI code reviewer on real pull requests, scored after the fact.
**16 of 30** findings matched — 12/20 runtime defects, 4/8 standards
violations, 0/2 test-quality gaps. It also produced about a dozen findings the
reference reviewer did not raise, and once published a finding whose mechanism
did not exist.

Those numbers cover 0.1.0, which reviewed one repository in isolation. The
cross-repo, stacked-PR and decision-record passes added since have had **no blind
trial**; `references/provenance.md` lists every one of them as unmeasured.

`skills/deep-pr-review/references/provenance.md` has the full method, the
per-trial scores and the known blind spots. Read it before trusting the skill
with anything that matters.

Use it as a second pass alongside an automated reviewer, not instead of one.

## License

MIT — see `LICENSE`.
