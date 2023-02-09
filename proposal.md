# Optimistic Relay Proposal

## Purpose

This document introduces the concept of the Optimistic Relay to accompany https://github.com/flashbots/mev-boost-relay/pull/279, which is a small 
PR which adds the functionality to Flashbots' `mev-boost-relay`. While the PR gets into the details of *how* the optimistic feature is added,
this document aims at motivating the change in the broader context of `mev-boost` and the existing Ethereum literature on Proposer-Builder Separation (PBS).

## mev-boost today

`mev-boost` has become critical infrastructure since it was introduced by Flashbots. There are a number of excellent data sources to demostrate this. 
These three are my favorites:

- https://www.mevboost.org/
- https://mevboost.pics/
- https://transparency.flashbots.net/

tl;dr; >90% of validators are using `mev-boost` to outsource block building. 

The Flashbots' team has continued to engage with the community around the future of software:

- https://collective.flashbots.net/t/toward-an-open-research-and-development-process-for-mev-boost/464
- https://collective.flashbots.net/t/mev-boost-development-philosophy/505
- https://collective.flashbots.net/t/development-next-steps-for-pbs-roundtable-at-devcon/438

Part of that discussion is around how the community can continue iterating on the initial architecture to build understanding around what could/should 
be enshrined in the protocol. 

## Proposer-Builder Separation (PBS)

PBS has been extensively researched. See https://notes.ethereum.org/@domothy/pbs_links for a good starting set of links for the literature and 
https://barnabe.substack.com/p/pbs for a thorough overview of the current research landscape. 

## Optimistic Relay: phase 1

The Optimistic Relay is an idea from Justin Drake to help bridge the gap between the research and the current `mev-boost` software, with the 
goal of building understanding about how the mechanisms proposed in the PBS literature behave in practice. 
This aims to be a step towards understanding what (if anything) should be enshrined in the protocol.

The primary difference between `mev-boost` and in-protocol PBS (IP-PBS) is the presence of the relay. The relay is a trusted intermediary between the block builders and 
the proposers, while in IP-PBS other validators enforce the rules of the block building auction through attestations. For this reason, phase 1 of the Optimistic Relay and
more generally the roadmap described below gradually reduce the role of the relay in block production.  

#### Asynchronous block validation
The key idea in phase 1 is to change the builder block submission flow to make the block validation an asnychronous process. When the builder submits a block 
to the relay, that bid is immediately elligible to win the auction, even if the relay hasn't had a chance to check that the block is valid. This allows builders 
to submit more blocks, and importantly submit blocks later in the slot (because they don't have to wait for the block to be validated for the bid to register). 
We call it "optimistic" because the relay assumes that the block is valid in the short-term, while deferring the actual validation the the subsequent slot. A 
builder must post collateral with the relay to take advantage of this optimistic processing of their blocks, and if they submit an invalid block they are "demoted" 
back to the current `mev-boost` implementation where each block is validated before the bid is considered elligible. Critically, if a builder submits an 
invalid block that ends up winning the auction and being proposed, that builder's collateral is slashed by the winning amount, and the proposer who missed their 
slot is refunded the bid. 

#### A builder's perspective 

The sequence diagram below shows the block building pipeline of regular `mev-boost`:
<img width="797" alt="Screen Shot 2023-02-09 at 2 41 14 PM" src="https://user-images.githubusercontent.com/24661810/217920211-1730f598-4542-495f-b910-b8ca87d77382.png">

The builders block must be validated before the proposer calls `getHeader` on the relay at the beginning of the slot. In this diagram, Δ denotes the amount
of time that it takes the relay to validate the block. Thus the builders submission must arrive at least Δ before the start of the next slot for the bid to be
valid. Compare this to the optimistic block building pipeline:

<img width="797" alt="Screen Shot 2023-02-09 at 2 53 41 PM" src="https://user-images.githubusercontent.com/24661810/217922756-6abe71fd-a7e6-49fa-850d-3741488f937e.png">

Here the builder immediately gets a response indicating that their bid is active and the relay might not simulate the block until the payload has already been 
published as a full beacon block. A builder is incentivized to use optimistic relaying because they no longer have to pay the Δ time penalty during their 
block submission. MEV is a game of small margins, and speed is critical for successful builders. To demonstrate that consider the following situation:  

<img width="797" alt="Screen Shot 2023-02-09 at 3 36 18 PM" src="https://user-images.githubusercontent.com/24661810/217931229-2d79c7b2-4e2d-4a8e-89f5-0fe9be440923.png">
In this case, the builder knows that their submission needs to reach the relay before time 12-Δ. Right after they submit their block, a large MEV opportunity arises.
They can try to submit a new block that captures more MEV and thus has a higher probability of winning the auction, but it is too late as 
the simulation will not complete before t=12. 

Intuitively, the longer the builder can wait before submitting their block, the longer they have to listen for transactions and thus the higher their bid.
We see this in the data collected on the `ultra-sound relay`. The figure below shows the time that the winning block arrived to the relay for a number of slots:

<img width="797" alt="Screen Shot 2023-02-09 at 3 36 18 PM" src="https://user-images.githubusercontent.com/24661810/217935966-8985f4cc-0388-4983-893c-61e602f3a66b.jpeg">


The following plot shows the amount of time it takes to simulate a block.
<img width="797" alt="Screen Shot 2023-02-09 at 3 36 18 PM" src="https://user-images.githubusercontent.com/24661810/217936342-aaac6105-8e65-4257-95db-e13f4a6b5d3c.jpeg">

Generally around ~200ms, where these milliseconds at the end of the slot are incredibly valuable to the builder. 

#### A proposer's perspective 
The proposers perspective is almost identical to the status quo of `mev-boost`, with the slight change that they no longer have a guarantee that the header 
they sign corresponds to a valid block. They still call `getHeader` at the beginning of the slot and sign the header they receive. 
The trust assumptions on the relay remain the same. If the block they end up proposing turns out to be invalid 
they must trust the relay to refund them for their lost slot, which is the same as trusting the relay to validate the block includes a payment corresponding to 
the bid amount in the base `mev-boost` case. 

## Security considerations

There were a number of concerns raised in https://github.com/michaelneuder/mev-boost-relay/pull/2 that we would like to address. Thanks to Chris Hager, Alex Stokes, and Mateusz Morusiewicz for this initial feedback.

#### Missed slots (liveness-attack)
This proposal increases the risk of missed slots, but only marginally. Because the relay doesn't validate the block, the proposer could end up signing a bad header. The proposer is 
unable to rectify the situation by signing a valid block, because signing two blocks of the same height is a slashing condition. However, we implemented it so that there will be 
at most one missed slot per-optimistc builder before a manual intervention. By default, all builders are treated non-optimistically and the process of posting collateral is manual. 
If that builder ends up submitting a bad block, a single slot will be missed and we will manually investigate the failure, initiate the refund, and 
communicate with the builder. Only once we have high confidence that we understand what went wrong and why it won't happen again will we allow the builder to re-post collateral and have access 
to optimistic processing once again. Additionally, we only optimistically process blocks that have a value less than the collateral posted for a builder. The builder is free to submit blocks with huge MEV that exceeds their collateral, we just will validate those bids synchronously because they can't cover refund amount. Lastly, have the benefit of rolling this change out slowly. By starting with just the `ultra-sound relay` we can continually monitor the number 
of missed slots caused by this change. If we ever determine that it exceeds what we are comfortable with, we can simply turn it off. 

#### Collusion 
Consider the case where the proposer and the builder are the same malicious actor. The builder could submit a large bid with an invalid block in order to ensure that they win the auction through the relay. This will result in an invalid block being proposed and the relay accounting system recording that the proposer is owed a large refund.
However, the bid was collateralized by that same builder, so even though we issue a refund, it is the same actor that is being slashed, so the money is just going from the malicious actor to themselves. The only downside is that the relay team had to manually issue a refund, but we anticipate this to be a rare occurance, and 
by only onboarding trusted builders, we minimize this toil. Additionally, the proposer didn't accomplish anything other than skipping their slot, which they could
do by just turning off their machine in the first place. 

#### Incentive compatiblity

The builder is incentivized to use the optimistic relay honestly because it strictly increases the amount of MEV they are able to produce. Submitting an invalid 
bid results in them losing their collateral, which is not a rational choice. 

The proposer is incentivized to use the optimistic relay because it increases the size of the bids against their slot. Larger bids means more value extracted 
for the proposer themselves. 

The relay remain a neutral party that serves as an escrow mechanism, but does not receive any rewards.

#### Moral hazard 
The more subtle argument is that this change introduces a moral hazard. While we acknowledge that there is a chance a few extra missed slots, we plan on approaching
this change conservatively. By only allow-listing trusted builders initially, we ensure that there will be no runaway amount of missed slots (again only one 
per-builder). We will actively collect data around any invalid blocks proposed and work with builders to understand what caused invalid block submissions. The builders
are highly incentivized to avoid being slashed and thus will want to know what is going wrong with their blocks if something fails. 

## Learnings from Goerli

TODO(mikeneuder): add when we have learned something :-)

## Optimistic relay: potential roadmap

TODO(mikeneuder)

## FAQ by Justin Drake

**Does an optimistic relay simulate blocks?**

_Yes, an optimistic relay simulates all blocks that could have been forwarded to a proposer. Simulation just happens away from the latency-critical path._

**Can optimistic relaying lead to mass missed slots?**

_Assuming no relay bug, the worst case is one missed slot per collateralised builder (prior to reactivation). The beacon chain is designed to handle missed slots._

**Are builders incentivised to produce invalid blocks?**

_The builder of an invalid winning block suffers a financial loss. Moreover, a single invalid block will disable optimistic relaying for the builder, yielding a latency disadvantage pending reactivation._

**What recommendations do you have for optimistic relay operators?**

_We have several recommendations for optimistic relay operators:_

  * _alerts – setup automatic alerts (e.g. email or phone call) for demotions_
  * _refunds – promptly transfer (e.g. within 24 hours) the bid value plus beacon chain penalties and missed rewards to the proposer fee recipient_
  * _max collateral – cap the maximum collateral amount per builder (e.g. to 10 ETH) to keep a level playing field_
  * _investigation – manually investigate demotions and ask builders to fix block building bugs before reactivating optimistic relaying_
  * _cool-off – impose a post-demotion cool-off period (e.g. 24 hours) before reactivating optimistic relaying_
  * _penalty – consider a fixed penalty (e.g. 0.1 ETH) per demotion, especially for repeat demotion_

