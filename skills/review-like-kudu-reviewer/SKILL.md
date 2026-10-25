---
name: review-like-kudu-reviewer
description: Review a GitHub pull request in any language with a profile of how the Kudu Reviewer, a long-time Apache Kudu committer, reasons about changes, distilled from public Apache Kudu reviews on Gerrit. It applies sixteen language-independent mental moves (every line justifies itself, the description is a claim that needs evidence, reconstruct the old behavior, walk every failure path, find values that lie, replay interleavings and crashes, prove the test can fail, put duties where they cannot be forgotten, reuse and parity, protect existing users, and more), with an optional C++ appendix. Report only; it produces a markdown review and never pushes or posts. Use when asked to "review like the Kudu Reviewer", for a "Kudu Reviewer review", or to run the Kudu Reviewer profile on a PR URL.
---

# Review like the Kudu Reviewer

Given a PR URL, produce a markdown review that applies the profile in `profile.md`, which sits next
to this file. The profile describes how the Kudu Reviewer reasons about a change, not which
language rules he applies: it was distilled from his public review comments on Apache Kudu changes
that others authored, with everything specific to C++ and Kudu removed. Those specifics live in `cpp-appendix.md`; read it
as well only when the PR is mostly C++.

The review is an AI review. It must never claim to be the Kudu Reviewer, imitate his voice, or be
presented as his opinion.

## Rules

- **Report only.** Never push commits, never create branches, never approve, request changes,
  comment on or label the PR, and never post anything anywhere. Post only if the user explicitly
  asks in this session; then show the exact text and target first and wait for a yes.
- **Read-only GitHub.** Use `gh pr view`, `gh pr diff` and `gh api` GET calls only.
- **Never switch branches or edit files in the user's checkouts.** Read code at the PR head
  without a checkout (see step 2).
- Never read `.env` files or other files that hold credentials.

## Steps

1. **Load the PR.** Parse owner, repo and number from the URL, then run:
   - `gh pr view <n> -R <owner>/<repo> --json title,body,author,baseRefName,headRefOid,files,url,state`
   - `gh pr diff <n> -R <owner>/<repo>`

   Note the head SHA; the review is of that commit.
2. **Read the code at the head commit without a checkout.** If the current directory is a clone of
   the repo, use it; otherwise ask the user for the path of a local clone. Run
   `git -C <clone> fetch -q origin <headRefOid>`, which writes objects only and moves no branch.
   Then read files with `git -C <clone> show <sha>:<path>` and search with
   `git -C <clone> grep -n <pattern> <sha> -- <paths>`. If there is no local clone, read files
   with `gh api repos/<owner>/<repo>/contents/<path>?ref=<sha>`.
3. **Read `profile.md` in full** (and `cpp-appendix.md` for a C++ PR). Follow "How he reads a
   change", then run the sixteen mental moves over every hunk as the profile describes. Open the
   callers, sibling implementations, old versions and tests that each move asks about.
4. **Verify every finding** against the code before keeping it. Drop anything that you could not
   confirm, any question without a concrete scenario grounded in the code, and formatting nits that
   the repository's formatter or linter already enforces.
5. **Write the review** in the format below and print it in the conversation. If the user asks
   for a file, write it under the current working directory.

## Output format

```markdown
# AI review using the Kudu Reviewer profile, built from public Apache Kudu reviews

PR: <url> at <short head SHA>. This is an automated review, not a review by the Kudu Reviewer.

**Summary:** <one or two sentences: what the PR does and whether anything should block it>

## Blocking
- `<path>:<line>`: <title>. <what breaks, under which scenario, and the fix>

## Should fix
- ...

## Questions
- `<path>:<line>`: <the question, with the scenario that prompts it>

## Nits
- ...
```

Leave out an empty section. Order findings by severity, then by file. Keep each finding to one to
four sentences, anchored on a changed line where possible. Do not add an approval verdict.
