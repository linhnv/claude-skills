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

### What it misses

Known from using it, and not fixed by any of the passes added since:

- **Test quality.** A loose mock expectation or a captured-but-unasserted value gets
  read, written into the working notes, and then dropped as "too weak".
- **Small diffs.** Steps already in the method — read the callers, walk every
  standards clause, check the PR is still open — get skipped when a diff looks
  self-contained. More rules do not fix that.
- **It has been wrong in public.** A finding was once published whose mechanism did
  not exist, because a truncated `grep` hid the migration that had removed it.

Use it as a second pass alongside an automated reviewer, not instead of one.

## License

MIT — see `LICENSE`.
