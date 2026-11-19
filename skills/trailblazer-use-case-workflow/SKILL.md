---
name: trailblazer-use-case-workflow
description: Use when starting to implement a new feature or use case with Trailblazer on Rails, designing an Operation's inputs/outcomes, adding failure branches or side effects, extracting a shared Activity, or preparing a Trailblazer change for review.
---

# Trailblazer Use Case Workflow

Step-by-step process from use case description to review. Source guide: `docs/rails/use-case-workflow.md`. Structure and naming rules live in `trailblazer-rails-standard`; tests in `trailblazer-minitest`.

Book lessons (*Building your own authentication library with Trailblazer*) are process guidance only; specific security/business rules in the book are not company policy. Verify DSL, wiring, `Subprocess` and test commands against `Gemfile.lock`.

## Steps

1. **Describe use case and outcomes** before code: who triggers it and why; allowed input; actor/dependencies; data/side effects created; outcomes to distinguish (success, invalid input, not found, not permitted, conflict, system error); HTTP response per outcome. One Operation = one named business action (`Register`, `Publish`), independent of controller actions (jobs/scripts may call it too).
2. **Choose boundaries and interface.** Controller = HTTP only. Operation = ordering and outcomes, no implicit request/session. Contract = payload + validation. Model = persistence/invariants. Write down the interface first: input keywords, stable result data callers read, terminuses. Only expose `ctx` variables callers need; renaming/removing a used output is an interface change.
3. **Implement the success track, then add failures.**
   - Order: find object → authorize → validate → write → side effects.
   - Each step does one thing, uses keyword args.
   - For each failure decide the stop point and what the caller may learn; ensure a later failure handler doesn't overwrite an earlier error.
   - `Subprocess(...)` for child Operation/Activity; wire custom terminuses and input/output mapping explicitly and test them.
   - Don't rescue exceptions for ordinary validation. Rescue only expected, identifiable ones (e.g. unique-constraint conflict) and convert to an outcome; let unexpected ones surface.
4. **Persistence and side effects.** Validation for user messages, DB constraints as last line (validation ≠ unique index; consider races). Define transaction boundary. Email/job/external call: define retry, idempotency, and what happens if it fails after commit. No orchestration in model callbacks.
5. **Test outcome and effect** (see `trailblazer-minitest`): success track, each important failure (no partial writes), meaningful edge cases (not found, denied, unique conflict, expired token, repeat call), side effects enqueued/produced, plus request/integration test for status/redirect/render/filtered params. Assert public outcomes, not private step order or every `ctx` temp. Use `wtf?`/trace to debug, never to assert.
6. **Refactor on evidence of sharing.** Extract only when two flows share responsibility *and* semantics; if only a few lines look alike but policy differs, keep separate or extract a small pure function. After extracting: test the child independently and at least one integrated caller. `input`/`output` mapping limits or renames data; don't use it to hide an unclear contract. Wire every outcome beyond `success`/`failure` in the caller.
7. **Finish and review.**
   - [ ] Components in right concept, plural namespaces
   - [ ] Controller only bridges HTTP and Operation; no repeated business branching
   - [ ] Input, output, access, failures, side effects stated
   - [ ] Tests cover success, key failures, data effect, HTTP responses
   - [ ] Gem syntax and test command verified against repo (`Gemfile.lock`, test config, CI)
   - [ ] No new duplicated code; shared component has interface and tests
   - [ ] Logs/traces don't leak passwords, tokens or private payloads

   Run targeted tests, then the project's suite/CI. A green trace or one unit test does not replace integration tests, security review or side-effect checks.
