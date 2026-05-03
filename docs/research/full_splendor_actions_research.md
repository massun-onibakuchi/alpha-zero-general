# Full Splendor Actions Research

## Scope

This document analyzes what "support full Splendor actions" means for this repository, what the current implementation supports today, and what must change to align the engine with base-game token rules.

This is research only. No implementation is included here.

## Request Interpreted

The requested target is:

- support full base-game Splendor actions
- specifically close the gap where the repo does not support taking tokens and then returning overflowed tokens in the same turn
- study the current supported actions and the full-action model in the reference material

The request also points to:

- `docs/specs/rules_base_splendor.md`
- reference repo: `https://github.com/massun-onibakuchi/splendid-splendor`

At the time of this research on 2026-05-03 UTC:

- `docs/specs/rules_base_splendor.md` is not present in this checkout
- the reference repo could not be fetched from this environment over HTTPS and GitHub returned `404` when queried directly

Because of that, this document uses the local codebase plus public Splendor rules references for rules clarification, and treats the reference repo as unavailable.

## Sources

Local repository:

- [README.md](/workspace/.worktrees/full-splendor-actions/README.md)
- [splendor/SplendorLogicNumba.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py)
- [splendor/SplendorLogic.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogic.py)
- [splendor/SplendorPlayers.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorPlayers.py)
- [splendor/SplendorNNet.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorNNet.py)
- [splendor/SplendorGame.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorGame.py)

Public rules references used for clarifying base-game behavior:

- Official rules summary: https://officialgamerules.org/game-rules/splendor/
- Dized Splendor rules: https://rules.dized.com/game/vdDSzuu4RsC8F45PsW0TKw
- Stack Exchange: token return clarification: https://boardgames.stackexchange.com/questions/34695/returning-tokens-for-the-10-token-limit-which-ones
- Stack Exchange: same-color take clarification: https://boardgames.stackexchange.com/questions/42304/can-you-take-2-gems-of-the-same-kind-for-2-3-players

## Executive Summary

The current Splendor implementation does not merely miss one edge case. It encodes a different action model:

- taking gems is one turn
- giving back gems is a different top-level turn
- pass is always available
- reserving only grants gold when the player is at 9 or fewer tokens

Base-game Splendor works differently:

- token return is not its own action
- overflow cleanup is part of the same turn after taking gems or reserving and taking gold
- the player may choose which tokens to return, including tokens they just took
- reserving while already at 10 tokens is still legal if gold is available, as long as the player returns to 10 before the turn ends
- there is no normal standalone discard action and no normal standalone pass action

That gap is baked into:

- action indexing
- valid move generation
- move application
- human move display
- neural-network policy output layout
- likely training data and pretrained checkpoints

The main engineering conclusion is that this should be treated as an action-contract change, not a small validator fix.

## Current Repo Behavior

### README-level limitation

The repository documents the simplification explicitly in [README.md](/workspace/.worktrees/full-splendor-actions/README.md): Splendor does not allow taking gems and giving some back in the same turn.

### Flat 81-action contract

The Splendor engine hardcodes an 81-action space in [splendor/SplendorLogicNumba.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py):

- 12 buy visible card actions
- 15 reserve actions
- 3 buy reserved-card actions
- 30 take-token actions
- 20 give-back actions
- 1 pass action

The contract is stated in comments at [splendor/SplendorLogicNumba.py:53](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:53) and enforced by `action_size()` at [splendor/SplendorLogicNumba.py:95](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:95).

### Valid-move generation

`valid_moves()` at [splendor/SplendorLogicNumba.py:180](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:180):

- exposes take-token actions
- exposes standalone give-back actions
- always enables pass at action `80`

That already deviates from base rules even before looking at overflow handling.

### Token taking is blocked at 10+

The current implementation rejects any token take that would exceed 10 total tokens:

- `_valid_get_gems()` at [splendor/SplendorLogicNumba.py:422](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:422)
- `_valid_get_gems_identical()` at [splendor/SplendorLogicNumba.py:429](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:429)

This forbids legal base-game turns such as:

- having 9, taking 2 same-color, returning 1
- having 10, taking 3 different, returning 3
- having 8, taking 3 different, returning 1

### Reserve does not allow overflow via gold

`_reserve()` at [splendor/SplendorLogicNumba.py:382](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:382) only grants gold when:

- gold is available, and
- the player currently has `<= 9` total tokens

The check is at [splendor/SplendorLogicNumba.py:398](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:398).

This blocks legal base-game turns where a player:

- reserves a card
- takes 1 gold
- temporarily goes above 10
- immediately returns one token

### Standalone discard is treated as a normal turn

The action taxonomy includes:

- `60..74`: give back up to 2 different gems
- `75..79`: give back 2 identical gems

Those are described in [splendor/SplendorLogicNumba.py:78](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:78) and formatted in [splendor/SplendorLogic.py:35](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogic.py:35).

In base Splendor, discard is not a standalone primary action. It exists only as post-action cleanup after overflow.

### Pass is always legal

`valid_moves()` sets `result[80] = True` at [splendor/SplendorLogicNumba.py:187](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:187).

That makes pass a normal move. Base Splendor does not normally include a pass action in its turn menu.

### UI and move-string assumptions

The human player UI in [splendor/SplendorPlayers.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorPlayers.py) assumes:

- take and give are separate top-level actions
- move display is one action per completed turn
- showing discard actions is a normal part of the menu

See especially:

- [splendor/SplendorPlayers.py:28](/workspace/.worktrees/full-splendor-actions/splendor/SplendorPlayers.py:28)
- [splendor/SplendorLogic.py:6](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogic.py:6)

### Policy head assumptions

The policy head in [splendor/SplendorNNet.py:100](/workspace/.worktrees/full-splendor-actions/splendor/SplendorNNet.py:100) explicitly emits logits for:

- reserve from deck: 3
- get gems: 30
- give gems: 20
- pass: 1

See [splendor/SplendorNNet.py:111](/workspace/.worktrees/full-splendor-actions/splendor/SplendorNNet.py:111) and [splendor/SplendorNNet.py:131](/workspace/.worktrees/full-splendor-actions/splendor/SplendorNNet.py:131).

This means the action-space simplification is also part of the NN architecture.

## Base Splendor Action Semantics

### Primary actions

On a turn, a player chooses one primary action:

1. Take up to 3 different gem tokens.
2. Take 2 tokens of the same color, only if at least 4 were present in that stack before taking.
3. Reserve 1 face-up card or 1 blind card from a deck, and take 1 gold token if available.
4. Buy 1 face-up card or 1 previously reserved card.

### Token-limit rule

The player cannot end their turn with more than 10 tokens total, including gold.

If a primary action causes overflow:

- the player must immediately return tokens until they are back to 10
- the returned tokens are chosen by the player
- the returned tokens may include newly taken tokens or tokens already owned before the action

This matters because "take" and "return" are one atomic turn from the rules perspective, but a multi-step decision from the implementation perspective.

### Reserve and gold interaction

Reserving still grants gold when available even if the player is already at 10, because the player can resolve overflow by returning one token before the turn ends.

### Same-color take rule

The condition for taking 2 of the same color is checked before the take. The bank must have at least 4 in that color before removal.

The current repo already matches that specific part at [splendor/SplendorLogicNumba.py:431](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:431).

## Supported Actions: Current vs Full

### Currently supported in this repo

- buy visible card
- reserve visible card
- reserve blindly from deck
- buy reserved card
- take 1 to 3 different tokens, but only if the move ends at 10 or below
- take 2 same-color tokens, but only if the move ends at 10 or below
- give back 1 to 2 tokens as a top-level turn
- pass as a top-level turn

### Full actions needed for base Splendor

- buy visible card
- reserve visible card
- reserve blindly from deck
- buy reserved card
- take 1 to 3 different tokens, then if needed return overflow to 10 in the same turn
- take 2 same-color tokens when bank had at least 4, then if needed return overflow to 10 in the same turn
- reserve and take gold if available, then if needed return overflow to 10 in the same turn

### Actions that should not remain top-level base actions

- standalone give-back turn
- unconditional pass turn

## Concrete Mismatches

### Mismatch 1: legal take-and-return turns are invalid

Current validator logic blocks any take that would cross 10. That is the main user-reported gap.

### Mismatch 2: legal reserve-plus-gold-overflow turns are invalid

The reserve path suppresses gold at 10 instead of granting gold and forcing cleanup.

### Mismatch 3: action space teaches the wrong game

Because discard and pass are top-level actions, the policy head and any training data are learning a different ruleset than base Splendor.

### Mismatch 4: UI semantics are wrong for overflow cleanup

Today the CLI/UI shows discard as a normal move choice, not as a forced continuation of the current turn.

### Mismatch 5: noble handling is adjacent rules risk

`_give_nobles_if_earned()` at [splendor/SplendorLogicNumba.py:465](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py:465) awards every earned noble in one buy. Public rules summaries commonly describe choosing one noble if multiple are available. This is not the main ticket, but it is a neighboring rules-accuracy risk and should be checked before or during implementation.

## Why This Is Not A Small Patch

### The action contract changes

Any correct full-action design must answer:

- how a state represents "the player is in overflow cleanup"
- how valid moves differ during cleanup vs normal play
- whether cleanup choices consume the turn immediately or require an intermediate state
- whether the action size stays fixed or changes

The current design assumes every turn is represented by exactly one action ID. Overflow cleanup breaks that assumption unless every possible take-plus-return combination is flattened into one larger action space.

### Flat encoding can grow quickly

If the repository keeps the rule "one turn equals one action ID", then token-taking actions would need to encode:

- what was taken
- what was returned if overflow occurred

That expands quickly because the returned tokens are chosen from the player's post-take inventory and may include several color patterns.

Even without exact combinatorics here, the important conclusion is stable:

- the current 30 token-take actions are not enough to represent full take-and-return behavior as one atomic action
- reserve actions may also need embedded return choices

### The neural-network interface is affected

Because the policy head is shaped to the current layout, any action-space redesign affects:

- policy output dimensions
- move masking
- pretrained checkpoint compatibility
- likely self-play and training pipelines

### This repo already has precedent for staged turns

The repository is not universally built around "one real-world turn equals one atomic decision". The README explicitly notes that Small World includes turns that need several actions. That matters because a staged Splendor turn is architecturally plausible within this codebase even if Splendor does not work that way today.

## Design Options

### Option A: keep one-step turns, enumerate every take-plus-return combination

Description:

- Replace token-take actions with a larger flat set that directly encodes both the take and the cleanup return.

Advantages:

- does not require an explicit "pending cleanup" phase in the board state
- keeps MCTS turn transitions simple

Disadvantages:

- action-space growth
- messy move-to-string/UI mapping
- harder symmetry bookkeeping
- reserve actions also need overflow variants
- larger policy change and likely larger retraining cost

Assessment:

- possible, but unattractive

### Option B: introduce staged turn resolution with an overflow-cleanup phase

Description:

- a take or reserve action applies immediately
- if the player is now above 10, the same player remains active in an internal cleanup phase
- valid moves during cleanup are restricted to legal token returns
- only after cleanup does turn control pass to the next player

Advantages:

- models the real rules directly
- avoids combinatorial flat-action growth
- naturally supports reserve-plus-gold overflow
- keeps token-return logic local and explicit

Disadvantages:

- requires new board-state fields or reused board rows for subphase/pending action state
- changes turn/round accounting semantics
- requires the UI and policy mask to respect phase-dependent actions

Assessment:

- best fit for correctness and maintainability

### Option C: hybrid approach with synthetic cleanup-only actions reused across phases

Description:

- keep a fixed action size
- repurpose current give-back actions as cleanup-only actions instead of top-level turn actions
- add phase state so those same action IDs are legal only during overflow cleanup

Advantages:

- avoids a combinatorial new action space
- may limit policy-head disruption if action count can remain 81 or near 81

Disadvantages:

- the existing 20 give-back actions only cover discarding up to 2 tokens, which is not enough for cases like taking 3 at 10 and returning 3
- still needs phase state
- may need extra discard combinations or repeated cleanup micro-actions

Assessment:

- viable only if cleanup is modeled as one-or-more micro-actions inside the same turn, which complicates turn accounting and MCTS semantics

## Recommended Direction

The cleanest target is Option B:

- introduce an explicit overflow-cleanup subphase
- treat reserve gold and take-token actions as provisional state transitions
- keep the active player unchanged until the cleanup obligation is satisfied
- only then advance the round and switch to the next player

Why this is the best fit:

- it matches the board-game rule directly
- it avoids encoding explosion
- it naturally removes standalone discard and pass from normal play
- it scales better if future rule-accuracy work adds related phase decisions

## Likely Implementation Surfaces

The following files are likely to be involved when implementation starts:

- [splendor/SplendorLogicNumba.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogicNumba.py)
  - board state layout
  - action size or phase semantics
  - valid move generation
  - move application
  - reserve and take-token helpers
  - round advancement
- [splendor/SplendorLogic.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorLogic.py)
  - move strings
  - possibly helper lists for discard combinations
- [splendor/SplendorPlayers.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorPlayers.py)
  - CLI display for phase-specific legal actions
- [splendor/SplendorNNet.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorNNet.py)
  - policy head dimensions and meaning
- [splendor/SplendorGame.py](/workspace/.worktrees/full-splendor-actions/splendor/SplendorGame.py)
  - action-size contract remains a central integration point
- training/checkpoint code outside `splendor/`
  - needed if the action size changes or old checkpoints become incompatible

## Questions To Lock Before Implementation

These decisions should be made explicitly before code changes:

1. Should the implementation preserve a fixed action size, or is action-count breakage acceptable?
2. If cleanup is staged, should discarding 3 tokens happen in one cleanup action or several cleanup micro-actions?
3. Is pretrained checkpoint compatibility required, or is retraining expected?
4. Should pass be removed entirely from Splendor, or only disabled outside exceptional no-legal-move states?
5. Should noble selection also be corrected in the same change or deliberately kept out of scope?

## Bottom Line

The current repo supports a simplified Splendor action model, not full base-game token actions.

The most important missing behavior is:

- take or reserve first
- then, if above 10, force same-turn cleanup by returning chosen tokens

That change touches the engine contract, not just token validation. The most defensible implementation direction is to introduce an explicit overflow-cleanup phase rather than trying to flatten every take-plus-return combination into a larger one-shot action space.
