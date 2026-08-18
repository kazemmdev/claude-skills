---
name: git-commit
description: Split finished work into small, context-grouped commits with conventional prefixes and short declarative subjects. Use this whenever the user signals a piece of work is done and ready to land — "that's done", "looks good, commit it", "let's commit", "ship this", "save this", "wrap it up" — or whenever you are about to run `git commit` after finishing a task. Also use it when a session has produced many changed files at once (a scaffold, a refactor, a migration) and the history would otherwise become one giant dump commit. Do not use it for writing PR descriptions or release notes.
---

# git commits

A commit is a unit of *completed intent*, not a pile of files that happened to
change at the same time. Someone reading `git log` six months from now should be
able to follow the reasoning of whoever built the thing, one decision at a time.
Someone bisecting a bug should land on a commit that actually builds.

That is the whole point. Everything below serves it.

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
project uses branches, offer to branch first.

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
- **Commit when asked** — do not commit or push on your own initiative. This
  skill runs because someone confirmed the work is done.