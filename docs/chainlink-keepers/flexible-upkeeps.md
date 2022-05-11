---
layout: nodes.liquid
section: ethereum
date: Last Modified
title: 'Build highly flexible, cheap, and secure automation using Chainlink Keepers'
whatsnext: { 'Utility Contracts': '/docs/chainlink-keepers/util-overview/', 'FAQs': '/docs/chainlink-keepers/faqs/' }
---

## Overview

In this tutorial, you will learn how the flexibility of [Chainlink Keepers](https://chain.link/keepers) enables important design patterns that reduce gas fees, enhance the resilience of dApps, and improve end-user experience.
You will start by integrating an unoptimized contract to Chainlink Keepers (cf. [Problem](#problem-on-chain-computation-leads-to-high-gas-fees)). Then, you will see how properly leveraging the flexibility of Chainlink Keepers (cf. [Solution section](#solution-perform-complex-computations-with-no-gas-fees)) allows you to save ~84% of gas fees!

**Table of Contents**

- [Prerequisites](#prerequisites)
- [Problem: On-chain computation leads to high gas fees](#problem-on-chain-computation-leads-to-high-gas-fees)
- [Solution: Perform complex computations with no gas fees](#solution-perform-complex-computations-with-no-gas-fees)
- [Conclusion](#conclusion)

## Prerequisites

Smart contracts themselves cannot self-trigger their functions at arbitrary times or under arbitrary conditions. Transactions can only be initiated by another account.
On August 5th, 2021, we [announced](https://blog.chain.link/chainlink-keepers-is-now-live-on-mainnet/) an important [hybrid smart contract](https://blog.chain.link/hybrid-smart-contracts-explained/) innovation: [Chainlink Keepers](https://chain.link/keepers).
By using Chainlink Keepers, developers don’t need to rely on centralized services or implement the infrastructure themselves to automate on-chain functions.
Several DeFi projects already live on [Ethereum Mainnet](https://keepers.chain.link/mainnet), [Polygon Mainnet](https://keepers.chain.link/polygon), and [Binance Smart Chain Mainnet](https://keepers.chain.link/bsc) benefit from the decentralized and secure automation brought by Chainlink Keepers.

This tutorial assumes you have a basic understanding of [Chainlink Keepers](https://chain.link/keepers). If you are new to Keepers, then please start with the following:

- [Chainlink Keepers announcement](https://blog.chain.link/chainlink-keepers-is-now-live-on-mainnet/)
- [Introduction](/docs/chainlink-keepers/introduction/)
- [Making Compatible Contracts](/docs/chainlink-keepers/compatible-contracts/)
- [Register UpKeep for a Contract](/docs/chainlink-keepers/register-upkeep/)

Chainlink Keepers are supported on these [networks](../supported-networks).
We will be using Remix. Hence, you must be familiar with [deploying a solidity contract using Remix and Metamask](/docs/deploy-your-first-contract/).

> 📘 ERC677 Link
>
> - Get [LINK](/docs/link-token-contracts/) on the supported testnet that you want to use.
> - For funding on Mainnet, you need ERC-677 LINK. Many token bridges give you ERC-20 LINK tokens. Use PegSwap to [convert Chainlink tokens (LINK) to be ERC-677 compatible](https://pegswap.chain.link/).

## Problem: On-chain computation leads to high gas fees

In the 3 steps tutorial, we deployed a basic [counter contract](/docs/chainlink-keepers/compatible-contracts/#example-contract) and verified that at every 30 seconds, the counter was incremented. However, more complex use cases can require looping over arrays or performing expensive computation. This leads to expensive gas fees and ultimately increases the premium end-users have to pay to use your dApp.
To illustrate this, let’s deploy a very simple contract that maintains internal balances.

Let’s walk through the contract:

- The contract has a fixed-size(1000) array `balances`. Each element of the array starts with a balance of 1000.
- At any time, we can call the `withdraw()` function to decrease the balance of several indexes of the `balances` array.
- Keepers are responsible for regularly triggering the rebalancing of the elements. This is done by:
  - Calling `checkUpkeep()` function to check if the contract requires work to be done. This will be the case if one array element has a balance of less than `LIMIT`. In this case, the function returns `upkeepNeeded == true`.
  - Calling `performUpkeep()` function to re-balance the elements. Note that all the computation is done within the transaction: The function finds all the elements which are less than `LIMIT`, decreases the contract `liquidity` and increases every found element to `LIMIT`.

```solidity
{% include 'samples/Keepers/BalancerOnChain.sol' %}
```

<div class="remix-callout">
    <a href="https://remix.ethereum.org/#url=https://docs.chain.link/samples/Keepers/BalancerOnChain.sol" >Open in Remix</a>
    <a href="/docs/conceptual-overview/#what-is-remix" >What is Remix?</a>
</div>

Follow these steps to test the example:

1. Deploy the contract using Remix on the [supported testnet](../supported-networks) of your choice.

1. Before registering the upkeep for your contract, decrease the balances of some elements. On Remix:
   Withdraw 100 at 10,100,300,350,500,600,670,700,900. Pass the following to the withdraw function: 100,[10,100,300,350,500,600,670,700,900]

   ![Withdraw 100 at 10,100,300,350,500,600,670,700,900](/images/contract-devs/keeper/balancerOnChain-withdraw.png)

   Note that you can also perform this step after registering the upkeep.

1. Register the upkeep for your contract as explained [here](/docs/chainlink-keepers/register-upkeep/). Because this example has high gas requirements, specify the maximum allowed gas limit of 2,500,00.

1. Once registration is confirmed, you will notice that an upkeep has been completed.

   ![BalancerOnChain Upkeep History](/images/contract-devs/keeper/balancerOnChain-history.png)

1. Click the transaction hash to see the transaction details in Etherscan. There you can find how much gas was used in the upkeep transaction.

   ![BalancerOnChain Gas](/images/contract-devs/keeper/balancerOnChain-gas.png)

In this example, the `performUpkeep()` function used **2,481,379** gas. This example has two main issues:

- All computation is done in `performUpkeep()`. This is a state modifying function which leads to high gas consumption.
- This example is simple but looping over large arrays with state updates can cause the transaction to hit the gas limit of the [network](../supported-networks), preventing `performUpkeep` from running successfully.  
  In the next [section](#solution-perform-complex-computations-with-no-gas-fees), you will leverage the flexibility of Chainlink Keepers to reduce gas fees and mitigate the risks of running out of gas.

## Solution: Perform complex computations with no gas fees

We will slightly modify the contract and move the computation to `checkUpkeep()` function, which <ins>doesn’t consume any gas</ins>. Moreover, we will support multiple upkeeps for the same contract to parallelize the work to be done.
The main differences between our new contract and the previous contract are the following (cf. contract code below):

- The `checkUpkeep()` function receives [`checkData`](/docs/chainlink-keepers/compatible-contracts/#checkdata), which is used to pass arbitrary bytes to the function. In this case, we will pass a `lowerBound` and an `upperBound` to scope the work to a subarray of `balances`. This will allow us to create several upkeeps with different values of `checkData`. The function loops over the subarray and looks for the indexes of the elements which require balancing and calculates the required `increments`. It then returns `upkeepNeeded == true`, and `performData` , which is calculated by encoding `indexes` and `increments`. Note that `checkUpkeep()` is a view function, meaning that any computation does not consume any gas.
- The `performUpkeep()` function takes [performData](/docs/chainlink-keepers/compatible-contracts/#performdata-1) as a parameter then decodes it to fetch the `indexes` and the `increments`.

> ⚠️ **Note on `performData`**
>
> This data should always be validated against the contract’s current state to ensure that `performUpkeep()` is idempotent. It also blocks malicious keepers from sending non-valid data. In this example, we test that the state is correct after rebalancing:
> `require(_balance == LIMIT, "Provided increment not correct");`

```solidity
{% include 'samples/Keepers/BalancerOffChain.sol' %}
```

<div class="remix-callout">
    <a href="https://remix.ethereum.org/#url=https://docs.chain.link/samples/Keepers/BalancerOffChain.sol" >Open in Remix</a>
    <a href="/docs/conceptual-overview/#what-is-remix" >What is Remix?</a>
</div>

Let’s now perform the same test to compare the gas fees:

1. Deploy the contract using Remix on the [supported testnet](../supported-networks) of your choice.

1. Withdraw 100 at 10,100,300,350,500,600,670,700,900. Pass the following to the withdarw function: 100,[10,100,300,350,500,600,670,700,900] (Same test as the [previous section](#problem-on-chain-computation-leads-to-high-gas-fees)).

1. Register 3 upkeeps for your contract as explained [here](/docs/chainlink-keepers/register-upkeep/). Because the keepers handle much of the computation off-chain, a gas limit of 200,000 should be sufficient. For each registration, pass the following _CheckData_ values(second column) to specify which balance indexes the registration will monitor. **Note**: Remove any breaking line when copying the values.

   | Upkeep Name             | CheckData(base16)                                                                                                                                      | Remark: calculated using [`abi.encode()`](https://docs.soliditylang.org/en/develop/abi-spec.html#strict-encoding-mode) |
   | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
   | balancerOffChainSubset1 | 0x000000000000000000000000<br/>00000000000000000000000000<br/>00000000000000000000000000<br/>00000000000000000000000000<br/>0000000000000000000000014c | lowerBound: 0<br/>upperBound: 332                                                                                      |
   | balancerOffChainSubset2 | 0x000000000000000000000000<br/>00000000000000000000000000<br/>0000000000014d000000000000<br/>00000000000000000000000000<br/>0000000000000000000000029a | lowerBound: 333<br/>upperBound: 666                                                                                    |
   | balancerOffChainSubset3 | 0x000000000000000000000000<br/>00000000000000000000000000<br/>0000000000029b000000000000<br/>00000000000000000000000000<br/>000000000000000000000003e7 | lowerBound: 667<br/>upperBound: 999                                                                                    |

1. After the registration is confirmed, you will notice that three upkeeps have run.

   ![BalancerOffChain1 History](/images/contract-devs/keeper/balancerOffChain1-history.png 'balancerOffChainSubset1')

   ![BalancerOffChain2 History](/images/contract-devs/keeper/balancerOffChain2-history.png 'balancerOffChainSubset2')

   ![BalancerOffChain3 History](/images/contract-devs/keeper/balancerOffChain3-history.png 'balancerOffChainSubset3')

1. Click each transaction hash to see the details of each transaction in Etherscan. Find the gas used by each of the upkeep transactions:

   ![BalancerOffChain1 Gas](/images/contract-devs/keeper/balancerOffChain1-gas.png 'balancerOffChainSubset1')

   ![BalancerOffChain2 Gas](/images/contract-devs/keeper/balancerOffChain2-gas.png 'balancerOffChainSubset2')

   ![BalancerOffChain3 Gas](/images/contract-devs/keeper/balancerOffChain3-gas.png 'balancerOffChainSubset3')

The total gas used by each `performUpkeep()` function was 133,464 + 133,488 + 133,488 = **400,440**. This is an improvement of **~84%** compared to the previous example(**2,481,379**)!

## Conclusion

In this tutorial, we’ve seen how we can leverage the flexibility of Chainlink Keepers to reduce gas fees.

Using Chainlink Keepers efficiently not only allows you to reduce the gas fees, but also keeps them within predictable limits. That’s the reason why [several Defi protocols](https://chainlinktoday.com/prominent-founders-examine-chainlink-keepers-role-in-defis-evolution/) outsource their maintenance tasks to Chainlink Keepers.
