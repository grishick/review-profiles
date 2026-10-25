---
name: review-like-grishick
description: Review a GitHub pull request in any language or project using a profile built from grishick's (Greg Solovyev's) past code reviews. It asks who and what the change touches beyond the happy path (existing users on upgrade and downgrade, every variant of the product, callers and consumers, failing dependencies), whether tests prove the claim, whether public contracts, docs and versions are right, whether secrets or insecure modes leak, and whether the PR carries dead code, duplication or unrelated changes. Report only; it produces a markdown review and never pushes or posts. Use when asked to "review like grishick", for a "grishick-style review", or to run the grishick profile on a PR URL.
---

# Review like grishick

Given a PR URL, produce a markdown review that applies the profile in `profile.md`, which sits next
to this file. The profile is project-agnostic. It was built from grishick's own review comments on
other people's PRs in four projects: a Java mail server, a data integration platform, a mobile and
desktop messenger, and a TypeScript backend. AI reviewer output that was posted from his account was
excluded.

The review is an AI review. It must never claim to be grishick, imitate his voice, or be presented
as his opinion.

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
3. **Read `profile.md` in full.** Do its "Adapting to the project" step first: learn how this
   project versions, releases and deploys, and which variants and sibling components it has. Then
   follow "How to work through a PR" (claim, existing users, variants, failure, tests, contracts,
   security, simplicity and scope, clarity), and scan the diff for every row of the triggers table.
4. **Verify every finding** against the code before keeping it. Drop anything that you could not
   confirm, and drop findings that the profile lists under "What this reviewer lets pass".
5. **Write the review** in the format below and print it in the conversation. If the user asks
   for a file, write it under the current working directory.

## Output format

```markdown
# AI review using a profile built from grishick's past reviews

PR: <url> at <short head SHA>. This is an automated review, not a review by grishick.

**Summary:** <one or two sentences: what the PR does and whether anything should block it>

## Blocking
- `<path>:<line>`: <title>. <what breaks, for whom, and the fix>

## Should fix
- ...

## Nits and questions
- ...
```

Leave out an empty section. Order findings by severity, then by file. Keep each finding to one to
four sentences, anchored on a changed line, or PR-level for scope, description, tests, upgrade
paths and variant gaps. Do not add an approval verdict.
