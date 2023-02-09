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

<img width="680" alt="Screen Shot 2023-02-09 at 3 36 18 PM" src="https://user-images.githubusercontent.com/24661810/217931229-2d79c7b2-4e2d-4a8e-89f5-0fe9be440923.png">
In this case, the builder knows that their submission needs to reach the relay before $t=12-Δ$




## Optimistic relay: potential roadmap

## Security considerations

## Learnings from Goerli

## FAQ

