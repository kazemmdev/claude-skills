---
name: git-branch
description: it's a git branch conven
---

# Branching models

Two units of work, nested:

- A **branch** is a unit of *deliverable scope* — something reviewed and merged
  as a single decision.
- A **commit** is a unit of *completed intent* inside it, not a pile of files
  that happened to change at the same time.

Someone reading `git log` six months from now should be able to follow the
reasoning of whoever built the thing, one decision at a time. Someone bisecting
a bug should land on a commit that actually builds. Someone reviewing a branch
should be able to hold all of it in their head at once.

That is the whole point. Everything below serves it.

## Choosing a branch

Do this before writing code where possible. If work has already started on
`main`, it is still recoverable — see the end of this section.

### Does this work need its own branch?

Check what the project actually does before deciding. `git branch -a` and
`git log --graph --oneline -20` show whether the repo lives on a single trunk or
routes everything through branches.

If the project merges through pull requests, branch. If it is a solo repo with a
linear trunk and the change is trivial — a typo, a comment, a version bump —
committing on `main` is honest and fine. When in doubt, branch: an unnecessary
branch costs one merge, while an unwanted commit on a shared `main` costs a
revert and an apology.

### Size the branch to the work, not to the feature

This is the part people get wrong. A big feature does not mean a big branch — it
means *more* branches. Branch lifetime should be measured in hours or days, not
weeks. Merge debt grows faster than linearly with time: every day a branch stays
open, `main` drifts further, conflicts multiply, and the review gets harder.

| Scope of work | Example | Approach |
| --- | --- | --- |
| Trivial, minutes | typo, comment, dependency bump | Direct on `main` if the project allows, otherwise a short branch |
| One coherent change, hours to a day | fix a bug, add an endpoint, add a migration | One branch, a few atomic commits, one review |
| A feature with parts, a few days | password login: model, hashing, session, endpoints | Still one branch *if* a reviewer can absorb the diff in one sitting — roughly a few hundred lines |
| A capability spanning weeks | "add authentication" end to end | Never one branch. Slice into independently mergeable branches that each land on `main` |

Authentication is the canonical example of the last row. Do not open
`feat/authentication` and disappear for three weeks. Slice it:

```
feat/auth-user-model         → review → merge
feat/auth-password-login     → review → merge
feat/auth-session-management → review → merge
feat/auth-google-oauth       → review → merge
feat/auth-rbac               → review → merge
```

Each slice is boring on its own, which is exactly what makes it reviewable. The
usual objection — that half-built authentication should not sit on `main` — has
a straightforward answer: code can be present without being reachable. Leave it
unrouted, disabled in config, or behind a feature flag. Unreachable code on
`main` is far cheaper than a three-week branch.

Slice along seams that stand up alone: a data model, then behaviour on top of it,
then the surface that exposes it. If a slice cannot build and pass its tests by
itself, the seam is in the wrong place — move the seam rather than merging the
slices back together.

### Naming

Use `<type>/<short-kebab-summary>`, with the same type vocabulary as the commit
prefixes so the branch and its commits agree:

```
feat/auth-google-oauth
fix/session-expiry-off-by-one
refactor/extract-tenant-scope
chore/bump-go-1-26
doc/adr-vector-storage
```

Include the ticket when the project tracks work that way:
`feat/AUTH-142-google-oauth`. Keep it lowercase, hyphenated, and short enough to
read in a branch list. Check `git branch -a` first — matching what the repo
already does beats this convention.

Avoid names that mean nothing to anyone else: `wip`, `test`, `new-branch`,
`fixes`, or your own name. The branch list is a shared surface.

### Creating it

```bash
git switch main && git pull      # start from current code, not stale code
git switch -c feat/auth-google-oauth
```

Only branch off another branch when deliberately stacking dependent work — and
then land them in order, parent first.

### Work already started on `main`

Uncommitted changes follow you across a switch, so nothing is lost:

```bash
git switch -c feat/auth-google-oauth
```

If commits have already landed on local `main` and should not be there, move the
branch pointer — but only when those commits have not been pushed:

```bash
git branch feat/auth-google-oauth   # capture the current position first
git reset --hard origin/main        # rewind main
git switch feat/auth-google-oauth
```

`--hard` discards working-tree changes. Confirm with the user before running it,
and verify the commits are genuinely captured on the new branch first.

## The workflow

### 1. Survey before you stage anything

```bash
git status --short
git diff --stat
git log --oneline -20
```

Look at what actually changed before deciding anything. Never reach for
`git add -A` and a single message — that destroys the information the history
exists to carry.

Watch for changes you did not make. If the working tree already had unrelated
edits when you started, do not sweep them into your commits. Point them out and
ask what to do with them.

### 2. Match the repository's existing conventions

The `git log` you just read is the style guide. Before inventing anything, note:

- Which prefixes the project already uses (`doc:` vs `docs:`, `chore:` vs
  `build:`), and whether scopes appear (`feat(auth):`)
- Subject casing, and whether subjects end with a period
- Whether bodies and trailers are common or the project is subject-only

Consistency with the surrounding history beats any external standard. A repo
that has used `doc:` for two years should keep using `doc:`.

### 3. Group changes into missions

Ask of each file: *what decision does this belong to?* Group by the intent that
produced the change, not by directory, file type, or the order you happened to
write things.

Signals you have the grouping right:

- The subject line describes the whole commit without needing the word "and"
- Someone could revert this one commit and get a coherent partial rollback
- The diff can be reviewed in one sitting without context switching

Signals you should split:

- The message wants to be "add X and fix unrelated Y"
- A pure rename or reformat is mixed in with logic changes — those bury the real
  diff under noise and deserve their own commit
- Generated or vendored files (lockfiles from an unrelated upgrade, build output)
  are riding along with hand-written code

Signals you should *not* split: a change that only makes sense as a whole. A new
function plus its test plus the wiring that calls it is one mission, not three.
Artificial splitting is as bad as dumping — do not manufacture commits to look
diligent.

### 4. Order so the history always builds

Sequence commits the way the work would have been built by someone who knew
where they were going: foundations first, then what depends on them.

Configuration and ignore rules before the files they govern. A leaf module
before its consumers. Schema before the code that queries it. Docs and CI can
come last, since nothing depends on them.

Keep each commit self-consistent. When a commit introduces a dependency, the
manifest and lockfile change belongs *in that commit* — not committed early in
final form and backfilled. If that means re-running the dependency tool at two
or three points in the sequence, do it. A history that claims a package was
required before the code that required it is a small lie, and lies in history
cost debugging time later.

For long sequences, sanity-check the ordering rather than exhaustively building
every intermediate state. If correctness per commit genuinely matters (a repo
that gets bisected often), verify with a rebase exec pass:

```bash
git rebase --exec "<build-or-test-command>" <base-commit>
```

### 5. Preview the plan before executing it

Show the first one or two commits — the exact subject line and the files each
one contains — plus a one-line sketch of the remaining sequence. Get a nod
before running the whole thing.

This is cheap and it catches grouping disagreements while they still cost one
message instead of an interactive rebase.

### 6. Verify, then commit

Run whatever the project uses to prove the code is sound — formatter, build,
tests, linter — before the first commit lands. Committing work you have not
verified passes a hidden liability to whoever pulls next.

Then check the branch. If the work is sitting on `main` or `master` and the
project uses branches, move it first — see "work already started on `main`"
above.

Finally, inspect what is staged before every single commit:

```bash
git diff --cached --name-only
```

### 7. Report what you did

List the resulting commits. State plainly anything that is not green — a
skipped test, an unverified path, a check you could not run. Never imply a
verification you did not perform.

## Writing the message

### Prefix

Pick the prefix from what the change *does*, not which files it touches. A test
that is part of a new feature is `feat:`; a test added to cover existing
behaviour is `test:`.

| Prefix      | Use for                                                    |
| ----------- | ---------------------------------------------------------- |
| `feat:`     | New capability a user or caller can observe                 |
| `fix:`      | Corrected behaviour that was wrong                          |
| `refactor:` | Restructuring with no behaviour change                      |
| `doc:`      | Documentation, comments, ADRs, READMEs                      |
| `test:`     | Tests for behaviour that already exists                     |
| `ci:`       | Pipelines, workflows, automation                            |
| `chore:`    | Tooling, config, scaffolding, dependency bumps              |
| `perf:`     | Changes made for speed or resource use                      |
| `revert:`   | Undoing a previous commit                                   |

Add a scope when the repo already uses them and it genuinely disambiguates:
`feat(auth): ...`. Skip it when the subject is already clear.

### Subject line

Keep it under 100 characters, and prefer 50–72 — that is what `git log --oneline`,
GitHub, and most terminals show without truncating. Under 100 is the hard limit;
50–72 is the target.

Write it declarative and natural, in the imperative mood — the tense Git itself
uses for its generated messages. The test: *"If applied, this commit will
\_\_\_"*. If the sentence reads correctly, the mood is right.

No trailing period. The subject is a title, not prose.

```
feat: add postgres and redis connection adapters with readiness checks
fix: reject expired sessions on token refresh
refactor: extract tenant scope resolution from the request handler
doc: record foundation architecture decisions
```

Not this:

```
Fixed the bug                            # what bug? not imperative
feat: stuff                              # says nothing
update files                             # no prefix, no content
feat: add auth and fix the redis timeout # two missions in one commit
feat: Add User Authentication.           # title case, trailing period
```

### Body

Most commits do not need one. Reach for a body only when the *why* is not
evident from the diff — a non-obvious trade-off, a constraint you worked around,
a bug whose cause is worth recording. The diff already shows what changed; the
body is for what the diff cannot say.

When you write one: blank line after the subject, wrap around 72 characters,
and keep it to a few short sentences. Long paragraphs go stale and stop being
read. If the explanation is genuinely long, it belongs in an ADR or the PR
description, with the commit pointing at it.

### Trailers

Machine-readable metadata at the end, after a blank line:

```
Refs: #482
Co-Authored-By: Name <email>
BREAKING CHANGE: sessions issued before v2 are no longer accepted
```

## Landing the branch

Merging into a shared branch and pushing are outward-facing and awkward to
undo. Confirm before doing either, even when the work itself was requested.

### Keep the branch current while you work

If the branch is yours alone and unpushed, rebase onto `main` — it keeps history
linear and avoids merge commits that say nothing:

```bash
git fetch origin
git rebase origin/main
```

If anyone else has the branch, merge instead. Rebasing rewrites commits that
other people already have, which forces them into a recovery they did not ask
for:

```bash
git merge origin/main
```

The same rule governs the whole skill: rewrite freely while history is private,
treat it as immutable once it is shared.

### Before merging

Run the project's checks on the branch and confirm they pass. Re-read the diff
against `main` as a reviewer would (`git diff main...HEAD`) — this catches debug
leftovers and stray files better than reading your own commits. Confirm no
secrets or build output are present.

If the project merges through pull requests, open one rather than merging
locally. Do not merge someone else's approval gate away.

### Choosing the merge strategy

| Strategy | Command | Use when |
| --- | --- | --- |
| Merge commit | `git merge --no-ff feat/x` | The branch's commits are atomic and meaningful. Preserves them and records that a feature landed as one unit |
| Rebase, then fast-forward | `git rebase main` then `git merge --ff-only` | You want strictly linear history and the branch is small and private |
| Squash | `git merge --squash feat/x` | The branch history is noise — "wip", "fix typo", "actually fix it" — and only the end state is worth keeping |

The choice interacts directly with the commit discipline above. If you curated a
sequence of atomic commits, squashing throws that work away and collapses it
into one lump; use `--no-ff` or rebase instead. Squash is the right answer for
exploratory branches, not for curated ones.

A squash merge produces one commit, so give it a message that follows the same
rules as any other: conventional prefix, declarative subject under 100
characters.

Whatever the project already does wins. Check `git log --graph --oneline -20` on
`main`: visible merge bubbles mean merge commits, a flat line means squash or
rebase.

### After the merge

Confirm `main` is green, then clean up — stale branches accumulate into a list
nobody trusts:

```bash
git branch -d feat/auth-google-oauth              # -d refuses if unmerged, which is the point
git push origin --delete feat/auth-google-oauth   # if it was pushed
```

Use `-d`, not `-D`. The refusal on an unmerged branch is a safety check worth
keeping.

## Keep these out of commits

- **Secrets.** `.env` files, keys, tokens, credentials. Confirm the ignore rules
  cover them, and scan staged filenames before committing. A secret in history
  survives deletion and means rotating the credential.
- **Build output and dependencies.** `node_modules/`, `dist/`, `.next/`,
  compiled binaries, caches.
- **Unrelated drive-by fixes.** Tempting, but they make the commit un-revertable
  as a unit. Give them their own commit.
- **Debug leftovers.** Stray print statements, commented-out blocks, temporary
  files from your own verification.

## Community practices worth knowing

These are the conventions the above is built on, in case you need to explain or
adapt them:

- **Conventional Commits** (conventionalcommits.org) — the `type(scope): subject`
  format. Its real payoff is machine-readable history: automated changelogs and
  semantic version bumps, where `feat:` implies a minor bump and
  `BREAKING CHANGE:` a major one.
- **Atomic commits** — one logical change per commit. The property that makes
  `git revert`, `git bisect`, and `git cherry-pick` useful rather than
  theoretical.
- **The imperative mood** — Git's own convention, from the kernel project.
  Generated messages ("Merge branch", "Revert") are imperative, so yours should
  match.
- **The 50/72 rule** — Tim Pope's guideline: ~50-character subject, blank line,
  body wrapped at 72. Predates and outlives most tooling.
- **Explain why, not what** — the diff is the authoritative record of what
  changed. Reviewers can read code; they cannot read intent.
- **Never rewrite published history** — amend, rebase, and squash freely on local
  work; treat anything pushed to a shared branch as immutable unless the user
  explicitly asks otherwise. Never force-push without being asked.
- **Short-lived branches** — the empirical finding behind trunk-based
  development and the DORA research: integration pain scales with how long
  branches stay open, so the fix for a large feature is more branches, not a
  longer one.
- **Feature flags over long branches** — the standard way to keep unfinished
  work on `main` without exposing it. Trades a branch you must eventually merge
  for a flag you must eventually remove; the flag is the cheaper debt.
- **Commit when asked** — do not commit, merge, or push on your own initiative.
  This skill runs because someone confirmed the work is done.

For the named branching models — trunk-based development, GitHub Flow, and Git
Flow — and how to pick between them, read `references/branching-models.md`. Reach
for it when a project has no established convention, or when someone asks which
model to adopt.