# Full Splendor Actions Plan

## Planning Goal

Implement support for full base-game Splendor token actions in this repository after research is accepted, with emphasis on:

- take tokens and return overflow in the same turn
- reserve plus gold plus overflow cleanup in the same turn
- removal of standalone discard as a normal turn action

This plan is intentionally implementation-oriented but does not make code changes yet.

## Proposed Plan

### Ticket 1

Title: Lock the target rules contract

Priority: P0

Feasibility: High

Proposed approach:

- Confirm the exact desired rules scope before coding.
- Treat the following as in scope:
  - take up to 3 different tokens, then forced cleanup if above 10
  - take 2 same-color tokens when the bank had at least 4 before taking, then forced cleanup if above 10
  - reserve visible or blind card, take gold if available, then forced cleanup if above 10
  - no standalone discard action in normal play
  - no always-legal pass action in normal play
- Decide whether noble-choice correction is in or out of scope for the same ticket stream.

Potential risks:

- Rules ambiguity around pass or multi-noble handling causes rework later.

### Ticket 2

Title: Design the new turn-state model

Priority: P0

Feasibility: Medium

Proposed approach:

- Introduce an explicit Splendor subphase for overflow cleanup.
- Add enough board state to encode:
  - whether cleanup is pending
  - which player is still resolving the same turn
  - if necessary, how many tokens still must be returned
- Preserve the rule that the active player does not rotate until cleanup finishes.

Potential risks:

- The current board tensor layout is dense and Numba-jitclass-backed, so adding state can cascade into indexing mistakes.

### Ticket 3

Title: Redefine legal actions by phase

Priority: P0

Feasibility: Medium

Proposed approach:

- Normal phase:
  - buy visible
  - reserve visible/deck
  - buy reserved
  - take-token actions that may temporarily exceed 10
- Cleanup phase:
  - only legal token-return actions
  - only enough return choices to move toward or reach the 10-token cap
- Remove standalone discard and unconditional pass from the normal phase.

Potential risks:

- If cleanup actions remain encoded with the current discard action IDs, some invalid combinations may still leak through.

### Ticket 4

Title: Update move application and round advancement

Priority: P0

Feasibility: Medium

Proposed approach:

- Change `make_move()` and helper methods so:
  - token taking or reserve-plus-gold can leave the player temporarily above 10
  - the same player continues if cleanup is pending
  - round count advances only when the whole turn is complete
- Ensure reserve gold logic grants gold whenever available, even if that triggers cleanup.

Potential risks:

- Off-by-one bugs in turn rotation or round counting can silently break MCTS and end-game evaluation.

### Ticket 5

Title: Update action presentation and policy contract

Priority: P1

Feasibility: Medium

Proposed approach:

- Update move strings and human UI so cleanup choices appear as forced continuation actions, not normal turns.
- Decide one of:
  - keep the current action count and reinterpret part of the action space by phase
  - expand/change action size and update the policy head accordingly
- Make checkpoint compatibility expectations explicit.

Potential risks:

- Pretrained models may become unusable if action semantics or dimensions change.

### Ticket 6

Title: Add focused rules tests for full actions

Priority: P0

Feasibility: High

Proposed approach:

- Add deterministic tests for:
  - take at 9, overflow by 1, then return 1
  - take at 10, overflow by 3, then return 3
  - reserve at 10 with gold available, take gold, then return 1
  - same-color take requires bank count >= 4 before the take
  - standalone discard not legal during normal phase
  - pass not legal during normal phase if legal actions exist
- Add regression tests for buy and reserve flows that should remain unchanged.

Potential risks:

- Without tests around intermediate phase states, the cleanup design can look correct while still masking invalid actions.

### Ticket 7

Title: Reassess training and backward compatibility

Priority: P1

Feasibility: Medium

Proposed approach:

- Audit where `getActionSize()` is assumed downstream.
- Decide whether:
  - pretrained checkpoints are explicitly invalidated
  - a compatibility shim is needed
  - Splendor must be retrained after merge

Potential risks:

- A logic-only merge without retraining guidance can leave the repo in a misleading partially broken state for users.

## Recommended Ordering

1. Ticket 1: Lock the target rules contract.
2. Ticket 2: Design the new turn-state model.
3. Ticket 3: Redefine legal actions by phase.
4. Ticket 4: Update move application and round advancement.
5. Ticket 6: Add focused rules tests in parallel with Tickets 3 and 4.
6. Ticket 5: Update action presentation and policy contract.
7. Ticket 7: Reassess training and backward compatibility.

## Plan Review

### Three potential errors, risks, or counterexamples

1. The plan may underestimate cleanup branching.

Counterexample:

- A player at 10 takes 3 different tokens and must return 3.
- If cleanup is modeled only with the current discard actions, the engine cannot express all legal return sets in one step.

Why this matters:

- A "reuse current give-back actions" design can still be rules-incomplete.

2. The plan may break turn accounting if cleanup is treated as a new turn.

Counterexample:

- The engine increments round count after every `make_move()` call today.
- If overflow cleanup uses several actions, naive reuse of that path will overcount rounds and rotate players too early.

Why this matters:

- MCTS values, end-game checks, and self-play traces can all become incorrect even if move legality looks right.

3. The plan may ignore policy and checkpoint fallout.

Counterexample:

- If action meanings change but `SplendorNNet` and saved checkpoints are not updated in lockstep, the model can emit logits for obsolete semantics such as standalone discard or pass.

Why this matters:

- The repo can become internally inconsistent: valid logic with invalid model behavior.

## Revised Plan With Explicit Mitigations

### Revision A

Mitigation for cleanup branching:

- Do not assume the current discard action family is sufficient.
- In the design phase, explicitly choose between:
  - multi-step cleanup with repeated discard micro-actions and stable same-player control
  - a larger cleanup action family that can express all required return counts
- Add test cases that require returning 3 tokens, not just 1.

### Revision B

Mitigation for turn-accounting errors:

- Treat "turn complete" as a separate concept from "one action processed".
- Add explicit state and tests for:
  - active player before cleanup
  - active player after cleanup
  - round counter before and after cleanup
- Do not increment round count until the entire Splendor turn resolves.

### Revision C

Mitigation for policy/checkpoint inconsistency:

- Add a dedicated checkpoint-compatibility decision as a required design output, not an afterthought.
- Gate implementation on one explicit choice:
  - preserve action size and remap semantics carefully, or
  - change action size and document checkpoint invalidation plus retraining needs
- Add a smoke test that validates policy output size against `getActionSize()`.

## Final Assessment Against Requested Criteria

Ticket/task granularity:

- Good after revision. The plan is split into engine contract, state model, move legality, move application, UI/policy, testing, and training impact.

Potential risks:

- Medium to high unless phase modeling, round accounting, and checkpoint impact are handled explicitly.

Feasibility:

- Feasible, but not as a validator-only patch. This is a moderate engine-contract change.

Priority:

- Highest priority items are the rules contract, phase model, legality, application, and tests.
- UI/policy and training compatibility follow immediately after because they are coupled to the action contract.

Proposed approach:

- Preferred approach is an explicit overflow-cleanup phase with same-player continuation, backed by tests and an explicit checkpoint-compatibility decision.
