---
title: 'Orbit Chain Configuration Guide'
sidebar_label: 'Orbit Chain Configuration'
description: 'Arbitrum SDK '
author: Mehdi Salehi
sme: Mehdi Salehi
target_audience: 'Developers deploying and maintaining Orbit chains.'
sidebar_position: 6
---
This guide will explore the essentials of configuring your Orbit chain. Configuration includes a range of settings from the parent chain and node configuration to specific child chain parameter configurations. Our focus here is on child chain configuration, guiding you through the necessary adjustments to customize your chain's operation.

In the introductory section, we outlined three primary configurations required for setting up and managing an Orbit chain. These configurations play a crucial role in ensuring the efficient operation of your chain. Let's delve into each type:

## 1. Parent Chain Configuration
Configuring the parent chain is an essential initial step in setting up your Orbit chain. Most of these configurations are specified during the setup phase. Detailed instructions can be found in the [Rollup Deployment Parameters](deployment-rollup#rollup-deployment-parameter) section of the rollup deployment guide. 

After the initial setup, the chain owner can modify configurations as needed. For instance, the validator set can be updated by invoking the [`setValidKeyset`](https://github.com/OffchainLabs/nitro-contracts/blob/90037b996509312ef1addb3f9352457b8a99d6a6/src/bridge/SequencerInbox.sol#L751) function with a new set of validators. This adaptability facilitates continuous optimization and management of the chain.

## 2. Node/Chain Configuration
This category includes settings adjustable within the `nodeConfig.json` file, directly impacting the operation of the chain's nodes, including special nodes like validators and sequencers. These settings are vital for tailoring the node's functionality to specific requirements or performance criteria. The chain owner can modify these configurations during the node config generation process, ensuring that each node operates with the desired settings from the start. For more information, refer to the [Node Configuration Preparation](node-config-preparation.md) documentation.

## 3. Child Chain Parameter Configuration
The final configuration type involves setting parameters on the child chain. This level of configuration is primarily achieved through the [ArbOwner precompile](https://github.com/OffchainLabs/nitro-contracts/blob/main/src/precompiles/ArbOwner.sol) on the child chain. These configurations are typically applied post-chain initialization and after the deployment of the token bridge. This guide will help you configure child chain parameters using the Orbit SDK, providing insights into effective management and optimization strategies for your child chain operations.

The child chain configuration can be performed after the chain initialization. These parameters are configurable via setter functions in the [ArbOwner precompile](https://github.com/OffchainLabs/nitro-contracts/blob/main/src/precompiles/ArbOwner.sol). Additionally, there are various getter functions in the ArbOwner precompile that you can use to read the current configuration. Below, we explain several methods in the ArbOwner precompile that you can use to configure the parameters or read their current state.

### Setter Functions
You can use these setter functions to configure the child chain parameters:

- `addChainOwner`: Allows you to add a new chain owner to your Orbit chain.

- `removeChainOwner`: Enables you to remove an existing owner from the list of chain owners.

- `setMinimumL2BaseFee`: Sets the minimum base fee on the child chain. The minimum base fee is the lowest amount that the base fee on the child chain can ever be. For example, the current minimum base fee on Arbitrum One is 0.1 gwei, and on Arbitrum Nova, it is 0.01 gwei.

- `setSpeedLimit`: The fee mechanism on the Arbitrum Nitro stack differs from the Ethereum blockchain. The Nitro stack has a parameter called the speed limit, which targets the number of gas consumed on the child chain per second. If the amount of gas per second exceeds this pre-specified amount, the base fee on the child chain will increase, and vice versa. The current speed limit on Arbitrum One is 7 million gas per second, meaning if the Arbitrum One chain consumes more than 7 million gas per second, its base fee will increase. For more information on the speed limit, please refer to this [document explaining the concept of speed limit in the Nitro stack](https://docs.arbitrum.io/inside-arbitrum-nitro/#the-speed-limit).

- `setInfraFeeAccount`: Sets the infrastructure fee account address, which receives all fees collected on the child chain. It is meant to receive the minimum base fee, with any amount above that going to the network fee account.

- `setNetworkFeeAccount`: Sets the network fee account address. As mentioned, this address collects all fees above the base fee. Note that if you set this amount to the 0 address on your chain, all fees will be deposited into the infrastructure fee account.

- `scheduleArbOSUpgrade`: If you plan to upgrade the <a data-quicklook-from="arbos">ArbOS</a> version of your chain, this method can help you schedule the upgrade. For a complete guide on this matter, please refer to the explanation of the [arbos upgrade](../how-tos/arbos-upgrade.md).

- `setChainConfig`: We discussed the chainConfig in the [Rollup deployment guide](deployment-rollup.md#chain-config-parameter) in detail. If you wish to change any field of the `chainConfig`, you need to use this method on the child chain.

### Getter Functions

To read the child chain parameters, you can use these getter functions:

- `getAllChainOwners`: Provides the list of all current chain owners.

- `getNetworkFeeAccount`:  Returns the network fee account address.

- `getInfraFeeAccount`:  Returns the infrastructure fee account address.

- `isChainOwner`:  Allows you to check whether an address is on the list of chain owners.

### Configuring the Child Chain Using the Orbit SDK

In the Orbit SDK, we use the [Client Extension](https://viem.sh/docs/clients/custom#extending-with-actions-or-configuration) feature of Viem to extend the public client. In the Orbit SDK, we defined `arbOwnerPublicActions` to use it and extend the client on Viem. An example of creating a public client extended with arbOwner public actions is:

```js
import { createPublicClient, http } from 'viem';
import { arbOwnerPublicActions } from '@arbitrum/orbit-sdk';

const client = createPublicClient({
  chain: arbitrumLocal,
  transport: http(),
}).extend(arbOwnerPublicActions);
```

With `arbOwnerPublicActions` and the public client in the Orbit SDK, we've added two new methods to the public clients:

#### 1. arbOwnerReadContract 

This method can be used to read the parameters of the child chain discussed in the [previous section](#getter-functions). An example of using this method with the `client` defined in the previous section is:

```js
  const result = await client.arbOwnerReadContract({
    functionName: 'getAllChainOwners',
  });
```

Changing the function names in the list in this [section](#getter-functions) will give you the other parameters.

#### 2. arbOwnerPrepareTransactionRequest

This method can be used to configure the parameters on the ArbOwner precompile, which are listed in this [section](#setter-functions). An example of utilizing this method to configure parameters using the `client` defined in the previous section is:

```js
  // Adding a random address as chain owner using the upgrade executor
  const transactionRequest = await client.arbOwnerPrepareTransactionRequest({
    functionName: 'addChainOwner',
    args: [randomAccountAddress],
    upgradeExecutor: false,
    account: owner.address,
  });

  // Submitting the transaction to add a chain owner
  await client.sendRawTransaction({
    serializedTransaction: await owner.signTransaction(transactionRequest),
  });
```

To use this method, as shown in the example above, some inputs need to be defined. `functionName` is the name of the method you want to use to set the parameter, which can be found in [this section](#setter-functions). `args` are the arguments for the defined function. The `upgradeExecutor` field specifies whether an `upgradeExecutor` contract governs your chain. If it is not using an `upgradeExecutor`, you can set it to `false`, similar to the example above. Also, the `account` field defines the chain owner if an `upgradeExecutor` does not govern the chain.

If an `upgradeExecutor` contract governs your chain, then you need to use the `arbOwnerPrepareTransactionRequest` method, similar to the example below:

```js
  // Adding a random address as chain owner using the upgrade executor
  const transactionRequest = await client.arbOwnerPrepareTransactionRequest({
    functionName: 'addChainOwner',
    args: [randomAccountAddress],
    upgradeExecutor: upgradeExecutorAddress,
    account: owner.address,
  });

  // Submitting the transaction to add a chain owner
  await client.sendRawTransaction({
    serializedTransaction: await owner.signTransaction(transactionRequest),
  });

```

In this example, all the fields are the same as in the first example, except the `upgradeExecutor` field, which you need to set to the upgradeExecutor address, and the `account` parameter, which needs to be set to the owner of the upgradeExecutor contract.

