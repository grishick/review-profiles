# Review profiles

Two code-review skills for Claude Code, each built from one experienced reviewer's past reviews.
They describe how each reviewer reasons about a change: what he looks for, in what order, and how
hard he pushes. They work on a pull request in any language.

| skill | built from | what it focuses on |
|---|---|---|
| `review-like-kudu-reviewer` | public review comments by the Kudu Reviewer, a long-time Apache Kudu committer, on Kudu changes that other people wrote | Sixteen mental moves: every line justifies itself, the description is a claim that needs evidence, reconstruct the old behavior, walk every failure path, find values that lie, replay races and crashes, prove that the test can fail, protect existing users. An optional appendix holds his C++ and Kudu checks. |
| `review-like-grishick` | grishick's (Greg Solovyev's) review comments on other people's PRs in a Java mail server, a data integration platform, a messenger and a TypeScript backend | Who and what the change touches beyond the happy path: existing users on upgrade and downgrade, every variant of the product, failing dependencies, tests that prove the claim, public contracts and versions, secrets, and scope. |

Both skills only report. They print a markdown review in the conversation and never push, comment,
approve or label. Every review says that it is an AI review and not a review by the person the
profile is named after; neither person reviewed the output.

## Install

The skills need the GitHub CLI (`gh`), signed in to an account that can read the repository.

### Claude Code

```text
/plugin marketplace add grishick/review-profiles
/plugin install review-profiles@review-profiles
```

To try a local copy without installing it:

```bash
claude --plugin-dir ./review-profiles
```

### Codex, Cursor, Grok Build and Antigravity

These tools read skills from `~/.agents/skills`. Clone the repository once and link both skill
folders there, so that a `git pull` updates them in every tool:

```bash
git clone https://github.com/grishick/review-profiles ~/src/review-profiles
mkdir -p ~/.agents/skills
ln -s ~/src/review-profiles/skills/review-like-kudu-reviewer ~/.agents/skills/
ln -s ~/src/review-profiles/skills/review-like-grishick ~/.agents/skills/
```

Restart the tool afterwards. If a tool does not list the skills, copy the folders with `cp -R`
instead of linking them.

#### Codex

Type `$` or run `/skills` to pick a skill. Codex's sandbox blocks network access by default, so
approve the first `gh` call or start the session with network access allowed.

```text
$review-like-kudu-reviewer https://github.com/owner/repo/pull/123
```

#### Cursor

Type `/` in Agent chat and search for the skill name. Skills in `~/.agents/skills` load only on the
local machine, so Cursor's Cloud Agents do not see them; for those, add the skill folders to the
project's `.agents/skills/` instead.

```text
/review-like-grishick https://github.com/owner/repo/pull/123
```

#### Grok Build

Grok Build lists each skill as a slash command.

```text
/review-like-kudu-reviewer https://github.com/owner/repo/pull/123
```

#### Antigravity

The Antigravity app reads `~/.agents/skills`, but the Antigravity CLI does not. For the CLI, link
the folders into `~/.gemini/config/skills/` as well:

```bash
mkdir -p ~/.gemini/config/skills
ln -s ~/src/review-profiles/skills/review-like-kudu-reviewer ~/.gemini/config/skills/
ln -s ~/src/review-profiles/skills/review-like-grishick ~/.gemini/config/skills/
```

If the CLI still does not list them in `/skills`, use `~/.gemini/antigravity-cli/skills/` instead.
Antigravity picks a skill from its description, so name the skill in your request:

```text
Review https://github.com/owner/repo/pull/123 with the review-like-kudu-reviewer skill
```

## Use

In Claude Code, ask for a review by name and give the PR URL. The install sections above show how
to invoke a skill in the other tools.

```text
Review https://github.com/owner/repo/pull/123 like the Kudu Reviewer
/review-profiles:review-like-grishick https://github.com/owner/repo/pull/123
```

Run the session from a clone of the PR's repository, so that the reviewer can read callers, siblings
and history at the PR's head commit without checking it out.

## How they were built

For each reviewer, his comments on PRs that other people wrote were collected, normalized and
grouped by the kind of problem that they point at. AI-generated comments posted from the reviewer's
account were dropped. The patterns were written up as a profile, and the profile was then
generalized: rules that were specific to one language, codebase or company were removed or moved to
an appendix, so that what remains is a way of reasoning that applies to any project.

## How they did on code that their authors never reviewed

Both profiles were tested blind on 31 merged Airbyte and LiveKit PRs (Go, Python, TypeScript,
Kotlin, Swift, Rust and Java) that neither reviewer had commented on. Each AI review was scored
against the human review comments that the PR's authors acted on, which GitHub marks as outdated
when their lines change later.

| reviewer | acted-on human comments found | findings per PR | findings judged useful |
|---|---|---|---|
| generic prompt that asks for 10 to 30 comments | 33% | 12.9 | 48% |
| `review-like-kudu-reviewer` | 31% | 8.8 | 54% |
| `review-like-grishick` | 18% | 5.3 | 67% |
| short generic prompt | about 23% | | |

- `review-like-kudu-reviewer` ties the verbose generic prompt with a third fewer findings, more of
  which are useful, and it makes the most catches that no other reviewer makes.
- `review-like-grishick` is the most precise reviewer but leaves the fewest findings. It rarely
  writes nits about names, imports and doc wording, which make up many of the human comments in
  these projects. Use it when you want a short list of findings that matter, not when you want
  coverage.

Neither profile is a better reviewer than a careful generic prompt on average. Each one is a
different reviewer, and they catch different things.

## License

MIT. See [LICENSE](LICENSE).
