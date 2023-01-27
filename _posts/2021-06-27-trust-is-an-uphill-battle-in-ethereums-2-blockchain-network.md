---
title: Trust is an uphill battle in Ethereum's 2.0 blockchain network
layout: post
author: Michael Nguyen
toc: true
---

## Summary
People often praise [Ethereum 2.0’s](https://ethereum.org/en/eth2/) security, however the security risks posed by new staking protocols is often downplayed. Staking protocols that are **decentralised** & don’t need to know staker **withdrawal keys** will play a critical role in eth 2.0 blockchain network security, & is not often discussed in detail.

## Centralized smoke and mirrors
In Ethereum 2.0, validator maintenance requirements are a *relative* hassle (not as high as [Solana](https://docs.solana.com/running-validator/validator-reqs), but still more hassle than most stakers will ever bother with). Since humans like money, lots of ETH holders will just stake their ETH using a protocol that takes custody of their ETH (and/or custody of their withdrawal keys) which upholds the maintenance requirements for them. An example of this would be a [CEX](https://www.binance.com/en), or [Lido](https://lido.fi/).

And this means we will have validator pools forming. In fact, *we already do*. Lido (a liquid staking provider) & the major centralised exchanges effectively support this functionality already. This is fine - the problem is that these are all **centralised** or take custody of **withdrawal keys**, & the security they uphold on their validator hardware remains unknown. Its a system based on trust, which is sub-optimal compared to trust-less systems.

Hence, these providers are good, but still introduce a **security risk** to the entire eth 2.0 network that could be solved with decentralisation (a difficult problem to solve, but we will get to that later). People say that the network wont be attacked because the attacker's ethereum gets slashed, but attackers can hack other validators & burn other people's deposits instead.

Low risk & high reward baby! Is it even possible to solve the decentralisation problem whilst [preserving other important attributes](https://docs.ethhub.io/ethereum-roadmap/ethereum-2.0/sharding/)?

## Decentralising human trust is bloody hard
This vulnerability of validator pools isn't discussed alot, & is often downplayed.

This vulnerability must be removed through decentralisation. Whilst *"truly"* decentralised staking protocols don’t exist just yet, there is 1 protocol paving a way towards it. And they solve it by hosting validators on numerous **cloud providers** which have a strong track record of [IaaS](https://azure.microsoft.com/en-au/overview/what-is-iaas/) security. This is in contrast to a **DAO** approving validator providers.

This decentralised protocol is called [Rocketpool](https://www.rocketpool.net/).

While not yet released on the main-net (at time of writing), its decentralises staking via hosting validators on numerous cloud providers. And let’s be honest, the cloud providers such as Microsoft, Amazon & Google are publicly listed companies that have the resources required to maintain a higher level of security & availability, compared to existing providers.

So this means the centralisation problem is solved.. right?

## Free lunch huh? Count me in!
Whilst Rocketpool solves the centralisation issue, it is indeed bleeding edge technology.

Bleeding edge technology solutions such as Rocketpool will usually require trading off 1 quality to improve another one. In Rocketpool's case:
- It requires a high **fee** from users to cover infrastructure costs
- It recently encountered a hurdle (aka **0x02**) where the [eth 2.0 protocol awards staking rewards to the validator directly](https://github.com/ethereum/eth2.0-specs/pull/2454), which in Rocketpool’s case, is actually someone validating for Rocketpool, rather than rocketpool themselves. Thus Rocketpool will be unable to distribute those staking rewards to their token holders (stakers)
- Its remains to be battle tested. As usual, new smart contracts potentially have **vulnerabilities** undiscovered by auditors

And the most important one that is basically **NOT discussed anywhere** - these "liquid staking" providers involving conversions between tokens, marking a [capital gains tax (CGT)](https://www.ato.gov.au/general/capital-gains-tax/) event & removing potential long-term CGT advantages that usually comes with staking alone.

## Great success?
With the above 4 points said (and whilst the internet usually argues 1 of the above), one can state that these tradeoffs are worth the network security improvement. Especially a [financially-oriented blockchain](https://defipulse.com/defi-list/) network such as Ethereum.

From my subjective perspective, Rocketpool's decentralized staking protocol will be a critical cornerstone (an absolute *necessity* , even) in the ethereum 2.0 network.

If the protocol fails, the potential security of the blockchain network decreases. And guess what happens if Rocketpool doesn't work? People will move to existing staking protocols that take custody of their **private** or **withdrawal** keys. And *HACKING* the DAO/company that holds those keys is a lot easier than breaching a reputable cloud provider such as AWS.

Trust is an uphill battle in Ethereum's 2.0 blockchain network, & whilst we can shed light on the vulnerabilities & disadvantages of existing staking protocols, Rocketpool appears to be the only protocol brave enough to tackle the problem head-on.

Will it succeed? Well, we should damn well hope so.
