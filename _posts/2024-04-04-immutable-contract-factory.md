---
title: Immutable Contract Factory
date: 2024-04-04 18:00:00 -0500
categories: [Deployment]
author: slvrfn
img_path: /blog
image:
  path: /icf_header.png
  alt: Visualization of an immutable contract factory
tags: [create2, create3, deployment, evm]
---

# Deterministic EVM Deployments and the Immutable Deployment Factory

The Ethereum blockchain has pioneered decentralized applications (dapps) with the introduction of smart contracts, forever changing how we participate in digital agreements and transactions. With the evolving landscape of blockchain technology, the methods for deploying these smart contracts have also advanced, leading to more efficient, secure, and deterministic deployment processes.

We're here today to teach you about EVM smart contract deployments and tell you about a factory contract that we've created (the `ImmutableDeploymentFactory`) that exploits some technical tricks to allow for deterministic contract deployments across any Ethereum virtual machine (EVM) compatible chain. This enables users/organizations to deploy contracts to the same address cross-chain, establishing a unified identity, while optionally choosing the contract's address to meet some vanity criteria.

This blog post explores the `create`, `create2`, and `create3` deployment techniques, their importance, and how they can be used to produce immutable smart contracts across any EVM-compatible chain.

> If you want to skip to the good part, the code for the `ImmutableDeploymentFactory` can be found [at this link](https://github.com/VitriolLabs/immutable-deployment-factory)

## The Basics

Before beginning, we assume you're familiar with the following concepts:

- Contract bytecode [\[1\]](https://medium.com/@eiki1212/explaining-ethereum-contract-abi-evm-bytecode-6afa6e917c3b) [\[2\]](https://blog.chain.link/what-are-abi-and-bytecode-in-solidity/)
- EOA vs Smart contract account [\[1\]](https://developers.circle.com/w3s/docs/programmable-wallets-account-types) [\[2\]](https://blog.ambire.com/eoas-vs-smart-contract-accounts/)
- Address nonce [\[1\]](https://www.quicknode.com/guides/ethereum-development/transactions/how-to-manage-nonces-with-ethereum-transactions#what-is-a-nonce) [\[2\]](https://help.myetherwallet.com/en/articles/5461509-what-is-a-nonce)
- Role of a cryptographic Salt [\[1\]](https://en.wikipedia.org/wiki/Salt_(cryptography))

## Why Deterministic Deployments?

### Gas Savings and Vanity Addresses

Deterministic deployment methods like `create2` and `create3` can be used to generate vanity addresses, or even optimize gas costs.

This can be a useful tool for individuals & organizations to provide an extra layer of trust to their online presence by establishing a unified identity. For instance, an organization can choose to deploy contracts to the same address across any EVM chain. This allows users to have a consistent address to interact with, something that can be very useful to establish trust and brand-identity. This means users do not have to juggle as many addresses when trying to interact with some cross-chain dapp.

Somewhat surprisingly, users save gas when interacting with contracts that are deployed to addresses that contain more 0-bytes. This is due to how the EVM encodes transactions. The gas savings per-transaction are not high (about 64 gas per 0-byte), but add up when you consider a smart contract that has many users such as Uniswap or OpenSea's Seaport. For example, If one of these protocols were to deploy to an address with say 4 leading 0-bytes (`0x00000000...`), that would save 256 gas per-transaction which when you have thousands of users quickly grows to be huge gas savings.

Additionally, deterministic deployments enable the creation of vanity addresses, which are custom contract addresses with desirable patterns, adding a layer of personalized branding to smart contracts. Since EVM addresses are represented as hexadecimal, they are able to contain the characters `0-9` and `a-f`. Even though there are a reduced number of characters available, you can still construct some interesting addresses such as:  `0xdeadbeef...` `0x0000000...` `0x0f0f0f0f...`.

While calculating deterministic deployment is a solved problem, searching for *desirable* deterministic deployments is a **hard** problem in the sense that they can be quite rare to mine. 0age [has created an excellent breakdown](https://medium.com/coinmonks/on-efficient-ethereum-addresses-3fef0596e263) on the complexity & likelihood of finding various vanity addresses.

### Immutable and Overlapping Deployments

The most significant advantage of deterministic deployments is the ability to know a contract's deployment address before it is actually deployed.

One interesting side effect of this, is the ability to deploy more than one contract to the same address. Contracts such as this are called [metamorphic contracts](https://medium.com/zeppelin-blog/the-promise-and-the-peril-of-metamorphic-contracts-9eb8b8413c5e). This is possible with the `create2` and `create3` opcodes since contract creation is related to a salt instead of deployer's nonce.

One way to combat this, is to set up a factory contract to track which addresses it has previously deployed. By doing so it can prevent deploying additional contracts to the same address, effectively creating "Immutable" smart contracts (such as with a standard `create` deployment). We believe immutable deployments to be the best approach, as they help protect the end-user from a malicious dev who could choose to overwrite an existing contract for some illicit behavior.

We have created such a factory contract, as described in the [later in this post](#the-immutabledeploymentfactory-a-game-changer).

## Enter Deterministic Deployments

We will cover 3 methods of deploying smart contracts on EVM-based chains: `create` `create2` and `create3`.
### The `create` Method

The `create` operation is the original method for deploying smart contracts on the Ethereum blockchain. When a contract is deployed using `create`, its address is determined by the following formula:

```
// create address derivation
keccak256(address ++ nonce)[12:]
```

With the above shown `create` formula, a contract's deployment address can be precomputed and is determined by:
- the creator's address (the sender of the contract-creation transaction)
- the creator's nonce

This makes predicting a contract's deployment address easy, but not that useful. If a user is careful and makes sure to use the same account (starting from the same nonce) on every EVM chain, they would end up deploying a contract to the same address regardless of the contract's bytecode.

The `create` method is the most straight-forward to understand, but it comes with its drawbacks. For instance, if you already are using the deployer's address (multi-chain or not) it is possible, but more difficult to deploy contracts with the same address to more than one chain.

Traditionally the `create` method of contract deployment is only performed by EOA accounts, but can also be accessed by smart contracts (important later).
### The Evolution to `create2`

The next logical step for contract deployments, is to have the ability to deterministically influence a contract's deployment address. This was introduced with the `create2` opcode in [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014) by Vitalik Buterin.

```
// create2 address derivation
keccak256(0xff ++ address ++ salt ++ keccak256(init_code))[12:]
```

With the above shown `create2` formula, a contract's deployment address can be precomputed and is determined by:
- the factory contract's address
- a salt (a 32-byte value)
- the contract's initialization code

This predictability is critical for many decentralized applications, as it allows a contract address to be known before deployment, with that address no longer relying on the deployer's nonce.

This makes predicting a contract's deployment address easy, and also much more useful. Now a user can more easily deploy a contract to the same address, regardless of chain, and does not need to worry about their nonce.

The `create2` method now enables deterministic contract deployments, but it also comes with its drawbacks. Specifically, the contract's bytecode affects the deployment address. This is not necessarily a bad thing, but if a contract's code (or even comments) change, then given the same salt a different deployment address would be found. This *typically* only affects developers during the development cycle, but is still a limitation.

The `create2` method of contract deployment (as of now) can only be performed by smart contracts. This ultimately means that the `address` field of the `create2` address formula is the smart contract which will be making the `create2` call (also known as a factory contract). We have created a factory contract which allows for `create2` contract deployments, [discussed later](#the-immutabledeploymentfactory-a-game-changer).

Due to how the `create2` opcode determines a contract's deployment address, an interesting behavior arises: overlapping deployments [explained later](#immutable-and-overlapping-deployments).

### The Introduction of `create3`

The `create3` method is not an EVM opcode but rather a clever deployment approach that pushes the boundary further, offering even more flexibility by combining the previous `create` and `create2` methods.

In doing so, `create3` allows for a deterministic contract deployment which only depends on the deployer's address (factory contract) and a chosen salt. Now, a user can easily deploy a contract to a consistent address, regardless of chain, and does not need to worry about their nonce **OR** the deployed contract's bytecode.

In an attempt to explain this clearly, consider the following pseudo-code which shows how a contract address is calculated for each deployment method we've covered:

```
create1_addr = create(deployer, nonce)
create2_addr = create2(deployer, salt, init_code)
create3_addr = create3(deployer, salt)
```

Next, consider the following (simplified) contract whose sole purpose is to deploy another contract:

```
contract create1 (deployBytecode) {
	// contract's nonce starts at 1
	return create(address(self), 1, deployBytecode)
}
```

Now, we can finally cover how a create3 deployment is performed. First, the factory contract performs a `create2` deployment of the above `create1` contract with the provided salt. Next, the `create1` contract is called, and provided with the bytecode the user wants deployed:

```
contract create3 (salt, deployBytecode) {
	create1_contract = create2(address(self), salt, create1.initCode)
	
	create3_addr = create1_contract(deploy_bytecode)
	
	return create3_addr
}
```

So again: the `create3` approach deploys a `create` factory contract using a chosen salt, and uses the new factory to deploy the user's bytecode. This removes `create2`'s dependency on the user's bytecode, by focusing on deploying the `create1` factory contract. And since a new `create1` factory contract was created, we know its nonce [starts at 1](https://ethereum.stackexchange.com/a/769/97303) for the `create` call that deploys the user's bytecode.

This may seem somewhat confusing at first, but you should come to realize that by using this approach we can perform deterministic deployments which no longer rely on the contract's bytecode, **or** the deployer's nonce. This makes predicting a contract's deployment address easy and (in my opinion) the most useful both for the developer and the organization they are developing for.

Similar to `create2`, a `create3` contract deployment can only be performed by smart contracts. We have created a factory contract which allows for `create3` contract deployments, [discussed in the next section](#the-immutabledeploymentfactory-a-game-changer).

## The ImmutableDeploymentFactory: A Game-Changer

We have created the `ImmutableDeploymentFactory` which enables deterministic, omnichain smart contract deployments.

> The code for the `ImmutableDeploymentFactory` can be found [at this link](https://github.com/VitriolLabs/immutable-deployment-factory) and is deployed to the Ethereum mainnet at [`0x0000086e1910D5977302116fC27934DC0254266C`](https://etherscan.io/address/0x0000086e1910d5977302116fc27934dc0254266c)

This factory contract allows developers to deploy immutable smart contracts on any EVM-compatible chain to the same address using `create2` and `create3`. This capability is invaluable for projects requiring consistent contract addresses across different networks, while also optionally preventing front-running of deployments.

As we have described, with deterministic deployments all that matters is the factory contract's address, and a chosen salt. This being the case, a malicious party could choose to use a salt consumed on one network to deploy a completely different contract to the same address on a different network. In the case of an organization like Uniswap, this could be disastrous for users moving to a new-chain.

Our factory contract allows for deployment salts to be used that prevent font-running of contract deployments. The `ImmutableDeploymentFactory` accepts salts in two formats:
- `0x1231231231231231231231231231231231231231XXXXXXXXXXXXXXXXXXXXXXXX` - limits deployment to caller `0x1231231231231231231231231231231231231231`
- `0x0000000000000000000000000000000000000000XXXXXXXXXXXXXXXXXXXXXXXX`- can be deployed by any address
> The X's here represent a 'wildcard' and can be any hexadecimal character

As shown in the example above, to prevent front-running of a deterministic deployment you must include the address that will be calling the factory contract to perform a deployment. Or, if front-running is not important, you just include 20 0-bytes in place of the caller's address.

> this front-running behavior is only guaranteed when using the `ImmutableDeploymentFactory`

#### Factory Deployment
In the situation that you want to use the factory on some EVM chain where the factory does not yet exist (`0x0000086e1910D5977302116fC27934DC0254266C`), we have set up a process where anyone can deploy the `ImmutableDeploymentFactory`. We have a full guide on how to do so [in our repository](https://github.com/VitriolLabs/immutable-deployment-factory#deployment). In short, you will need to fund a deployer address determined by a "keyless" contract deployment ([Nick's method](https://yamenmerhi.medium.com/nicks-method-ethereum-keyless-execution-168a6659479c)), and then submit a pre-created deployment transaction. We'll create a post on this topic separately, at some point.

## Conclusion

The evolution from `create` to `create2` to  `create3` marks a significant leap in smart contract deployments, offering developers unprecedented control, security, and even gas-efficiency. The `ImmutableDeploymentFactory` exemplifies this progress, providing a robust tool for deterministic, immutable smart contract deployments across any EVM-compatible blockchain.

I hope you've gained a better understanding of one small aspect of the EVM and how smart contract deployments work. As we move into the ever-changing future of blockchain development, the importance of these innovations cannot be overstated, paving the way for more sophisticated, reliable, and user-friendly decentralized applications.

#### Honourable Mentions

As always, all great works are built on the shoulders' of previous giants. Our `ImmutableDeploymentFactory` would not be possible without the following contributions:

- [Solady's CREATE3 Library](https://github.com/Vectorized/solady/blob/main/src/utils/CREATE3.sol)
- [0age's Pr000xy factory contract](https://github.com/0age/Pr000xy/blob/master/contracts/Create2Factory.sol)
