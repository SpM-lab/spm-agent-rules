# Contributing To SpM-lab Agent Rules

## Process

- Rule changes go through a pull request against this repository. No direct
  pushes to the default branch.
- One coherent topic per pull request. Describe the failure that motivated the
  change; if it came from a real defect, describe the defect class, not the
  incident.
- Adding a rule file means adding it to [`rules/index.md`](rules/index.md) and
  to its routing-table row in the same pull request. An unrouted rule file will
  not be loaded.

## What A Rule Must Do

- **Name the failure it prevents.** A rule without a failure mode is a
  preference. Write it as the observable bad outcome: silent zeros, a segfault,
  a wrong answer that looks physical, a green suite that ran nothing.
- **Be imperative and concrete.** "Convert with an explicit dtype, then take
  the pointer from the converted object" — not "be careful with dtypes".
- **Be checkable.** A reviewer or a test must be able to decide whether a diff
  complies. Prefer rules a CI check could enforce, and say so when one could.
- **Be durable and generic.** No line numbers, no file paths into member
  repositories, no narrative about when it was discovered. The rule must still
  read correctly after the code is refactored.
- **Show a violation and its fix** when the distinction is subtle, as a
  miniature code snippet. Keep snippets to the few lines that carry the point.

## Style

- English, imperative mood, short paragraphs, bullet lists.
- Wrap prose at roughly 80 columns.
- Title-case section headings, matching the existing files.
- Keep each file tight. If a file grows past a few screens, split it by topic
  and update the index.
- Use the naming conventions in [`rules/common.md`](rules/common.md) —
  including for the repositories and packages themselves.

## Before Merging

- Cross-reference rather than duplicate: link to the file that owns a rule
  instead of restating it. A duplicated rule drifts.
- If a rule constrains one language only, it belongs in that language file, with
  a link from the topic file.
- If a rule supersedes or narrows an existing one, edit the existing rule.
  Never leave two rules that a reader must reconcile.
- Check that member repositories' `AGENTS.md` files still route correctly if you
  renamed or moved a file; leave a redirect note in the index for one release
  cycle.

## Removing A Rule

State why it is no longer needed: the failure is now impossible by
construction, a CI check enforces it mechanically, or it was never a real
failure mode. Do not remove a rule merely because it is inconvenient.
