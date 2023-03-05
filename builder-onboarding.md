# Optimistic relaying—builder guide

Thank you for your interest in low-latency optimistic relaying with [the ultra sound relay](https://relay.ultrasound.money)! This document is an onboarding guide for you, Ethereum block builder. Please take the time to understand it :)

**TLDR**

>1. Share with us over Telegram or Discord the list of builder pubkeys you want promoted for optimistic relaying. We will manually review recent bid submissions from those pubkeys to ensure a low historical rate of bad bids. A bad bid is one with an invalid block or an insufficient payment to the proposer.
>2. Post a maximum of 1 ETH collateral to `relay.ultrasound.eth` and share the transaction details with us. The transaction sender must be an address publicly associated with one of your builder pubkeys, ideally your primary fee recipient address.
>3. The relay will automatically demote you for submitting a single bad bid to the relay. You will only be re-promoted after the underlying reason for submitting a bad bid is addressed.
>4. A bad bid winning the auction will cause an on-chain incident, i.e. a missed slot or an insufficient proposer payment. We expect you to directly compensate the proposer the bid value plus a fixed 0.01 ETH penalty within 24 hours and send us the transaction details.
>5. If a proposer is not compensated within 24 hours we may use your collateral to compensate the proposer ourselves.

### Purpose

This document outlines key aspects of optimistic relaying with the ultra sound relay. Optimistic relaying allows builders to reduce the latency of their bid submissions through asynchronous simulation. For more detail, see [the proposal](https://github.com/michaelneuder/opt-relay-docs/blob/main/proposal.md), [the implementation](https://github.com/flashbots/mev-boost-relay/pull/285), and the discussion in [MEV community call #0](https://collective.flashbots.net/t/mev-boost-community-call-0-23-feb-2023/1348).

### Optimistic logic

The optimistic relay implementation add three DB fields for every builder pubkey:

1. `is_optimistic`: This boolean, which defaults to `false`, indicates whether or not the pubkey is eligible for optimistic relaying. Promoting a builder pubkey for optimistic relaying is a manual process. When a builder submits a bad bid `is_optimistic` is set to `false` before moving to the next slot, with demotion details recorded in the relay DB.
2. `collateral`: This integer reflects the collateral value in wei protecting proposers against bad bids. Optimistic relaying happens when `is_optimistic` is `true` and `collateral` is at least as large as the bid value.
3. `builder_id`: This string is used to share collateral across multiple pubkeys. A demotion from one pubkey will result in all pubkeys sharing the same builder ID being demoted simultaneously.

Consider the example below:

```
 builder_pubkey | is_optimistic | collateral         | builder_id
----------------+---------------+--------------------+------------------
 0xaaaaaa...    | true          | 990000000000000000 | mike
 0xbbbbbb...    | true          | 990000000000000000 | mike
 0xcccccc...    | false         | 990000000000000000 | flashbots
 0xdddddd...    | false         |                  0 | bloxroute
```

Pubkeys `0xaaaaaa` and `0xbbbbbb` share the same builder ID `mike` and collateral of 0.99 ETH. (0.99 ETH is the maximum 1 ETH collateral minus 0.01 ETH for the fixed penalty.) Since `is_optimistic` is `true`, any bid with a value of less than or equal to 0.99 ETH will be relayed optimistically. If either pubkey submits an invalid bid both pubkeys will be demoted before the next slot.

Pubkey `0xcccccc` also has 0.99 ETH of collateral but `is_optimistic` is `false` so their bids will not be relayed optimistically. Builder `0xdddddd` has no collateral so their bids will also not be relayed optimistically.

### Collateral

Collateral for optimistic relaying must be posted to `relay.ultrasound.eth` from an address publicly associated with one of your builder pubkeys, ideally your primary fee recipient address. The maximum collateral per pubkey is currently 1 ETH—this value may be increased or decreased from time to time.

### Builder ID

For collateral efficiency you may reuse the same piece of collateral for multiple pubkeys. Let us know which pubkeys you want to share a builder ID. A demotion for one pubkey will result in the demotion of all pubkeys sharing a builder ID.

### Promotions and demotions

We will manually promote your pubkeys by setting `is_optimistic` to `true` after collateral is posted and you have indicated you are ready. If a bad bid is submitted, even if the bid does not win the auction, the demotion logic resets `is_optimistic` to `false` before the next slot. Only after the root cause of a demotion is understood and fixed can we manually reset `is_optimistic` to `true`.

### On-chain incidents

An on-chain incident happens when an invalid bid wins the auction and the proposer needs to be compensated for a missed slot or an insufficient payment. Please note that we expect you to directly compensate the proposer for a bad bid within 24 hours of the demotion. If not, we may use your collateral to compensate the proposer ourselves. We will publicly publish on [TBD](...) a post-mortem for each on-chain incident with details of the bad bid.

<!-- public API for them to check builder status? dashboard on USR? -->
