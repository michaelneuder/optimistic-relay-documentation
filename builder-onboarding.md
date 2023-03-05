# Optimistic relaying—builder guide

Thank you for your interest in low-latency optimistic relaying with [the ultra sound relay](https://relay.ultrasound.money)! This document is an onboarding guide for you, Ethereum block builder. Please take the time to understand it :)

**TLDR**

>1. Share with us over Telegram or Discord the list of builder pubkeys you want promoted for optimistic relaying. We will manually review recent bid submissions from those pubkeys to ensure a low historical rate of bad bids. A bad bid is one with an invalid block or an insufficient payment to the proposer.
>2. Post a maximum of 1 ETH collateral to `relay.ultrasound.eth` and share the transaction details with us. The transaction sender must be an address publicly associated with one of your builder pubkeys, ideally your primary fee recipient address.
>3. The relay will automatically demote you for submitting a single bad bid to the relay. You will only be re-promoted after the underlying reason for submitting a bad bid is addressed.
>4. A bad bid winning the auction will cause an on-chain incident, i.e. a missed slot or an insufficient proposer payment. We expect you to directly compensate the proposer the bid value plus a fixed 0.01 ETH penalty within 24 hours and send us the transaction details.
>5. If we do not receive proof that the proposer was compensated within 24 hours we may use your collateral to compensate the proposer ourselves.

### Purpose

This document outlines key aspects of optimistic relaying with the ultra sound relay. Optimistic relaying allows builders to reduce the latency of their bid submissions through asynchronous simulation. For more detail, see [the proposal](https://github.com/michaelneuder/opt-relay-docs/blob/main/proposal.md), [the implementation](https://github.com/flashbots/mev-boost-relay/pull/285), and the discussion in [MEV community call #0](https://collective.flashbots.net/t/mev-boost-community-call-0-23-feb-2023/1348).

### Optimistic logic

The optimistic relay implementation adds three DB fields for every builder pubkey:

1. `is_optimistic`: This boolean, which defaults to `false`, indicates whether or not the pubkey is eligible for optimistic relaying. Promoting a builder pubkey for optimistic relaying is a manual process. When a builder submits a bad bid `is_optimistic` is reset to `false` before moving to the next slot, with demotion details recorded in the relay DB.
2. `collateral`: This integer reflects collateral value in wei backstopping the value of optimistic bids. That is, optimistic relaying happens when `is_optimistic` is `true` and `collateral` is at least as large as the bid value.
3. `builder_id`: This string is used to share collateral across multiple pubkeys. The demotion of a pubkey will result in the simultaneous demotion of all pubkeys sharing the same builder ID.

Consider the example below:

```
 builder_pubkey | is_optimistic | collateral         | builder_id
----------------+---------------+--------------------+------------------
 0xaaaaaa...    | true          | 990000000000000000 | mike
 0xbbbbbb...    | true          | 990000000000000000 | mike
 0xcccccc...    | false         | 990000000000000000 | flashbots
 0xdddddd...    | false         |                  0 | bloxroute
```

Pubkeys `0xaaaaaa` and `0xbbbbbb` share the same builder ID `mike` and collateral of 0.99 ETH. (0.99 ETH is the maximum 1 ETH collateral minus 0.01 ETH for the fixed penalty.) Since `is_optimistic` is `true`, any bid with a value less than or equal to 0.99 ETH will be relayed optimistically. A large bid with value 10 ETH will not be relayed optimistically. If either pubkey submits an invalid bid both pubkeys will be demoted before the next slot.

Pubkey `0xcccccc` also has 0.99 ETH of collateral but `is_optimistic` is `false` so their bids will not be relayed optimistically. Builder `0xdddddd` has no collateral so their bids will also not be relayed optimistically.

### Collateral

Collateral for optimistic relaying must be posted to `relay.ultrasound.eth` from an address publicly associated with one of your builder pubkeys, ideally your primary fee recipient address. The maximum collateral per pubkey is currently 1 ETH—this value may be increased or decreased from time to time. Please contact us if you wish to stop optimistic relaying and have your collateral returned.

### Builder ID

For collateral efficiency you may reuse the same piece of collateral for multiple pubkeys. Let us know which pubkeys you want to share a builder ID. The demotion of a pubkey will result in the demotion of all pubkeys sharing the same builder ID.

### Promotions and demotions

We will manually promote your pubkeys by setting `is_optimistic` to `true` after collateral is posted and you have indicated readiness for optimistic relaying. For transparency, we intend to publicly disclose `is_optimistic`, `collateral`, `builder_id` for every pubkey.

If a bad bid is submitted, even if the bid does not win the auction, the demotion logic resets `is_optimistic` to `false` before the next slot. Only after the root cause of a demotion is understood and fixed can we manually reset `is_optimistic` to `true`.

### On-chain incidents

An on-chain incident happens when an invalid bid wins the auction and the proposer needs to be compensated for a missed slot or an insufficient payment. Proposers should be made whole of the full bid value plus 0.01 ETH. The fixed 0.01 ETH penalty attempts to cover missed consensus rewards as well as accounting hassle from the delayed payment.

Note that we expect you, within 24 hours of the demotion, to directly send ETH to the proposer's fee recipient to compensate for a bad bid. Please share with us details of the corresponding transaction. Without proof the proposer was compensated we may use your collateral to compensate the proposer ourselves. For transparency, every on-chain incident will trigger a public post-mortem with details of the bad bid.
