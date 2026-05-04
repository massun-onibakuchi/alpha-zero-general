# Full Splendor Actions Research

## Scope

This document studies "full Splendor actions" in detail using the reference spec and reference implementation, then compares that contract to the Splendor implementation in this repository.

The research goal is to answer:

- what actions are supported by the reference rules and engine
- how overflow token return is modeled there
- how noble choice, reserve gold, and buy spend are encoded
- what this repo currently supports instead
- what the exact gap is between the two systems

This is research only. No implementation is included here.

## Primary Sources

Reference rules and implementation:

- [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:1)
- [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:1)
- [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:1)
- [engine/api.py](/tmp/splendid-splendor/engine/api.py:1)
- [tests/test_rules_tokens.py](/tmp/splendid-splendor/tests/test_rules_tokens.py:1)
- [tests/test_rules_reserve.py](/tmp/splendid-splendor/tests/test_rules_reserve.py:1)
- [tests/test_rules_buy.py](/tmp/splendid-splendor/tests/test_rules_buy.py:180)

Current repo implementation:

- [README.md](/workspace/.worktrees/research-full-splendor-actions-ref/README.md:29)
- [splendor/SplendorLogic.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogic.py:6)
- [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:53)
- [splendor/SplendorPlayers.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorPlayers.py:28)
- [splendor/SplendorNNet.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorNNet.py:100)

## Executive Summary

The reference repo defines full Splendor actions as structured, fully specified turn actions. A legal action already includes every choice required to complete the turn correctly:

- exact tokens taken
- exact tokens returned if the take or reserve would overflow past 10
- exact spend vector for buys when more than one spend decomposition exists
- exact noble choice when more than one noble becomes eligible

That means the reference system does not model overflow cleanup as a hidden post-action rule and does not model discard as a separate action family. Overflow return is part of action legality itself. Noble choice is also part of action semantics.

By contrast, this repo currently encodes a simplified 81-action flat space that:

- forbids taking and returning in one turn
- exposes standalone give-back actions as top-level turns
- exposes pass as an always-legal top-level move
- suppresses reserve gold when the player is already at 10 tokens
- awards all nobles earned, not one chosen noble

The main conclusion is that "support full Splendor actions" is not just "allow overflowed coins to be returned." In the reference design, full actions are a richer action contract. The current repo differs at the levels of rules semantics, action encoding, UI assumptions, and model contract.

## What The Reference Rules Require

### Turn structure

The reference rules define exactly six legal move families:

1. Take 1 to 3 different colored tokens.
2. Take 2 tokens of the same color.
3. Reserve a visible development card.
4. Reserve the top card of a hidden deck.
5. Buy a visible development card.
6. Buy a reserved development card.

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:15).

Two important negatives are explicit:

- no pass action exists when at least one legal move exists
- if a legal move creates a successor where the next player has no legal move, the engine closes out the game as terminal instead of inventing a pass

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:28) and [engine/api.py](/tmp/splendid-splendor/engine/api.py:45).

### Token limit is enforced inside action legality

The reference rules state:

- a player may never end a turn above 10 tokens
- if a move would overflow, the action must include the exact return-token choice needed to end at 10 or below
- return-token obligations are part of action legality, not hidden post-processing

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:48).

This is the most important semantic difference from this repo.

### Reserve rules include overflow-aware gold handling

When reserving:

- if gold is available, exactly 1 gold is taken
- if gold is not available, reserve remains legal
- if taking that gold would overflow the token limit, the reserve action must include the return choice

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:56).

That means "reserve while at 10 with gold available" is legal in the reference model as long as the action specifies the matching return.

### Noble choice is part of turn semantics

The reference rules explicitly require:

- exactly one noble must be claimed if one or more are eligible
- if multiple nobles are eligible, the choice is part of the action semantics for that turn

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:100).

This is a second major semantic difference from the current repo, which currently grants all eligible nobles.

## How The Reference Repo Encodes Full Actions

### Structured action object

The reference action object contains:

- `action_type`
- `take_tokens`
- `spend_tokens`
- `return_tokens`
- `card_ref`
- `tier_ref`
- `noble_index`
- `schema_version`

See [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:29).

This is the core design choice: actions are not flat integer IDs with implicit meaning. They are replay-safe structured objects that fully describe the move.

### Overflow return is first-class action data

`return_tokens` is normalized and validated as part of action construction. See [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:42).

For reserve actions:

- reserve actions may take at most one gold token
- reserve actions cannot take colored tokens
- reserve actions may return at most one token

See [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:62).

For buy actions:

- buy actions cannot take bank tokens
- buy actions cannot return bank tokens
- buy actions may include `spend_tokens`

See [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:93).

For token takes:

- `TAKE_DIFF` must take 1 to 3 distinct non-gold colors
- `TAKE_SAME` must take exactly 2 of one non-gold color

See [engine/actions.py](/tmp/splendid-splendor/engine/actions.py:121).

### Supported action types in the reference implementation

The reference engine supports exactly these action types:

- `TAKE_DIFF`
- `TAKE_SAME`
- `RESERVE_VISIBLE`
- `RESERVE_BLIND`
- `BUY_VISIBLE`
- `BUY_RESERVED`

See [engine/types.py](/tmp/splendid-splendor/engine/types.py:16).

There is no discard action type and no pass action type.

## How The Reference Repo Generates Legal Full Actions

### Token actions

Token actions are generated by `generate_token_actions()`. See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:200).

Behavior:

- for each legal 1-color, 2-color, or 3-color distinct take, generate actions with all required legal return vectors
- for each color with at least 4 tokens in bank, generate the two-of-a-kind take with all required legal return vectors
- no gold is ever taken by token actions

The helper `_action_with_returns()` computes the post-take inventory, computes overflow beyond `TOKEN_LIMIT`, enumerates all exact return vectors for that excess, and emits one action per return vector. See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:167).

This means token-overflow branching is handled by action enumeration, not by a cleanup subturn.

### Reserve actions

Reserve actions are generated by `generate_reserve_actions()`. See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:230).

Behavior:

- if the player already holds 3 reserved cards, no reserve actions are generated
- if gold is available, reserve actions include `take_tokens = (0, 0, 0, 0, 0, 1)`
- if gold is unavailable, reserve actions use an empty take vector
- visible and blind reserves both branch over any required return vectors

The tests make the intended semantics explicit:

- initial state generates 12 visible + 3 blind reserve actions
- reserve at 10 can require one-token return
- when a return is needed, actions are ordered so returning already-held colored tokens is preferred before returning the just-taken gold
- reserve without gold remains legal

See [tests/test_rules_reserve.py](/tmp/splendid-splendor/tests/test_rules_reserve.py:25), [tests/test_rules_reserve.py](/tmp/splendid-splendor/tests/test_rules_reserve.py:165), and [tests/test_rules_reserve.py](/tmp/splendid-splendor/tests/test_rules_reserve.py:174).

### Buy actions

Buy actions are generated by `generate_buy_actions()`. See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:280).

Important behaviors:

- spend vectors are enumerated when multiple colored/gold payment combinations are possible
- if buying a card makes one or more nobles eligible, buy actions branch by `noble_index`
- the resulting action already includes both the payment choice and noble choice

See `_payment_vectors_for_card()` and `_buy_actions_for_card()` in [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:62) and [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:83).

The reference tests explicitly assert branching when multiple nobles become eligible in the same buy. See [tests/test_rules_buy.py](/tmp/splendid-splendor/tests/test_rules_buy.py:180).

## How The Reference Repo Applies Actions

### One action equals one completed turn

The reference engine still keeps "one action equals one turn." The difference is that the action already contains overflow-return and noble-choice details, so no extra cleanup turn is needed.

Token action application:

- add `take_tokens`
- subtract `return_tokens`
- update bank symmetrically
- advance to next player

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:327).

Reserve action application:

- add reserve gold if encoded in the action
- subtract `return_tokens`
- move the chosen visible or blind card into reserve
- refill the market if needed
- advance to next player

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:374).

Buy action application:

- subtract `spend_tokens`
- update bank
- add the purchased card to the player's purchased set
- remove the chosen noble if `noble_index` is provided
- refill market or close reserve gap as appropriate
- advance to next player

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:447).

### Deadlock handling replaces pass

After applying an action, the API checks whether the next active player has any legal actions. If not, the successor is marked terminal rather than generating a pass action. See [engine/api.py](/tmp/splendid-splendor/engine/api.py:45).

This is a deliberate substitute for pass semantics in pathological states.

## Non-Obvious Intricacies In The Reference Model

### "Full actions" means more than overflow support

The phrase "full actions" in the reference repo includes at least four separate dimensions:

1. Exact token take vectors.
2. Exact overflow return vectors.
3. Exact buy spend vectors when multiple payments are possible.
4. Exact noble choice when multiple nobles are eligible.

Overflow support is only one part of the action richness.

### The reference design chooses atomic branching over cleanup subphases

There are two plausible designs for overflow:

- a same-player cleanup subphase
- atomic turn actions that already include the return choice

The reference repo clearly chooses the second design. `_action_with_returns()` enumerates legal return vectors and turns them into distinct legal actions. See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:167).

That means the reference engine does not need extra turn-phase state for overflow cleanup.

### Some legal actions are intentionally "no-op-ish"

Because the return vector can include the same tokens just taken, the reference legal-action set can contain actions where `take_tokens == return_tokens`. The heuristic tests explicitly recognize such actions and score them below state-changing alternatives. See [tests/test_baseline_heuristics.py](/tmp/splendid-splendor/tests/test_baseline_heuristics.py:145).

This matters because removing pass does not mean every legal action changes resources materially.

### Legal-action ordering is deterministic

The reference tests lock down deterministic family ordering and serialization round-tripping. See [tests/test_legal_action_determinism.py](/tmp/splendid-splendor/tests/test_legal_action_determinism.py:1) and [tests/test_actions.py](/tmp/splendid-splendor/tests/test_actions.py:1).

That matters for:

- replay stability
- search consistency
- ML dataset generation
- policy target ordering

## What This Repo Currently Supports Instead

### Explicitly documented limitation

This repo's README states that Splendor does not allow taking gems and giving some back in the same turn, and that players are limited to either taking 1 to 3 gems or giving back 1 to 2 gems. See [README.md](/workspace/.worktrees/research-full-splendor-actions-ref/README.md:29).

### Current action model

This repo hardcodes an 81-action flat space:

- 12 buy-visible actions
- 15 reserve actions
- 3 buy-reserved actions
- 30 take-gem actions
- 20 give-back actions
- 1 pass action

See [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:53).

This is already incompatible with the reference action taxonomy because the current repo has explicit discard and pass families, while the reference repo has neither.

### Current valid-move semantics

`valid_moves()` in this repo:

- includes standalone discard actions
- always enables pass
- rejects take actions that would exceed 10 tokens

See [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:180), [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:422), and [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:446).

### Current reserve behavior

`_reserve()` only grants gold when:

- gold is available, and
- the player currently has 9 or fewer tokens

See [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:382).

That is directly incompatible with the reference rules and tests.

### Current move strings and UI semantics

The current move strings explicitly include "give back" actions. See [splendor/SplendorLogic.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogic.py:35).

The human UI also conditionally surfaces give-back actions as part of the player menu. See [splendor/SplendorPlayers.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorPlayers.py:28).

This means the current UX contract also teaches discard as a primary move family.

### Current policy-head semantics

The current policy head emits logits for:

- reserve-from-deck: 3
- get-gems: 30
- give-gems: 20
- pass: 1

See [splendor/SplendorNNet.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorNNet.py:110).

This is a second-order but important issue: even if the rules logic were updated, the policy contract would still reflect the simplified game unless it changed too.

## Direct Gap Analysis: Reference vs Current Repo

### Gap 1: overflow return is encoded vs forbidden

Reference repo:

- overflow return is encoded into legal actions using `return_tokens`

Current repo:

- take actions are invalidated if they would exceed 10

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:167) versus [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:422).

### Gap 2: reserve-at-10-with-gold is legal vs blocked

Reference repo:

- reserve takes gold whenever available and includes the required return if necessary

Current repo:

- reserve suppresses gold when the player is already above 9 tokens

See [tests/test_rules_reserve.py](/tmp/splendid-splendor/tests/test_rules_reserve.py:165) versus [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:398).

### Gap 3: discard is implicit-in-action vs top-level action family

Reference repo:

- no discard action type exists
- discard is represented only through `return_tokens` inside token/reserve actions

Current repo:

- 20 top-level give-back actions exist

See [engine/types.py](/tmp/splendid-splendor/engine/types.py:16) versus [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:78).

### Gap 4: pass is absent vs always legal

Reference repo:

- no pass action family exists
- deadlock closes the game instead

Current repo:

- action 80 is always legal

See [rules_base_splendor.md](/tmp/splendid-splendor/docs/specs/rules_base_splendor.md:28), [engine/api.py](/tmp/splendid-splendor/engine/api.py:56), and [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:187).

### Gap 5: noble choice is explicit vs all nobles are granted

Reference repo:

- buy actions branch by `noble_index` when multiple nobles are eligible
- exactly one noble is claimed

Current repo:

- `_give_nobles_if_earned()` loops through all nobles and grants every eligible noble

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:83), [tests/test_rules_buy.py](/tmp/splendid-splendor/tests/test_rules_buy.py:191), and [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:465).

### Gap 6: buy payment semantics are structured vs implicit-flat

Reference repo:

- buy actions may branch into multiple legal `spend_tokens` variants

Current repo:

- buy logic resolves payment internally and action IDs do not encode spend choice

See [engine/rules.py](/tmp/splendid-splendor/engine/rules.py:62) versus [splendor/SplendorLogicNumba.py](/workspace/.worktrees/research-full-splendor-actions-ref/splendor/SplendorLogicNumba.py:344).

This may or may not need to be replicated in this repo for the current ticket, but it is part of what "full actions" means in the reference implementation.

## What "Supported Actions" Means In The Reference Repo

If the request is interpreted literally as "what actions are supported there," the accurate answer is:

### Supported action families

- `TAKE_DIFF`
- `TAKE_SAME`
- `RESERVE_VISIBLE`
- `RESERVE_BLIND`
- `BUY_VISIBLE`
- `BUY_RESERVED`

### Supported action fields

- `take_tokens`
- `spend_tokens`
- `return_tokens`
- `card_ref`
- `tier_ref`
- `noble_index`

### Supported semantic variants

- token-take actions with zero or more required returns
- reserve actions with or without gold, and with or without required returns
- buy actions with one or more legal spend decompositions
- buy actions with zero or one noble claim choice

This is a significantly richer notion of "supported actions" than the flat move-index contract in the current repo.

## Important Conclusions For Future Work

1. The reference repo solves overflow by enlarging action semantics, not by adding a cleanup phase.
2. The current repo currently models a different game, not just a missing overflow edge case.
3. Matching the reference behavior fully would require more than allowing take-then-return:
   - discard must stop being a top-level move family
   - pass must stop being a normal action
   - reserve gold must allow overflow with explicit return
   - multi-noble choice must be fixed
   - the action/model contract likely needs redesign
4. If the target is "match the reference full-action semantics," then overflow return, buy payment branching, and noble branching should be treated as one coherent action-contract problem.

## Bottom Line

The reference docs and engine define full Splendor actions as complete, deterministic, replay-safe turn objects. Overflow return is not an afterthought there; it is part of the move itself. The same is true for reserve gold overflow and noble choice.

This repo currently uses a simpler flat action space that explicitly forbids take-and-return on the same turn and exposes discard and pass as top-level moves. That means the current gap is deeper than "support overflowed coins." The gap is between two different action models.
