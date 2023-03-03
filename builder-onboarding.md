# Optimistic relaying — builder onboarding

### Purpose
Outline the steps needed for builders to onboard to optimistic relaying on the
[Ultra Sound Relay](https://relay.ultrasound.money/). Optimistic relaying allows
builders to reduce the latency of their block submissions by asynchronously 
validating their blocks. This gives builders more time at the end of the slot to
get their last bids in to try and win the auction. For further context, see the [proposal](https://github.com/michaelneuder/opt-relay-docs/blob/main/proposal.md) and [implementation](https://github.com/flashbots/mev-boost-relay/pull/285); it 
was also discussed in the [MEV community call #0](https://collective.flashbots.net/t/mev-boost-community-call-0-23-feb-2023/1348).

### Concepts 
The implementation introduces 3 new concepts:

1. __Demotion status__ — Each builder pubkey has a new property called `is_demoted`, which indicates
whether or not the pubkey is eligible for optimistic block handling. All builders
are initially demoted and it is a manual process to be promoted to optimistic mode. 
2. __Demotions__ — Demotions occur when a builder submits a block that is optimistically processed, 
but ends up resulting in a simulation error. Once this happens, the relay marks the builder pubkey
as demoted, and beginning in the next slot, their blocks are handled pessimistically. Details about
the demotion are recorded in the DB.
3. __Collateral__ — To disincentize builders from submitting invalid blocks, collateral must be posted 
to ensure a proposer who misses a slot as the result of a bad block is refunded. This collateral
is controlled by the relay operators. 
4. __Collateral IDs__ — To allow builders to share collateral across many pubkeys, we allow
collateral IDs to uniquely identify the collateral associated with each key. However, a demotion
results in all pubkeys that share a collateral id to be demoted simultaneously. 

### Optimistic blocks
Two factors determine if a given block is processed optimistically:

1. the builder's demotion status, and
2. the value of the collateral associated with the pubkey.

A block that has value less-than or equal to `collateral_value` (units in Wei) from a builder
who is *not* demoted will be handled optimistically. Consider the example below:

```
 builder_pubkey | is_demoted |   collateral_value   |  collateral_id   
----------------+------------+----------------------+------------------
 0xacea6a...    | false      | 1000000000000000000  | mikes-collateral
 0xb67a51...    | true       | 1000000000000000000  | flashbots
 0xaa1488...    | true       |                   0  | bloxroute
```
Builder `0xacea6a` is not demoted and has a collateral value of 1 ETH, so any block
that they submit that has a value of less than 1 ETH will be optimistic. Builder `0xb67a51` 
also has 1 ETH of collateral, but because `is_demoted=t`, their blocks will all be processed
normally. Builder `0xaa1488` has no collateral, so their blocks will be processed normally.

Once the collateral is updated in the database and the builder indicates that they are ready,
we will manually change the `is_demoted` to `false`. At any point if a demotion occurs, `is_demoted` will be set back to `true`, and
a follow up analysis will be conducted to determine the cause of the block simulation failure.
Once the error is understood and there is a consensus that it is safe to resume optimistic building, 
we will manually reset `is_demoted` to `false`.

### Handling missed slots

### Handling other block simulation failures 

### Posting collateral

### Using collateral IDs

### Block simulation success criterion

### Acknowledgement of risks
