---
simd: 'XXXX'
title: Restrict Transaction Fee Bid Space
authors:
  - Ajayi-Peters Oluwafikunmi (Eclipse)
category: Standard
type: Interface
status: Draft
created: 2024-12-20
feature: (fill in with feature tracking issues once accepted)
supersedes: SIMD-110
extends: SIMD-110
---

## Summary

This proposal outlines a data structure, algorithm and breaking change to the 
JSON RPC API that will allow users to make informed bids for the state they
want to access.

Importantly, the design of the mechanism is such that there is no additional
overhead on voting nodes, and it will always maximize blockspace utilization.

## Motivation

The current implementation of Solana's fee market is a pure first price auction
(FPA) and there is theoretical and empirical evidence to the fact this mechanism
is suboptimal.

Given that Ethereum solved the same problem with EIP-1559, there has been push
to move in that direction on Solana a la SIMD-110. However, salient 
concerns around the proposal have been raised. In addition, SIMD-110 is an 
"invasive" change as it requires changing parts of Solana core.

The most minimal implementation that can achieve the same effects as 
(or better than) EIP-1559 and by extension SIMD-110 without the externalities 
of SIMD-110 is highly desirable.

### Effectiveness as a TFM
Research suggests that that "first-price auctions with an exogenously restricted 
bid space are the only static credible mechanisms". In short, all blockchains should
strive to implement TFMs that model FPAs with restricted bid spaces.

### Minimalism
When working on (especially complex) systems, solutions that affect smaller portions
of code are always favored over their counterparts for simplicity and ease of
implementation and maintenance.

### Concerns with SIMD-110
There are a number of concerns with SIMD-110 that can be found in the discussion
section of the proposal; they will be left out for brevity but the concerns do not
affect the proposed mechanism for reasons discussed below.

## New Terminology

- recommended priority fee: This is the priority fee that the mechanism "recommends" that
a transaction pay. It is calculated per account by the *priority fee recommender* based
on *recent fees paid*. It is similar to but distinct from a per account EIP-1559 base fee.

- recommended high priority fee: This is the priority fee that the mechanism "recommends"
that a transaction seeking faster-than-normal inclusion pay.

- LRUCache or simply cache: Describes an LRUCache data structure that maps the most 
contentious accounts to the corresponding recommended priority and high priority fees.

- recent fees paid: is the median fee paid per account (or globally) in the three (3) 
most recent blocks.

- recent high fees paid: is the **p90** fee paid per account in the three (3) most recent
blocks.

- priority fee recommender: a gadget that determines the `recommended priority fee` to be paid 
per account (or globally) based on `recent fees paid`. The algorithm is 
described in a following section.

- contention: This describes how contentious an account is, for the purpose of this proposal,
contention is simply calculated as the number of transactions writing to an account.

- `getPriorityFee`: A JSON RPC method to replace the `getRecentPrioritizationFees` or 
`getFeeForMessage` method. The `getPriorityFee` request takes a mandatory list of
transactions and returns the recommended fee to land a transaction locking all the accounts 
in the list.

- `getHighPriorityFee:` A new JSON RPC method. It takes a mandatory list of accounts and returns
a recommended priority fee to quickly land a transaction locking all the accounts in the list

## Detailed Design

At the core of this proposal is an LRUCache that maps accounts to the recommended priority fee 
(calculation discussed below). The fee is described as a recommendation because rather than
enforcing that transactions pay at least this fee to be included in a block ala EIP-1559 or
SIMD-110, the fee is merely supposed to guide users on how much to bid.

Functionally, the recommendation is the same as the base fee or write-lock fee in EIP-1559 and SIMD-110 
respectively, in that (given that fee markets are deterministic), it is a good reference point for users
vying for inclusion. But because this fee is not enforced in protocol, it also has the benefit of 
allowing blocks to be maximally packed by including transactions that don't bid as high as the recommendation.
This property is important as it theoretically improves the accuracy of the recommendations made by the mechanism.

Additionally, the fact that the fees are recommendations and not protocol-enforced means validator nodes
don't need to maintain this cache; only RPC nodes which are responsible for responding to these types
of requests.

### The LRUCache

The cache is an in-memory LRU cache maintained by RPC nodes. The cache maps accounts to a `fee_data`
struct.
Let the most recent block seen by the RPC node be `b_n`,

  the `fee_data` struct contains the recommended fee `f_r` for `b_n + 1` and the median and p90 fees
  for the corresponding account in the last three blocks (`b_n`, `b_n-1`, `b_n-2`).

### RPC Processing Requests

When an RPC node receives a `getPriorityFee` request, based on the request it:
- checks the cache for all the accounts in the accounts list attached.
- if none of the accounts in the list are in the cache then it simply returns the recommended
global priority fee.
- if one or more of the accounts in the list are in the cache, then it returns the recommended
per-account priority or high priority fee for the most contentious account (should also 
be the most expensive).

### RPC Processing Blocks

When an RPC node receives `b_n`, it:
- calculates the global median.
- calculates the per-account median for transactions in the block
- updates the cache with recommendations for `b_n+1`

#### Cache Eviction Policy

- Accounts with their most recent entry older than three blocks i.e., the most recent entry is from a block
`b_i` where i > 3, are evicted from the cache. This is done because an account that does not have any
transactions in the last three blocks that suggests one of two things:

  1. The account is no longer contentious and all future accesses should be priced based on global
  fees
  2. The global median fee has risen above the recommended fee for that account in whcih case, users
  should pay the recommended median fee.

- If the recommended fee for an account in the cache is less than the global median, the account is
evicted from the cache.

#### Cache updates
- accounts with both median and  p90 values higher than the global median are added to the cache or updated.
- accounts with stale data are evicted.

### Priority Fee Recommender

The priority fee recommender is a gadget that takes the priority fee (median or p90) for the last
three blocks and contention and returns the recommended priority or (high priority fee) for the 
upcoming block.

The gadget looks at the previous data and based on the trends makes recommendations.
The exact mathematical relation is TBD but a rough idea is that:

- if the fee paid is increasing block-on-block, recommend a higher fee than the
previous block,
- if the fee paid is reducing, and contention is increasing, recommend a lower fee,
- if the fee paid is neither increasing nor decreasing, recommend the average of the last three
blocks.

Given that transaction arrival rate can be closely modelled by a Poisson's distribution, it's likely
that an exponential controller will be the best choice but a linear controller will be the first.

## Alternatives Considered

What alternative designs were considered and what pros/cons does this feature
have relative to them?

1. EIP-1559 like system with strict base fee settings
The primary alternative consideration is a per account EIP-1559 implementation which could be
SIMD-110 or any other similar mechanism. The proposed mechanism is less tasking on validator nodes
than any protocol-enforced implementation and should be more efficient since it considers additional
data.

2. Burning the base fee completely.
In addition to the feature above, burning the per signature base fee was also considered but the benefits
of such a mechanism are debatable. Although, there are no known drawbacks.

3. Excluding high priority fees.
Excluding high priority fees and all associated data and methods was considered but it gives users and
developers more expressiveness without running into the same problems that the mechanism originally 
intended to tackle--an unbounded bid space.

4. Doing Nothing.
Users will continue to overbid and the UX will be subpar.

5. Blocking transactions that contain a fee less than the median.
It's possible to implement the mechanism such that RPC nodes drop all transactions with fees less than the recommended 
global fee. This has the same effect as having an in-protocol base fee. This was left out because there's not much 
evidence that suggests that users will send low-fee transactions in hopes of landing if fee markets are deterministic. 
Only an attacker would. However, should the need ever arise to add this feature, it should be easy to add.

6. Having a max cap on the cache size.
At the moment, the cache size is unbounded. Judging from the number of contentious accounts on mainnet-beta, it's 
unlikely that the cache will be very large but in the future it might be necessary to sacrifice some efficiency by setting 
a limit on the cache size. In such a scenario, when the cache is updated at the end of every slot, the bottom X accounts 
can be replaced by the most contentious accounts in the new block.

7. Using contention to scale the recommendations.
In the current calculation for recommended fee contention is not explicitly considered, it can be useful as more 
contentious accounts should have prices raised more aggressively.

## Impact

- validators: unaffected
- core contributors: unaffected
- RPC Nodes: They improve transaction landing probability for users.
- users: Users can make better-informed bids.
- dapp developers: they will have to switch to the new method and conform to it's format

## Security Considerations

None

## Drawbacks

While it only affects non-critical parts of the codebase, this is still a complex change to the TFM.

## Backwards Compatibility

For the sake of adoption, this proposal favors deprecating other fee request RPC methods so
developers use this one and bids are standardized. That means the implementation might not be
backwards compatible.
