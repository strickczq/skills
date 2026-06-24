---
name: where-knowledge-lives
description: >-
  Guides where each piece of project knowledge should live — README,
  black-box tests, unit tests, a code comment, a decision record, an
  operational runbook, or nowhere (deleted). Use whenever organizing project
  documentation, writing or reviewing tests, capturing a design decision, or
  unsure whether something belongs in a test, a doc, a comment, or the code
  itself. Especially use when breaking up a large, mixed SPEC.md (one tangling
  behaviors, implementation details, and design decisions) into tests, decision
  records, and code. Also use when a spec drifts from the code, when tests lean
  heavily on mocks, or when documentation duplicates the implementation.
  Triggers even on a bare "write a test," "document this," or "where should this
  go?" — the routing decision is the point.
---

# Where Knowledge Lives

## Core idea

The primary purpose of tests is not coverage — it is to **encode behavior**. A healthy project minimizes the gap between what the software *should* do, what the code *does*, and what future maintainers *believe* it does.

The central move: **whenever knowledge can be expressed as an executable test, prefer the test over prose.** A spec document and the code it describes drift apart; a passing test cannot.

## The routing decision (run this every time)

Whenever you hold a piece of knowledge — a requirement, a rule, a rationale, a how-to — route it through this before deciding where it lands:

1. **Can it be expressed as a test?** → Write a test, not a document.
   - Observable behavior of the whole app (a command succeeds, an error is reported, a file is created, state updates) → **black-box test**. These act as executable specifications.
   - A local rule independent of the app's overall behavior (parsing, normalization, validation, a calculation, a transformation) → **unit test**, defining a local invariant: given these inputs, this output must hold.
   - *If it qualifies as both* (a local rule that is also externally observable), default to the **black-box test** — it pins the contract a user actually sees — and add a unit test only if it catches a class of inputs the black-box test can't reach economically.
2. **Does it explain *why* something exists** — why a dependency was chosen, an alternative rejected, a tradeoff accepted, a limitation kept on purpose? → capture the why, but **a standalone decision record is the last resort, not the default home.** The default home is a **comment next to the code it explains**: the reader is already there, so it can't drift and needs no archaeology to find. Promote to a **separate decision file in `docs/`** only when the why spans several files (belongs to no single code location), will be re-litigated, and matters enough that a future maintainer reversing the decision would get burned without it. A commit message is the *worst* home and not recommended for rationale you expect anyone to need again: retrieving it means knowing the decision exists, guessing which commit, and running `git blame` — an archaeology path nobody takes by instinct, which is the opposite of what good placement does. The test: *would a future maintainer trying to reverse this get burned for not knowing why?* If yes, it needs a place they'll actually look — a comment or a decision file, not the history. Decision files are coarse — one per architectural choice, not one per small tradeoff. Resist a catch-all `decisions.md`: a single file that swallows every why just becomes the next monolith under a new name.
3. **Is it operational know-how** — how to deploy, roll back, rotate a secret, set an env var, recover from a known failure? → **runbook / operational doc** (often `docs/ops/` or the README's operations section). This is neither behavior to assert nor a design rationale; it's a procedure a human runs. Don't force it into a test or a decision file.
4. **Does it explain *what* the software is or *how* to use it**, for users and new contributors? → **README** (avoid implementation detail).
5. **Does it merely restate the implementation?** → **Delete it.** Duplicated knowledge is liability, not documentation.

If you're tempted to write a standalone spec document for something in category 1, stop — that's a test trying to be born as prose.

## A worked example

Suppose a SPEC.md contains this paragraph:

> The `import` command reads a CSV, skips rows whose `email` column is empty,
> trims surrounding whitespace from every field, and writes the cleaned rows to
> `out.json`. We use the `fast-csv` library here rather than the built-in
> parser because the built-in one choked on quoted newlines in our 2023 data
> dump. After import the command prints `Imported N rows`.

Tag and route it sentence by sentence:

- *"reads a CSV … writes the cleaned rows to `out.json`"* and *"prints `Imported N rows`"* → **behavior**. Black-box test: run the command on a fixture CSV, assert `out.json` contents and stdout.
- *"skips rows whose `email` column is empty"*, *"trims surrounding whitespace from every field"* → **behavior**, each its own black-box (or focused unit) test with a crafted input row.
- *"We use `fast-csv` … because the built-in one choked on quoted newlines"* → **decision**. This is borderline: if it's a settled, unlikely-to-be-revisited choice, a one-line comment at the import site (`// fast-csv: std parser chokes on quoted newlines, see 2023 dump`) captures the why with zero new files. Promote to a decision file in `docs/` only if you expect it to be challenged. Default to the comment.
- The bare mention that it uses `fast-csv` *as a fact about how it works* (stripped of the why) → **implementation / duplication**. The code already imports it. Delete.

Result: three or four tests, **zero or one** decision artifact (a comment if that suffices), zero surviving prose. If this paragraph had produced four new doc files, the decomposition went wrong.

## Decomposing a monolithic SPEC.md

A common, painful starting point: one giant SPEC.md written *before* implementation, mixing behaviors, implementation details, and design decisions. Writing prose ahead of code is itself a gap factory — prose is faster to write and easier to lie than code, so the document and the implementation drift apart almost immediately. The fix is to dissolve the monolith into the right layers and let it go.

Work through it like this:

1. **Read it once, tag every passage** as one of: `behavior`, `implementation`, `decision`, `operational`, or `duplication`. Don't reorganize yet — just classify, paragraph by paragraph, so nothing is missed or double-counted.
2. **`behavior` → write the test, confirm it pins the claim, then delete the prose — as one action, not two.** Migration isn't done while the prose still exists; "wrote the test" and "deleted the prose" are not separable steps you can stop between. Don't file it as a TODO. Write a black-box test now (it can start red), implement until it's green, and **before deleting, verify the test actually fails when the claimed behavior is violated** — a test that passes either way has migrated nothing, and deleting the prose then loses the requirement silently. Once the test genuinely pins the same semantics, the prose is redundant; leaving it is drift waiting to happen.
3. **`decision` → capture the why in the lightest place a reader will actually look, then delete the prose.** Default to a code comment at the relevant spot; promote to a standalone decision file in `docs/` only when it spans files and will be re-litigated (see routing step 2). Don't route rationale into commit messages — nobody digs there. Don't mint a file per tradeoff, and don't pour them all into one catch-all doc.
4. **`operational` → move to the runbook, then delete the prose.** Procedures belong with the other ops docs, not in the spec.
5. **`implementation` → delete by default.** The code is the source of truth for how it works. Promote to a decision file *only* when the passage carries a non-obvious *why* (why it's done this surprising way), not a *what*. The plain "what" goes away.
6. **`duplication` → delete outright.** It restates code or another doc; it's pure liability.
7. **Finish by deleting SPEC.md, and prove the file count went down.** Once every passage has landed in a test, a decision artifact, a runbook, the README, or the trash, the file has no remaining content to maintain — so remove it. Then do the arithmetic out loud as an acceptance check, not a nicety: *N tests added; doc files net −M; SPEC.md deleted ✅; the agent file (CLAUDE.md) still an index, not newly swollen with migrated content ✅.* A healthy decomposition usually nets **more tests, fewer doc files**. If you produced more documentation files than you deleted, stop and re-examine — you're over-filing (see routing step 2). A half-emptied SPEC.md husk left lingering counts as not done: it re-accumulates drift and you're back where you started.

Migrate incrementally and keep the suite green between passages, so the decomposition is always in a shippable state rather than one big risky rewrite.

### When the document has external consumers

Before deleting, check who depends on the document. If a spec is consumed outside this repo — referenced by CI, cited in a contract, relied on by another team, published as an interface guarantee — you cannot unilaterally delete it. In that case **don't delete; demote.** Mark the document as non-authoritative, point it at the tests or decision files that now own the knowledge (e.g. "this behavior is specified by `tests/import_test`"), and coordinate removal with its consumers. Deletion is the right end state only for knowledge whose sole audience is this repo's maintainers.

## What each layer answers

- *What does the software do?* → README + black-box tests
- *What rules must remain true?* → unit tests
- *Why was this design chosen?* → a comment next to the code, or a decision record if it spans files — never buried in commit history
- *How do I operate it?* → runbook / ops docs
- *How is it implemented?* → the source code itself

Keeping these separate is what shrinks the spec↔code↔behavior gap. When concerns blur (a spec that documents call sequences, a README full of internals), the gap grows and maintenance cost follows.

## Black-box first, test-first by default

Test from the outside. Verify inputs, observable behavior, outputs, and side effects. Avoid depending on internal modules, private functions, call ordering, or any implementation detail. The litmus test: **a successful refactor should require little or no change to your black-box tests.** If a refactor breaks them, they were testing the wrong thing.

The black-box suite defines the project's **compatibility boundary**, which gives you a clean way to classify any change: if the suite stays green, the change is usually a *refactoring* (internals moved, observable behavior didn't). If the change forces you to update black-box tests, it's usually a *behavior change* (the contract itself moved). This distinction is worth naming out loud in reviews and commit messages — it tells readers whether the externally visible contract shifted.

For behavior, **default to test-first**: write the black-box test before the implementation (let it start red), then implement until it passes. This isn't ceremony — it's what keeps the spec and the code from drifting. The test is simultaneously the specification and the acceptance check, written in a language that can't go stale. Reach for prose to describe a behavior only after you've concluded it genuinely can't be executed as a test. When you're about to write a behavior down instead of testing it, treat that as a signal to stop and write the failing test instead.

This applies even to behavior you know about but haven't built yet: **prefer a failing test over a TODO.** A TODO is prose that rots silently and that nothing enforces; a red test names the missing behavior, fails loudly until it exists, and turns green exactly when the work is done. It's a to-do the build can see.

## Keep implementation out of specs

Specifications describe behavior, which changes rarely — not implementation, which changes constantly. Do not put internal call sequences, module interactions, private APIs, specific algorithms, or exact execution order into a spec. If reviewing a spec that contains these, flag them: they will rot.

## Keep agent files as indexes, not stores

An agent-facing file (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, and the like) has a narrow job: common commands, project rules and conventions, and an **index** — pointers to where the real knowledge lives (the tests, the code, the few standalone decision files). It is not a store for decisions, behaviors, or migrated spec prose. Its size should stay roughly flat over the project's life, because commands and conventions change rarely. The tell: if it grows every time a decision is made, it's quietly turning into the next monolith — the same SPEC.md drift you just escaped, wearing an agent-friendly name. When decomposing a SPEC, route knowledge to its real home and let the agent file *point* at that home, not absorb it.

## Testing dependency hierarchy

Reach for the most real thing that's practical, in this order:

1. **Real systems**
2. **Lightweight test environments** (temporary files, temporary databases, local repositories, isolated environments — run the actual application)
3. **Fakes**
4. **Mocks** (last resort)

Mocks are legitimate when external systems are expensive, unavailable, hard to reproduce failures for, or when isolation is genuinely required. But mocks must not become the primary source of behavioral knowledge — a suite that mostly asserts against mocks tells you the mocks behave, not the software. If you notice that pattern, treat it as a smell and push toward fakes or real environments.

## Optimize for future readers

Underneath every routing decision is one question: **where would a future reader naturally look for this?** Those readers are maintainers, new contributors, future versions of yourself, and increasingly automated agents — and they all reach for the same places by instinct. Someone asking "what does this do?" opens the README or runs the tests; "what must stay true?" looks at the unit tests; "why is it like this?" wants a decision record; "how do I run or recover it?" reaches for the runbook; "how does it work?" reads the code. Put each piece of knowledge where its reader already expects it, and the project becomes legible without a map. Put it somewhere clever instead, and it's effectively lost no matter how well written it is.

## How to apply this skill

This skill is principles, not templates — it tells you *where* knowledge goes, not what format to force it into. When helping:

- **Follow the project's existing conventions.** If decision records already live in `docs/` in a particular shape, match it. If tests have a house style, match it. Detect first, don't impose.
- **State the routing out loud** when it's a judgment call, so the user can correct you: "This is a behavior, so I'll write it as a black-box test rather than adding to the spec doc — sound right?"
- **Prefer adding a test over adding prose** by default, and say so. Only fall back to a document when the knowledge genuinely can't be executed (the "why," the "how to operate," or the "how to use").
- **Make subtraction part of "done," not an optional follow-up.** A migration with the prose still sitting there is unfinished. When you remove prose for a test, confirm the test fails if the behavior breaks; when you remove a document with outside consumers, demote and coordinate rather than delete. And at the end, check the file count actually fell — if your "cleanup" left more files than it removed, you did addition wearing cleanup's clothes.
- **Call out duplication and drift** when you see it: a spec re-describing code, a doc that a test could replace, mocks standing in for behavior.

The goal throughout: make five questions trivially answerable in this project — what it does, what rules hold, why it's built this way, how to operate it, and how it's implemented — each in its proper place, none of them duplicated.
