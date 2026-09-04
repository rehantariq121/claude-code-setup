# Design scope control — HARD CONTRACT for plans, specs, and architecture

Paste this at the top of any request that asks for an implementation plan, a design
specification, an architecture, or a "how should I build X" answer.

It is the counterpart to the `scope-control` block in `~/.codex/AGENTS.md`. That one governs
**diffs against existing code**. This one governs **designing something new**, where "no new
files" and a diff budget do not apply and therefore do not protect you.

---

## The contract

This is not advice you may weigh against other goals. Breaking any rule below makes the plan a
FAILURE, however well-reasoned it is. A smaller plan that obeys this contract always beats a
more thorough plan that does not.

Your known failure mode is designing for a system that does not exist yet: mainnet-grade
infrastructure around a local test build, distributed-systems machinery for a single-node
deployment, recovery paths for failures the target environment cannot produce. Depth of
THINKING is unlimited. Depth of the BUILD is capped. Think as hard as you like, then specify
the smallest system that makes the stated requirement true.

**A request to design a working system is never permission to design a production platform.**

## Step 1 — State the target before you design anything

Open the plan with these four lines, filled in with real values. If you cannot fill one in,
ask instead of assuming the larger number.

```
Environment:   local / staging / production
Users:         <number>
Data volume:   <rows or requests, order of magnitude>
Money at risk: <amount, or "none — valueless test">
```

Every infrastructure decision in the plan must be defensible against those four values. A
mechanism that only pays off at a larger value than the one written there does not go in the
build.

## Step 2 — The one-line justification test

Every non-obvious mechanism in the plan carries one line:

> Without this, **<what breaks>**, **<when>**.

If `<when>` is "when we scale", "when we upgrade", "when we go to production", "when there are
multiple X" — the mechanism is **not in this build**. It goes in the DEFERRED section.

If you cannot write the line, delete the mechanism.

## Step 3 — Banned unless explicitly requested or justified by Step 1

- **Machinery for a second instance that does not exist yet.** No multi-version registries,
  multi-tenant models, multi-region logic, plugin points, or migration frameworks when the
  count today is one and no second is scheduled.
- **Subsystems the target environment cannot exercise.** If the plan describes handling a
  failure that cannot occur in the stated environment, cut it. You cannot test it, so you do
  not know it works, so it is not an asset.
- **Incremental recovery where a full rebuild is cheap.** Default to "delete and rebuild from
  source of truth". Incremental repair, partial rollback, and resume-from-checkpoint need a
  measured rebuild time that makes them necessary.
- **Versioned or negotiated contracts with one known client.** No URL versioning, schema
  documents, content negotiation, or deprecation policy when your own app is the only caller.
- **Temporal, versioned, or append-only storage.** Default is: rows overwrite in place. History
  comes from replaying the source of truth. Anything else requires a stated requirement to
  query the past.
- **Consistency machinery beyond one request.** No snapshot tokens, authenticated cursors, or
  cross-request anchoring unless a stated requirement makes a torn read actually harmful.
- **Observability beyond structured logs.** Distributed tracing, metrics pipelines, and alert
  routing require someone on-call. Name them, or cut it.
- **Ceremony proportional to imagined value, not real value.** Multi-signature approvals,
  time delays, and staged rollouts must match the Money at risk line. Zero value means zero
  ceremony.
- **Abstractions with fewer than three real call sites.** Duplication is accepted. Premature
  abstraction is not.
- **Unrequested extras.** Export features, analytics windows, batch operations, admin tooling,
  and convenience APIs that nobody asked for.

## Step 4 — Split the output

Every plan ends with two lists, both with time estimates:

```
BUILD NOW      <what, and why it is required by Step 1>     <estimate>
DEFERRED       <what, and the trigger that brings it back>  <estimate>
```

Nothing is silently dropped and nothing is silently included. The DEFERRED list is how you keep
good ideas without paying for them today.

## Step 5 — Estimate before you submit

Write the BUILD NOW estimate. If it exceeds the budget you were given, cut before submitting —
do not submit an over-budget plan with a note. If no budget was given, state the estimate and
say what you would cut first if it is too high.

## Security carve-out — this rule does NOT cut safety

Simplicity is never a reason to weaken any of the following. These are correctness, not
gold-plating:

- Authentication, authorization, and session handling
- Payment, balance, and money-movement logic, and its arithmetic
- Secret and key handling — never in code, never in logs, never held by a service that does
  not need it
- Input validation and injection defenses at every trust boundary
- Conservation and accounting invariants, and the tests that prove them
- Whatever safety control the stated requirements or the law actually demand

When simplicity and safety conflict, safety wins and you say so in the plan.

If the target environment genuinely permits a weaker control — a single owner on a valueless
local test chain, for example — you may simplify **only** with an explicit written warning in
the plan naming what must be restored before real value is involved. Never carry a test-only
simplification into a production design.

## What good looks like

- The plan is buildable and the estimate is honest.
- Every mechanism has its one-line justification.
- The BUILD NOW list is boring.
- The DEFERRED list is where the interesting engineering went, with triggers on it.
- Business rules, money arithmetic, and safety controls are specified exactly, in full detail.
  Precision there is not over-engineering — that is the product.

---

## Short version, for pasting inline

> Before designing: state environment, user count, data volume, and money at risk. Justify
> every mechanism with one line — "without this, X breaks, when". If the answer is "when we
> scale / upgrade / go to production", it goes in a DEFERRED list, not the build. No machinery
> for a second instance that does not exist. No subsystem the target environment cannot
> exercise. Default to full rebuild over incremental recovery, overwrite-in-place over
> versioned storage, no API versioning with one client, structured logs over tracing. End with
> BUILD NOW and DEFERRED lists, both estimated. Never simplify away authentication, money
> logic, secret handling, input validation, or accounting invariants — specify those in full
> detail; that is the product, not over-engineering.
