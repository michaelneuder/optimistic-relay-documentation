# Optimistic relaying — builder onboarding

Thanks for considering optimistic relaying with the ultra sound relay :-) This 
document outlines the process of onboarding a builder to the system.

**tl;dr;**

>1. Post a max of 1 ETH collateral to `relay.ultrasound.eth`. Send us the transaction details
over Telegram or Discord along with the associated builder pubkeys. The transaction must come from
an address associated with one of the pubkeys. 
>2. Reach out to us when a builder pubkey is ready to be activated. This will be a manual step where
we look at recent block submissions to ensure a low historical simulation error rate on a per-pubkey basis.
>3. In the case of an insufficient proposer payment or a missed slot caused by optimistic relaying, we will reach out to the builder with the details and the expectation
is that the builder directly refunds the proposer. The size of the refund is the bid value + 0.01 ETH to compensate for the missing consensus rewards. If the refund isn't done in 24 hours,
we will use the collateral to repay the proposer.
>4. After any demotion for which the builder is at fault, we will send the error details to the builder and engage with a discussion on
what could have caused the error. Once we agree that the issue is addressed, we will reactivate 
optimistic relaying for that builder.
>5. We ask builders to acknowledge that they have seen this document and understand the requirements. 

### Purpose
Outline the steps needed for builders to onboard to optimistic relaying on the
[ultra sound relay](https://relay.ultrasound.money/). Optimistic relaying allows
builders to reduce the latency of their block submissions by asynchronously 
validating their blocks. For further context, see the [proposal](https://github.com/michaelneuder/opt-relay-docs/blob/main/proposal.md) and [implementation](https://github.com/flashbots/mev-boost-relay/pull/285); it 
was also discussed in the [MEV community call #0](https://collective.flashbots.net/t/mev-boost-community-call-0-23-feb-2023/1348).

### Concepts 
The implementation introduces 4 new concepts:

1. __Demotion status__ — Each builder pubkey has a new property called `is_demoted`, which indicates
whether or not the pubkey is eligible for optimistic block handling. All builders
are initially demoted and it is a manual process to be promoted to optimistic mode. 
2. __Demotions__ — Demotions occur when a builder submits a block that is optimistically processed, 
but ends up resulting in a simulation error. Once this happens, the relay marks the builder pubkey
as demoted, and beginning in the next slot, their blocks are handled pessimistically. Details about
the demotion are recorded in the DB.
3. __Collateral__ — To disincentize builders from submitting invalid blocks, collateral must be posted 
to ensure a proposer who misses a slot or isn't fully paid is refunded. This collateral
is controlled by the relay operators, but will only be used to issue a refund if a builder 
doesn't handle the refund directly within 24 hours of the demotion.
4. __Builder IDs__ — To enable builders to share collateral across many pubkeys, we allow
builder IDs to uniquely identify the collateral associated with each key. However, a demotion
results in all pubkeys that share a builder ID being demoted simultaneously. 

### Optimistic blocks
Two factors determine if a given block is processed optimistically:

1. the pubkey's demotion status, and
2. the value of the collateral associated with the pubkey.

A block that has value less-than or equal to `collateral_value` (units in Wei) from a builder
who is *not* demoted will be handled optimistically. Consider the example below:

```
 builder_pubkey | is_demoted |   collateral_value   |  builder_id   
----------------+------------+----------------------+------------------
 0xacea6a...    | false      | 1000000000000000000  | mikes-collateral
 0xa841ce...    | false      | 1000000000000000000  | mikes-collateral
 0xb67a51...    | true       | 1000000000000000000  | flashbots
 0xaa1488...    | true       |                   0  | bloxroute
```
Pubkeys `0xacea6a` and `0xa841ce` are not demoted and share a collateral value of 1 ETH with the `builder_id=mikes-collateral`, so any block
that they submit that has a value of less than 1 ETH will be optimistic. If either pubkey submits an invalid block, they will both be demoted. Builder `0xb67a51` 
also has 1 ETH of collateral, but because `is_demoted=true`, their blocks will all be processed
pessimistically. Builder `0xaa1488` has no collateral, so their blocks will also be processed pessimistically.

Once the collateral is updated in the database and the builder indicates that they are ready,
we will manually change the `is_demoted` to `false`. At any point if a demotion occurs, `is_demoted` will be set back to `true`, and
a follow up analysis will be conducted to determine the cause of the block simulation failure.
Once the error is understood and there is a consensus that it is safe to resume optimistic building, 
we will manually reset `is_demoted` to `false`.

> Any block simulation failure will result in a demotion that records the details of the
failed simulations, even if that failure didn't result in a missed slot.

### Posting collateral
The first step in onboarding is posting collateral. To start, we are capping this 
collateral at 1 ETH per-pubkey. Collateral can be sent to `relay.ultrasound.eth`. 
Please send us the transaction details on Telegram or Discord and we will manually update
the database. 

### Using builders IDs
If a builder wishes to use the same collateral for multiple pubkeys, please let us
know on Telegram or Discord and we will assign a builder ID to all the relevant pubkeys.
Again, the result of using builder IDs is that any demotion for one of these pubkeys 
results in all of them being demoted.

### Handling missed slots
Missed slots occur when 

1. an optimistic builder submits an invalid block, and
2. that block wins the auction and the header is signed by the proposer.

Once this occurs, the invalid block will be be rejected by the network and the proposer
has no recourse, because signing a different header for that same slot is a slashing condition.
This is the worst-case scenario for optimistic relaying. When a slot is missed we will publish
a post-mortem that identifies

1. the builder and proposer involved,
2. the missing slot number,
3. a timeline of events including the timestamps leading up to the invalid header being signed,
4. the details of the simulation error (including the invalid payload and the resulting error message), and 
5. an analysis of what caused the error and a solution.

We will keep a public log of these missed slots and engage with the community about 
the frequency and overall network impact of optimistic relaying. 

> When a slot is missed or there is an insufficient proposer payment, the proposer needs to be refunded based on the value of the 
winning bid and an additional 0.01 ETH for the missing consensus layer rewards. The relay operators will reach out to the builder with the details of the
error and the size of the refund, and the expectation is that the builder directly refunds the proposer. Thus 
the collateral held by the relay remains untouched. If after 24 hours, the builder 
has not responded and refunded the proposer, the builder collateral will be used 
to execute the refund. We hope that in the vast majority of cases, the builder 
handles the refund so that there is no need to repost collateral when they want
to activate optimistic building again. 

### Handling other block simulation failures 
Any block simulation error for an optimistic builder will result in a demotion that
requires manual intervention. Even if the invalid block does not win the auction, we 
want to examine the error before manually reactivating optimistic relaying for that builder. 
Unlike missed slots, we don't plan on posting a post-mortem for each simulation error that results in a demotion, but we still
want to understand what went wrong and have high confidence that it won't occur again.

### Acknowledgement of risks and requirements

Thanks again for considering optimistic relaying! The last thing we ask is that 
each builder acknowledges through Telegram or Discord that they have seen this document and understand the risks
and requirements of this experiment. 

<!-- public API for them to check builder status? dashboard on USR? -->