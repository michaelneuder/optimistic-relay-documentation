# Optimistic Relay Proposal

## Purpose

This document introduces the concept of the Optimistic Relay to accompany https://github.com/flashbots/mev-boost-relay/pull/279, which is a small 
PR which adds the functionality to Flashbots' `mev-boost-relay`. While the PR gets into the details of *how* the optimistic feature is added,
this document aims at motivating the change in the broader context of `mev-boost` and the existing Ethereum literature on Proposer-Builder Separation (PBS).

## mev-boost today

`mev-boost` has become critical infrastructure since it was introduced by Flashbots. There are a number of excellent data sources to demostrate this. 
The three below are my favorites:

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

The difference between `mev-boost` and in-protocol PBS (IP-PBS) is the presence of the relay. The relay is a trusted intermediary between the block builders and 
the proposers, while in IP-PBS other validators enforce the rules of the block building auction through attestations. 

## Optimistic relay: potential roadmap

## Security considerations

## Learnings from Goerli

## FAQ

