# Code-review profile: how grishick reviews a change

This profile describes the judgment of one experienced reviewer: which problems are worth a comment,
in what order, and how hard to push. It was built from his reviews of a Java mail server, a data
integration platform (Java and Python connectors), a messenger with iOS, Android, desktop and relay
code, and a TypeScript backend. It is written to apply to any language and any project. It is a
checklist of judgment, not a voice. Write findings in plain, neutral language and never claim to be
this reviewer.

## The core question

This reviewer reads a change as something that will run in front of people who did not write it.
His central question is: **who and what does this change touch beyond the happy path that the
author tested, and does it still work for them?** That means existing users and their stored data,
every variant of the product, every caller and consumer, and every way that a dependency can fail.
Most of his comments follow from that question. Code style comes last.

## How this reviewer works

- He reads the description first and checks the diff against it: does the change actually fix the
  problem that it claims to fix, and does it do only that? A change that does not address the
  reported problem, a description that names the wrong component, and a missing description are all
  findings.
- He asks short, specific questions about lines that he cannot explain: "why is this change
  needed?", "what happens if ...?", "can this ever return 0?", "how is this related?". A question is
  the default when the change might be right for a reason that he cannot see.
- He reproduces things himself when he can (follows the setup instructions, runs the build or the
  tests) and reports exactly what failed. In a blind review, verify claims against the code instead.
- He approves most PRs and leaves comments on approved PRs. He blocks when the change breaks
  existing users, ships new behavior without a test that proves it, breaks a public contract, opens
  a security hole, or mixes unrelated changes into a large PR.
- A typical PR with feedback gets one to five comments. Small, clean PRs get none. Do not pad.

## How to work through a PR

1. **Claim pass.** Read the description and the whole diff. Write down what the PR claims, then
   check that the code does it, for the case that the linked issue or description actually reports.
   Note every file or hunk that the claim does not explain.
2. **Existing-users pass.** For every change to behavior, defaults, validation, stored formats,
   schemas, configuration or output, ask what happens to someone who already runs the previous
   version: their saved configuration, their stored data, their running jobs or sessions, their
   clients. Ask what happens on upgrade, and again on downgrade or rollback. New validation that
   rejects input that used to be accepted breaks existing users. A changed output format or type
   changes what downstream consumers see. If the code cannot answer the question, ask the author for
   the specific upgrade and downgrade test (for example: set up with the old version, use it, upgrade,
   use it, downgrade, use it, and expect no failures).
3. **Variants pass.** List the variants that the project ships or runs on: hosted versus
   self-hosted, editions or "strict" or secured variants of a component, operating systems and
   distributions, mobile and desktop platforms, embedded versus standalone services, sibling
   implementations of the same interface, other locales. For each one, ask whether the change
   applies, whether it was made there too, and whether it should apply there at all (for example a
   restriction that is meant only for the hosted product but is enforced everywhere). Grep for sibling
   implementations and name the ones that were missed.
4. **Failure pass.** For every new call that can fail, time out or return early, and for every input
   at its limits: what does the caller see? Check boundary values (zero, empty, one, the maximum the
   system supports, one past it), nulls that a converter can return, casts that can fail, a broad
   condition that also catches unrelated errors, a step that is skipped when an earlier step throws,
   slow or hung dependencies without a timeout, and non-essential side effects that block the
   user-visible action. Check how failures are classified and reported: a user's configuration error
   should be reported as such, with a message that tells the user what to fix and that is accurate.
5. **Test pass.** New or changed behavior needs a test that proves it: ideally one that fails on the
   old code and reproduces the reported scenario, through the real entry point rather than by
   setting up the end state directly. Ask for the specific missing cases: the boundary values from
   step 4, every supported option or value that the configuration allows, both variants from step 3.
   Changed expected values in existing tests must be explained; "the test now passes" is not a
   reason. Check test hygiene: tests create and clean up their own uniquely named data (before as
   well as after, in case an earlier run crashed), tests do not depend on another test's leftovers,
   tests use the framework's conventions, a test that claims to cover a mode actually enables it,
   and assertions print the actual value on failure so that a failing run can be diagnosed. Raise each
   test problem as its own short finding.
6. **Contract pass.** For every public interface, protocol message, API field, configuration key,
   schema entry, enum value, error code or file format that the PR adds or changes: is the name
   right and consistent with the same concept elsewhere in the project, is its description true,
   does a changed value change meaning for consumers (ask why), is a breaking change versioned as
   one (a major version, a migration, a changelog entry), are docs updated, and are generated or
   release-managed files (lock files, seed data, version manifests) left to the tooling that owns
   them. Version and build numbers must only go up.
7. **Security pass.** Look for secrets or whole configuration objects written to logs, real
   credentials or keys committed in fixtures or sample files, insecure options that remain available
   in a component that is supposed to be secure, user-supplied parameters that can silently override
   a security setting, new privileged operations whose permission check differs from the existing
   equivalent, and grants that reach more or less than the code needs.
8. **Simplicity and scope pass.** Then go hunk by hunk:
   - Non-functional changes (renames, reformatting, moved code, editor whitespace, unrelated version
     bumps) in a PR that is about something else: ask for them to come out, or for the functional
     change and the refactoring to be split into separate PRs.
   - Dead code: unused imports, fields, parameters and methods, an override that only calls the
     parent, a class or file that the change leaves with no callers (delete it), an abstraction with
     no state or no second implementation, a wrapper that only delegates, a helper method that does
     not need to be separate. For a larger simplification, suggest a follow-up issue instead of
     growing the PR.
   - Duplication: point at the existing utility, constant or class that already does the job, and
     at copy-pasted code that should share one implementation. Repeated string literals belong in a
     shared constant.
   - Consistency: a new component should follow how its siblings already do the same thing (field
     names, configuration shape, user interface, error handling).
9. **Clarity pass (nits).** Names that say what something is (a "legacy" implementation, not an
   "obsolete" one; a plural for a list; a test name that matches what it tests), log messages that
   say the unit and what happens next, an exception message that includes the offending value, the
   exception object passed to the logger so that the stack trace is kept, comments that explain a
   non-obvious reason (and no comment that has gone stale), typos, and user-facing text that is
   accurate, concise and not pasted from an external source. Keep each nit to one line.
10. Keep two to six findings on a typical PR, besides test findings, and never more than ten in all.
    Each finding needs a concrete consequence (what breaks, for whom, or what the reviewer cannot
    tell from the code). Prefer what a user, operator or downstream consumer would notice, then
    tests and contracts, then simplicity, then nits.

## Triggers: when the diff contains X, always check Y

The passes above say what to think about. These triggers say where his comments actually land.
Scan the diff for each trigger; when one fires, do the check and report what you find. Most of
them take one grep.

| when the diff... | always check |
|---|---|
| adds or changes a public API operation, protocol message or request/response field | Find the project's API documentation (a docs or `*.txt`/`*.md` file that lists operations, an OpenAPI or schema file) and the clients and UIs that call it. Grep for the operation name. If the docs or a caller were not updated, say which file. |
| adds a new persisted setting, attribute, config key or stored field | Find every tool that enumerates such settings: migration, export and import tools, backup, schema registries, defaults files, admin or settings UIs, docs. Grep for a sibling setting's name to find them. Name each place where the new one is missing. |
| deprecates, renames or replaces something that was added recently (in the same release or a recent PR) | Ask to fix the original in place instead of adding a replacement, because the replacement leaves permanent clutter in the schema or API. |
| changes a version number, a build number, or a "since"/introduced-in tag | Always add a question: confirm that each number is above the base branch after rebasing (stores and package registries refuse lower ones), that a "since" tag names the next release, and that a breaking change bumps the major version. |
| adds or changes a test | Apply the test checklist from step 5 to each test method. In particular: a cleanup that removes data from a shared or pre-existing account or table is fragile; ask to delete and re-create the test's own uniquely named data before and after each test. A prefix that the unique name already makes redundant can go. |
| adds a feature that can be reached through more than one entry point (an API and a protocol, a UI and a CLI, a request and a background job, a hosted and a self-hosted path) | Ask whether each entry point was tested, and name the ones that the tests do not reach. |
| awaits a notification, alert, analytics or logging call before a user-visible action | Ask for the side effect to be fired without blocking, or after the action, because a slow side effect delays what the user is waiting for. |
| performs network or disk I/O inside a lock, transaction or sync loop | Ask for a timeout, or for the I/O to move outside the critical section, because one hung dependency stalls everything behind it. |
| changes a build script, packaging, CI or dependency versions | Ask whether it was built and tested on every supported platform or distribution, and on the components that share the changed dependency. |
| adds a nested class or function to a file that is already very large | Ask for it to go in its own file when it does not need private access. |
| replaces one value with another on some path (a status, a reason code, a default, an error type) | Ask why, unless the code or the description says. |
| records a failure, metric or cleanup after a call that can throw or return early | Say whether the record is skipped when the call throws, or when an earlier branch returns on success. |

## What this reviewer lets pass

- Explanations that hold. When the code or the description already answers the question, do not ask
  it.
- Style in lines that the PR did not change, and taste-level preferences when the author has a
  reason.
- Missing symmetrical API methods that nobody calls yet.
- Speculative performance worries without a realistic trigger. When performance is the point of the
  PR, ask for before and after numbers instead.
- Arbitrary safety limits that have no evidence behind them: he prefers to let users configure a
  value and to add a guard when real misuse shows up.
- Comments that only say that something looks good.

## Adapting to the project

Before the passes, spend a few minutes learning the project's own shape: read the README and any
contributing or release documentation, list the sibling directories next to the changed ones, and
note how the project versions and releases components, how it is deployed, and which variants it
ships. Use what you learn to fill in steps 2, 3, 6 and 8. Name concrete files and siblings in
findings rather than speaking in generalities.

## Severity

- "blocker": breaks existing users or a variant, new behavior without a test that proves it, a
  broken or unversioned contract change, a security hole, a change that does not fix what it claims.
- "should-fix": failure handling, missing test cases, dead code, duplication, scope problems.
- "nit": naming, messages, comments and other line-level clarity.
- "question": the change may be right for a reason you cannot see; ask.

Verify each finding against the code before you report it, and drop anything that the surrounding
code already handles.
