---
simd: 'XXXX'
title: Restrict Transaction Fee Market Bid Space
authors:
 - Ajayi-Peters Oluwafikunmi (Eclipse)
category: Standard
type: Interface
status: Draft
created: 2024-12-20
feature: (fill in with feature tracking issues once accepted)
supersedes:
extends:
---

## Summary

This proposal makes breaking changes to the RPC API, and introduces a target 
utilization amount to Solana's TFM to improve fee paying and inclusion UX.
Importantly, the design of the mechanism is such that there is no additional
overhead on voting nodes, and it will always maximize blockspace utilization.

## Motivation

The vast majority of Solana transactions overpay for inclusion because there is
little guidiance on how much they need to pay. Another set of transactions
never make it on-chain because even though they were willling to pay the price for
inclusion, they were unaware of it. The UX degrades further during
periods of high activity and some users are effectively locked out of the chain.
The primary reason for this phenomenon is that the current implementation of 
Solana's fee market is a pure first-price auction (FPA). There is extensive
research and empirical evidence that suggests that this mechanism is suboptimal for users.

There is also extensive research on how to mitigate the externalities of the mechanism and restricting the auction bidspace is the only method that is known to work.

EIP-1559 is an implementation of a restricted bidspace, but it comes with it's own problems including an extreme underutilization of network resources and underutilization of block space (when the base fee is too high.)
Additionally, a naive replication of the 1559 mechanism on Solana is non-beneficial as Solana is a multi-resource (contextually: local fee-market) environment.
EIP-1559 is also an invasive change as it requires modification of the core 
protocol and adds additional overhead to voting nodes who must keep track
of the base fee.

Therefore, a mechanism that can achieve the desired results without any of the externalities discussed above is highly desirable.

## New Terminology
- target compute unit utilization; This is the protocol defined target compute unit 
utilization (can be per block or per account). This is a soft cap on how many
compute units can be fit in a valid block.

- maximum compute unit utilization; This is the protocol-defined maximum compute
unit utilization. This is a hard cap on the number of compute units that can be
packed into a valid block.

- slack: the ratio of target CU-utilization to maximum CU-utilization. It  is a measure of how much congested the corresponding account or global market is.

- recommended priority fee: This is the priority fee that the mechanism "recommends" that
a transaction pay. It is calculated per account by the *priority fee recommender* based
on *recent fees paid*. It is similar to but distinct from a per-account EIP-1559 base fee.

- cache: An in-memory a cache that tracks the most congested accounts, the Exponential
Moving Average (EMA) of the compute unit utilization, and the recommended priority fees for
the corresponding accounts.

- `getPriorityFee`: A JSON RPC method to replace the `getRecentPrioritizationFees` and
`getFeeForMessage` methods. The `getPriorityFee` request takes a mandatory list of
transactions and returns the recommended fee to land a transaction locking all the accounts
in the list.

- fee: fee is used loosely throughout this document to refer to compute unit cost.

## Detailed Design
Solana's fee market is designed such that only users bidding to access **congested**
(accounts for which demand is greater than supply) accounts need to pay more. Everyone else
(that is strictly interested in quick inclusion) only need to pay the minimum fee for inclusion which is determined by the global demand for blockspace.
Unfortunately, as is the case with FPAs, the correct fee for the desired outcome is undefined and unknown to
users.

The most useful tools that currently exist are the `getRecentPrioritizationFees` RPC method and other propitery modifications that all do roughly the same thing:
- take a list of accounts locked by a transaction 
- return the priority fee paid by a transaction that locked all accounts in the list in the last 150 blocks.

These tools, while useful, are subpar because:
- they do not (correctly) price blockspace; they make recommendations based on (previous) uninformed bids.
- they waste RPC resources (150 blocks worth of data is far too much, and can even be harmful).

Furthermore because different providers have different implementations, there is some loss in efficiency.

These are the gaps that this proposal intends to address.

In the proposed system; when a user makes a `getPriorityFee` request to an RPC node; the node checks the attached list of transactions against an in-memory cache that tracks congested accounts and returns the recommended fee for the most expensive (should also be the most congested) account in the list. If none of the accounts in the list are in the cache then the method simply returns the recommended global priority fee. The fee is described as a recommendation because there is no enforcing that transactions pay
(at least) this fee to be included (EIP-1559 base-fee-style). The recommendation will remain like a normal bid to validators.

The defining benefits of this approach compared to a protocol-enforced base fee are:
1. blocks can be maximally packed at all times, and
2. there is no additional overhead from tracking a base fee on the protocol; only RPC nodes that are already
responsible for responding to these types of requests have to keep the "cache".

### RPC Processing Requests

When an RPC node receives a `getPriorityFee` request it:
- checks the cache for all the accounts in the accounts list attached.
- if none of the accounts in the list are in the cache then it simply returns the recommended
global priority fee.
- if one or more of the accounts in the list are in the cache, then it returns the recommended
per-account priority or high-priority fee for the most contentious account (should also
be the most expensive).

### The cache

At the core of this proposal is a 
cache that tracks congested accounts and maps the accounts to the "recommended"
priority fee (calculation discussed below). Global CU utilization is also tracked.

The "cache" is a finite capacity in-memory data structure maintained by RPC nodes that tracks congested accounts and maps accounts to an `AccountData` struct.

```Rust
Cache<Pubkey, AccountData>
```

The `AccountData` struct contains:
- the exponential moving average (EMA) of the compute unit utilization of the associated account in the five most recent blocks,
- the recommended fee for block n (the most recent block seen by the node),
- the recommended fee that transactions that want to access the account in the next block (n + 1) should pay.

```Rust
struct AccountData {
	ema_of_cu_utilization_over_the_last_five_blocks // n-4 through n
	median_fee_in_block_n
    recommended_priority_fee_to_access_account_in_block_n_plus_one
}
```

The median (as opposed to a previous recommendation) is tracked because it allows the mechanism to detect changes in demand at a particular price point without setting a base fee (that would do so by dropping transactions below the price point).

The global median fee and global compute unit utilization are also tracked.

```Rust
struct GlobalData {
	ema_of_global_cu_utilization_over_the_last_five_blocks
	median_fee_global_in_block_n
	recommended_priority_fee_global_in_block_n_plus_one
}
```

### Target and Maximum Compute Unit Utilization

This proposal introduces a target compute unit utilization value at both block and account level.

Settting this value is required because it is necessary to allow the TFM to reliably determine when demand outstrips supply. If there is no target utilization amount it is difficult to detect if demand matches or outstrips supply. Additionally, distinguishing between the two values improves the UX by allowing additional transactions when there is a sudden increase in demand.

The greater the *slack* (difference between target and maximum compute unit utilization) the more pronounced the aforementioned effects. However, setting the target CU utilization to half of the maximum compute unit uilization like in EIP-1559 results in severe underutilization of block space. Because of this, it is proposed that the target CU utiization be 85% of the maximum CU utilization (i.e., target cu_utilization = $0.85$ * max cu utilization) to balance the benefits and externalities of setting a target compute unit utilization.

Keep in mind that this does not require any changes to the core protocol.

### How the recommended priority fee is calculated

The recommended priority fees are determined based on a simple princriple; if the EMA of CU-utilization is lower than the target, recommend a lower fee than the median of the previous block and if it is higher, recommend a higher fee. The degree of how much higher (or lower) depends on the difference between the observed value and target. Mathematically, this can be expressed as:   

let $n$ be the most recent block seen by the node (such that the next block is $o$)  
let $f^g_n$ be the global median fee in block $n$  
let $f^g_o$ be the recommended global fee for block $o \ni  o = n + 1$  
let $\mu^g_{\lambda}$ be the  EMA of global compute unit utilization over the last five blocks  
let $\mu^g_{\tau}$ be the target per block compute unit utilization  
let $\theta$ be a sensitivity parameter 

The recommended global fee for block $o$ is determined by:  
if $\mu^g_{\tau} > \mu^g_{\lambda}$,  
$\ \ \ \ f^g_o = f^g_n * exp(\theta, \ (\frac{\mu^g_{\lambda}}{\mu^g_{\tau}} -1))$    
  
if $\mu^g_{\tau} < \mu^g_{\lambda}$,  
$\ \ \ \ f^g_o = f^g_n * (1 - exp(\theta, \ (\frac{\mu^g_{\lambda}}{\mu^g_{\tau}} -1)))$


let $n$ be the most recent block seen by the node (such that the next block is $o$)  
let $f^{\alpha}_n$ be the median fee for account $\alpha$ in block $n$  
let $f^{\alpha}_o$ be the recommended global fee for block $o \ni  o = n + 1$  
let $\mu^{\alpha}_{\lambda}$ be the  EMA of compute unit utilization for account ${\alpha}$ over the last five blocks  
let $\mu^{\alpha}_{\tau}$ be the target per block compute unit utilization  
let $\theta$ be a sensitivity parameter 

The recommended per account fee for block $o$ is determined by:  
if $\mu^{\alpha}_{\tau} > \mu^{\alpha}_{\lambda}$,  
$\ \ \ \ f^{\alpha}_o = f^{\alpha}_n * exp(\theta, \ (\frac{\mu^{\alpha}_{\lambda}}{\mu^{\alpha}_{\tau}} -1))$    
  
if $\mu^{\alpha}_{\tau} <  \mu^{\alpha}_{\lambda}$,  
$\ \ \ \ f^{\alpha}_o = f^{\alpha}_n * (1 - exp(\theta, \ (\frac{\mu^{\alpha}_{\lambda}}{\mu^{\alpha}_{\tau}} -1)))$  

As is observable from the above, the controller is exponential. An exponential controller was chosen for two reasons:
1. There is no desire to impose additional costs on users unless they are bidding for congested accounts. Because of this, the slack (difference between target and maximum utilization) must be as small as possible. Given the small slack, aggressive responses are crucial even if they cause some loss in efficiency. However, because the TFM does not enforce the base fee, as long as there is sufficient blockspace, users with lower willingness to pay can still be included.
2. Research shows that transaction arrival can be modelled by a Poisson's process and expoenential controllers are effective for this class of problems given the additional desiderata.

### Cache updates and eviction

Every RPC node indepently updates its cache when it receives a new block.
During each update, the global and per account EMAs are updated and the recommended fees for the upcoming block are calculated according to the relations above.

Also the entries with the least congestion are evicted and the most congested accounts not already in the cache are inserted.

To allow effective tracking of congested accounts, we propose tracking up to $5 * ceil(\frac{max \ block \ compute \ unit \ utilization}{target \ per \ account \ compute \ unit \ utilization})$ accounts with utilization greater than 50% of the target utilization.

This value is based on the fact that:
1. the EMA is tracked over 5 blocks
2. the maximum number of accounts that can satisfy the criteria of being congested is given by $ceil(\frac{max \ block \ compute \ unit \ utilization}{target \ per \ account \ compute \ unit \ utilization})$.

This is a simple but effective way to set an upper bound on the cache size, it is also small enough that there is no meaningful overhead from keeping it--under the current conditions (48 M CUs per block and 12M per CUs per account), the cache will have a capacity of just 25.

Finally we propose a (maximum) churn rate of 5 entries per slot for the same reason as above.

## Alternatives Considered

What alternative designs were considered and what pros/cons does this feature
have relative to them?

1. EIP-1559-like system with a protocol-enforced base fee.
The primary alternative consideration is a per-account EIP-1559 implementation. But for reasons
already discussed in the main body of this text, the proposed implementation is superior to
EIP-1559.

2. Burning the base fee completely.
In addition to the features above, burning the per signature base fee was also considered but the benefits
of such a mechanism are debatable. Although, there are no known drawbacks.

3. Doing Nothing.
Users will continue to overbid (best case) and the UX will continue to be subpar.

## Impact

- validators: unaffected
- core contributors: unaffected.
- RPC Nodes: They improve transaction landing probability for users.
- users: Users can make better-informed bids and enjoy a significantly better experience. They should also pay less for uncongested accounts.
- dapp developers: they will have to rewrite applications and switch to the new method.

## Security Considerations

None

## Drawbacks

Extreme care must be taken to ensure that TFM modifications do not create attack vectors, especially for myopic block producers. But seeing as the new proposal does not actually modify any parts of the existing mechanism, by reduction, it is as sound as the existing mechanism. 
However, as discussed earlier, the existing TFM is already not incentive compatible for myopic block producers and although this proposal does not cmake it worse, it does not make it better either.

## Backwards Compatibility

Yes, core code remain unchanged.
