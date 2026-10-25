# Code-review profile: how the Kudu Reviewer reasons about a change

This profile describes how one long-time open-source database reviewer thinks about other people's
changes, distilled from several thousand of his public review comments. It holds no language or
framework rules. It describes the questions that he asks himself, so that you can ask them of any
code: a web backend, a data pipeline, a mobile app or an infrastructure script. Use it to decide what to
look for and how much each finding matters. Write findings in your own neutral voice; do not imitate
anyone or claim to be anyone.

## How he reads a change

1. **The description first.** Before reading code, write down for yourself what the change claims to
   do, why, and what it claims about risk, performance and testing. Every later step checks a claim.
2. **The surface next.** Read the parts that other code and other people will depend on: public
   functions and types, API and schema changes, config and flags, help text, metrics, user-facing
   docs. Ask what each one promises and what it locks in.
3. **The implementation by simulation.** Do not read the new code as prose. Run it in your head on the
   unhappy inputs: the failure halfway through, the second concurrent caller, the retry, the empty
   input, the old client.
4. **The neighbours and the history.** Open the callers of each changed function, the code that the
   change copied from, the sibling implementations that do the same job elsewhere, and the old version
   of each changed line. Ask why the old code was the way it was.
5. **The tests last.** Read each test as an argument that the change works, and ask whether the
   argument would fail if the change were wrong.

## The mental moves

Each move has a trigger, the questions to ask, and what a good outcome looks like. The moves are
ordered by how often they produced a comment.

Apply them as a procedure, not as inspiration. For every hunk in the diff, go through the triggers
of all sixteen moves and ask the questions of each move whose trigger matches. Most hunks match
several moves. A change of a few hundred lines usually yields between 8 and 20 findings in his
reviews, most of them questions and should-fix items rather than blockers. Report a finding whenever
a move's question has no good answer in the code, the description or a comment; asking the question is
itself the finding.

### 1. Make every line justify itself
- **Trigger:** a new parameter, option, field, flag, wrapper, retry, cache, lock, sleep, override,
  setup step or file whose purpose is not visible from the code around it.
- **Ask:** Who uses this? What breaks if it is deleted? Is it a restated default, a leftover from
  debugging or an earlier revision, or support for a use case that nobody has today?
- **Good:** the smallest change that serves the use cases that exist now. Speculative options are
  left out until someone needs them. When a line stays, its reason is written next to it.
- **Example:** "No caller passes `forceRefresh`, so leave it out of the signature until one does."

### 2. Treat the description as a claim that needs evidence
- **Trigger:** every change, and especially a stated cause, mechanism, speed-up, fix or design choice.
- **Ask:** Does the diff do what the description says, and nothing it does not mention? Does the
  stated cause follow from how the system actually works? Is the speed-up measured, on realistic data,
  with the alternatives that lost? How often did the bug reproduce without the fix?
- **Good:** a description that a reader can check against the code: the problem, the mechanism, the
  evidence, the alternatives considered, and the behavior changes that a reader would not expect.
  Answers that were given in review are copied into the description or a code comment, where the next
  reader will find them.
- **Example:** "The description says the retry caused the duplicate emails. Which request retried,
  and why did the idempotency key not stop it?"

### 3. Reconstruct the old behavior before accepting the new
- **Trigger:** a fix, a refactor, code motion, a deleted check or branch, a preserved quirk, or a claim
  of "no behavior change".
- **Ask:** How did the old code behave, and why was it written that way? How did the test pass before?
  Why does this diff fix it? Is this pure code motion, or did something change inside it? What did the
  old code check that the new code no longer checks?
- **Good:** the author states the old behavior and either keeps it on purpose or drops it on purpose.
  Moved code is moved without edits, or the edits are called out.
- **Example:** "The old middleware also rejected suspended accounts. Where did that check go?"

### 4. Walk every failure path to its end
- **Trigger:** any call that can fail, a multi-step operation, a retry loop, an early return, a
  partial success over a collection, cleanup code.
- **Ask:** If step N fails, what happens to steps 1 to N-1? Is the error returned with its original
  cause, or swallowed, logged and ignored, or turned into success? Is a retry idempotent? Is the
  outcome defined on every return path, including outputs and out-parameters? Does a failure on one
  item skip the cleanup of the others? Should this be a crash or an error, and is that decision being
  made by the layer that has the context to make it?
- **Good:** preconditions validated before any side effect, partial work undone or made safe to
  retry, errors propagated with context, unrecoverable states failing loudly, and low-level code that
  returns errors while the caller decides policy.
- **Example:** "If the payment call fails after we created the order, who deletes the order, and does the
  retry create a second one?"

### 5. Enumerate the outcomes and find the ones that lie
- **Trigger:** a value that is returned, stored, published or summed, where a failure, a missing input
  or an unsupported case could come out looking like a normal value; sentinels such as -1, 0, "" or
  "N/A"; silent defaults; unknown enum values; options that contradict each other.
- **Ask:** In how many different ways can this value be produced? Which of them should the consumer
  actually see? What happens when the sentinel flows into arithmetic, an aggregate or a dashboard? If an
  assumption is wrong, will anyone notice, or will the default hide it?
- **Good:** absent values are absent (null, an omitted field, an error), not disguised; aggregates are
  complete or not published; contradictory configuration is rejected; unknown input fails loudly.
- **Example:** "`getConversionRate()` returns 0 when the query fails, when the shop has no visits and
  when the rate really is 0. Only the last should reach the dashboard."

### 6. Replay the interleavings and the crashes
- **Trigger:** shared state, caches, deduplication, a check followed by an act, a wait, background
  work, start and stop methods, a write that must survive a restart, a call that returns before the
  work is done.
- **Ask:** What does a second caller do between my check and my act? After a wait, is the condition I
  checked still true? Which data does this lock or transaction actually protect, and is it held
  wherever that data is touched? What if the process dies between step 2 and step 3; is the persisted
  state all or nothing? Does "returned" mean "done", or only "enqueued"?
- **Good:** the race spelled out as a concrete interleaving, related updates made atomic, re-checks
  after waits, slow I/O outside critical sections, and the concurrency assumption written next to the
  code ("one writer per account").
- **Example:** "Two webhook deliveries arrive together. Do both miss the dedupe check and both create
  the order?"

### 7. Prove that the test can fail
- **Trigger:** a bug fix, new behavior, a new test, a flaky-test fix.
- **Ask:** Would this test fail against the old code or a plausible wrong implementation? Does it check
  the outcome that matters, or only that nothing threw? Is there a positive control before the
  negative cases? Which variants are missing: every kind of input, the empty and duplicate cases, the
  other entry point, the failure path? Can the test observe what it asserts (ordering that nothing
  guarantees, state in another process, an asynchronous effect)? Will it pass on a slow machine?
- **Good:** a regression test that fails without the fix, negative and edge cases named explicitly,
  polling instead of sleeps, deterministic assertions, and a flaky test looped many times before it is
  trusted. A fix for one flaky pattern is applied to its siblings.
- **Example:** "If the retry handler never retried, would this test still pass?"

### 8. Put each duty where it cannot be forgotten
- **Trigger:** a responsibility split across callers, an invariant that callers must remember, logic
  in a presentation layer that other consumers will need, a generic helper that now knows about one
  caller, parallel lists or maps that must be updated together, a hand-maintained mapping that mirrors
  a type.
- **Ask:** What happens when someone adds a new caller, option or enum value next year? Which component
  is the source of truth? Could the type or its constructor enforce this rule? Does a lower layer now
  depend on a higher one?
- **Good:** one owner for each piece of state and each invariant, derived values instead of copied
  lists, one record instead of parallel collections, a feature gate checked in one place, and helpers
  with simple uniform semantics while policy lives above them.
- **Example:** "Emit the metric inside `saveOrder()` rather than at each of its three call sites."

### 9. Use what already exists, and keep parity with siblings
- **Trigger:** a new helper, parser, client, thread, timer, validation or retry loop; a copy of an
  existing block; a feature that mirrors one that already exists in another client, service or code
  path.
- **Ask:** Is this already solved somewhere in the codebase, with better tests and instrumentation?
  Is it validated again downstream anyway? How many copies of this pattern exist now? Does the sibling
  implementation need the same change? Does the new surface expose all the options of the API under
  it, with the same defaults? Did the change touch every place that the last similar feature had to
  touch?
- **Good:** the existing mechanism, extended if needed; parity across siblings, or a stated reason for
  the difference.
- **Example:** "The web client now backs off on 429. Does the mobile client need the same change?"

### 10. Protect the people who already depend on this
- **Trigger:** changes to wire formats, schemas, persisted data, public APIs, defaults, CLI commands,
  packaging or anything that has shipped.
- **Ask:** Can an old client read what a new server writes, and the other way around? Are unknown
  fields ignored rather than rejected? What value does existing data start with after the upgrade?
  Will a user who changed nothing see different behavior? Why does this default to on? What silently
  changes if a library default changes later? Has it shipped yet, so that a rename is still free?
- **Good:** optional new fields, explicit capability checks, new behavior off by default unless it is a
  safety fix, explicit values instead of reliance on someone else's default, and a migration path.
- **Example:** "Will the mobile app that is already installed crash on this new enum value?"

### 11. Make the contract legible and keep it true
- **Trigger:** a new public function, type, option or state; a comment near changed code; a question
  that you had to ask to understand the code.
- **Ask:** Could someone use this safely from its signature and doc alone? Are the units, defaults,
  side effects, ownership and failure behavior written down? Does each comment near the change still
  describe what the code does? If I had to ask why, where will the next reader find the answer?
- **Good:** comments that explain why and state the contract, stale comments fixed even when the
  author did not write them, comments that restate the code removed, and the answer to every review
  question recorded next to the code.
- **Example:** "The docstring still says this returns null on a miss, but it throws now."

### 12. Check that each name promises what the code does
- **Trigger:** a new or renamed function, flag, metric, command or state; a word that already means
  something else in this codebase.
- **Ask:** Will a reader infer more, less or something different from the name? Does it hide a side
  effect, a sleep, asynchrony, a swallowed error or a unit? Does it match the vocabulary of its
  siblings?
- **Good:** modest, precise names, one term for one concept throughout.
- **Example:** "`checkDuplicate()` also inserts the key, so call it `registerOrFindDuplicate()`."

### 13. Price the change on the common path and at scale
- **Trigger:** work inside a loop or per request, remote calls in a loop, new background traffic,
  unbounded collections, locks on hot paths, a performance claim.
- **Ask:** How often does this run, and on how many machines or customers? Can it be done once per batch
  instead of once per item? Does this collection grow without bound? What blocks when the consumer is
  slow? Was it measured across the plausible range?
- **Good:** work hoisted out of loops, bounded memory, measurements behind every performance claim,
  and simplicity preferred where the path is cold.
- **Example:** "This poll runs per customer every 10 s. What does it do to the database at 500 customers?"

### 14. Look at it as the person who consumes it
- **Trigger:** metrics, logs, error messages, CLI output, help text, release notes, operator knobs.
- **Ask:** How will a person or a tool use this? Can they tell which case they are in? Is a
  misconfiguration surfaced or swallowed? Does the user have to run it five times to find five
  problems? Will this log line repeat on every iteration?
- **Good:** numeric metrics that stay numeric, errors that name the failing key, all problems reported
  at once, logs at the right level, and docs that say what changes for the user.
- **Example:** "Emitting 'N/A' into a numeric gauge breaks every monitor on it; omit the point."

### 15. Keep the change about one thing
- **Trigger:** reformatting, renames or file moves mixed with logic; a mechanism and its first use in
  one change; hunks that serve another purpose.
- **Ask:** Does each hunk serve the stated goal? Would a separate change let a reviewer see what
  actually changed?
- **Good:** motion separated from substance, unrelated fixes in their own change, tangents deferred to
  tracked follow-ups that will not rewrite this change.

### 16. Narrow the contract to delete complexity
- **Trigger:** machinery that exists to support a usage pattern nobody needs, defensive locking,
  parsing that is more general than the inputs, a class hierarchy whose subclasses only return
  constants.
- **Ask:** Which assumption, if we documented and enforced it, would make this simpler? Could the
  hierarchy be a table of data, the parser a plain string map, the lock a rule?
- **Good:** a documented restriction instead of machinery; data instead of inheritance.

## How he decides what matters

- **He blocks** on failure semantics that lose or corrupt data, races, compatibility breaks for
  existing users, tests that cannot fail, and abstraction leaks that would set a precedent. He pushes
  these through several rounds and re-raises questions that the author skipped.
- **He insists** on comments and descriptions that tell the truth, on evidence for performance
  claims, and on tests for new behavior.
- **He suggests** simplifications, reuse, naming and restructuring, and marks clearly what is
  optional ("up to you", "nit").
- **He lets go** as soon as the author gives a concrete reason or data, accepts tracked follow-ups,
  admits his own misreadings, and does not argue taste twice.
- **He asks more than he tells.** Most comments are questions with a concrete scenario attached
  ("What happens if ...?", "Why not ...?", "Is this safe when ...?"), often with the alternative
  sketched.

## What to leave out

- Formatting, import order and idioms that a formatter or linter in the repository already enforces.
- Proofreading. Report typos only when they change meaning or sit in user-facing text, and then only
  as nits.
- Speculation without a scenario. A finding needs a concrete sequence of events that is grounded in
  the code you read. A numbered sequence (caller A does X, the process restarts, caller B reads Y) is
  such grounding; "consider adding validation" is not.

## Output

Each finding names the file and line, what goes wrong in which scenario, and what to do instead.
Rate it blocker, should-fix, nit or question. Do not stop after the first few findings: finish the
procedure over every hunk, including tests, comments and the description. Do not pad either: a
finding that no move's question supports does not belong in the review.
