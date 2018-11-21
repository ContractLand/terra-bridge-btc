## Background
Terra-Bridge is a protocol for interchain token transfers between EVM and non-EVM based blockchains.

### Purpose
The purpose of this protocol is to establish a standardized mechanism for transferring assets between Ethereum and external chains in a permission-less manner.

### Usecases
Cross-chain token transfer protocol on Ethereum is useful in fields such as:
- **Stable coins**: External chain assets such as BTC could diversify the collaterals used in stable currencies on Ethereum.

- **Decentralized exchange**: Ability to transfer external assets onto Ethereum expands the available assets for trading.

- **Financial derivatives**: Interconnecting blockchain based assets opens up possibilities for various financial derivative products.

### Existing works
There are some existing solutions that exists in this field such as the [Parity Bridge](https://github.com/paritytech/parity-bridge), [PoA Bridge](https://github.com/poanetwork/poa-bridge). However, these solutions are not meant to bring assets onto Ethereum, but rather scalability solutions for EVM based side-chains. Therefore these solutions are limited to EVM <-> EVM transfers under a permissioned authority based schema.

[BTC Relay](https://github.com/ethereum/btcrelay) was one of the first attempts to bring external asset (BTC) onto Ethereum, But it's design limits it's potential to only a one-way relay from BTC to Ethereum, and the costs of relays and lack of proper incentive has rendered the project quiet.

### Design Goals
Our goals for the design of the bridge are:

  - **Multi-Chain two-way transfer support**: The bridge should support two-way transfer of assets between EVM and non-EVM based chains.

  - **Extensible**: The bridge should initially be compatible for EVM and BTC based chains, and extensible for other types such as DAG, EOS, etc.

  - **Standardized Interface**: All implementations of the bridge should follow a standardized interface for easy integration purposes.

  - **Permission-less**: The bridge should function under a non-authority based security model.

## Bridge Design

### Terminologies

*Home* - The chain where we are transferring external assets onto in the form of pegged tokens. Home is assumed to be EVM based (e.g Ethereum).

*Foreign* - The chain where we are transferring assets from and onto home. Foreign can be EVM or non-EVM based (e.g Bitcoin).

*Bridge* - The system that helps to perform message relay, verification, and hold/release of assets between users on home and foreign.

*Validator* - Role that runs bridge client. Similar to miners of traditional PoW chains.

## Overview

The bridge is a two way pegging mechanism that works through a rotation validator set and two bridge contracts (“contract” here refers to an executable piece of logic in the context of its residing chain). The bridge contracts resides on the home and foreign chains, hereon referred to as home-bridge and foreign-bridge.

The bridge contracts must be able to accepts and lock funds, verifies cryptographic signature of incoming cross-chain transfer transactions, and release token to user address on successful transfers. The relay of messages between the two bridges on different chains happen in a byzantine fault tolerant way by the bridge validators

Example Foreign -> Home Transfer Flow:
1. User *U* deposits some amount *T* of coin *Cforeign* to the foreign-bridge at address *Bforeign*. The transaction contains metadata for validators to relay the transfer:
  - *Cforeign* - coin address on foreign
  - *T* - transfer amount
  - *R* - recipient address on home
2. Validator queries and find a new incoming transaction on the address *Bforeign* on foreign chain.
3. Validators (1 to N) send message to the home bridge at address *Bhome* to relay the transfer with the following parameters:
  - *Cforeign* - coin address on foreign
  - *R* - recipient address on home.
  - *T* - transfer amount
  - *TX* - hash of transfer transaction to Bforeign
  - *Sig* - validator signature of this message
4. The home bridge receives the incoming message, verifies validator signature and keeps track of the signatures collected from validators. When more than N/2 validator signatures are collected for a given *TX* then *T* amount of *Chome* (pegged home representation of *Cforeign*) is released from *Bhome* to *R*.

## Validator Selection Criteria
Through the choice of a BFT consensus mechanism with validators formed from a set of stakeholders determined by depositing stakes in the bridge staking contracts, we are able to get a secure consensus with an infrequently changing and modest number of validators.

The staking contract resides on the home chain. Any validators that deposits the stake will become a validator in the next block. Similarly any valiator that withdraws from the contract will have their valdiator status revoked in the following block.

## Consensus
Any transfer requests with approval from *N/2 + 1* validators are finalized. This process continues indefinitely unless *N/2* or more validators become unresponsive, in which case the bridge halts.

## Security Assumptions
The bridge is secure as long as the attacker controls less than 51% of the validator pool.

## Rewarding and Slashing
Validators are incentivized to stay online and contribute to the bridge's security by relaying transfer requests for transfer fee rewards. For each relayed transfer message, the relaying validator earn a portion of the corresponding transfer fee. At the same time, to prevent validators from going offline without revoking their validator status (i.e withdraw their stake), a slashing mechanism is put in place. If a validator fails to perform the relay for a transfer message, he will get slashed a portion of his deposited stake.

- how to proof validator missing tx

## Home Bridge Implementation
The home bridge will be written in Solidity and ran on EVM, the implemention of which would closely ressemble that of Parity Bridge <here>.

The home bridge will follow the following interface:
- `transferToForeign(bytes homeTokenAddress, address recipient, uint amount)`
- `TransferToForeign(...)`
- `TransferFromForeign(...)`

The HomeToken (pegged token used on home chain to represent foreign token) will be a Mintable and Burnable token <openzeppelin>. This allows the token to be burned on transfer outs (to foreign), and minting of new tokens to recipient on transfer in (from foreign), keeping the total balance of the token supply static.

- bridge contracts

## Foreign Bridge Implementation for EVM Based Chains
The Foreign bridge can be implemented via smart contracts in Solidity. Using EVENTs, validators can efficiently be notified of incoming transfer requests from both side of the bridge. Messages relayed by the validators into the bridge contracts are signed with elliptic curve digital signature (ECDSA) and validated on-chain using *ecrecover*.

In this model, bridge validator nodes would have to do little other than listen for events, sign messages and send transactions bridge contracts. To receive the events from and get transactions actually routed onto the foreign and home chains, we assume either validators themselves would also reside on the respective networks (i.e running their own full nodes) or, utilize public node services (such as Infura [8] for the Ethereum network). The latter while being a more lightweight method, involves a trust factor in the public node provider.

## Foreign Bridge Implementation for BTC Based Chains
The challenge with Bitcoin is how the deposits can be securely controlled from a rotating validator set. Unlike Ethereum which is able to make arbitrary decisions based upon combinations of signatures, Bitcoin is substantially more limited, with most clients accepting only multisignature transactions with a maximum of 3 parties. Extending this to tens, or indeed thousands as might ultimately be desired, it is impossible under the current protocol. One option is to alter the Bitcoin protocol to enable such functionality, however so-called “hard forks” in the Bitcoin world are difficult to arrange judging by recent attempts. Another alternative is to use threshold signatures, cryptographic schemes to allow a singly identifiable public key to be effectively controlled by multiple secret “parts”, some or all of which must be utilised to create a valid signature. Unfortunately, threshold signatures compatible with Bitcoin’s ECDSA are computationally expensive to create and of polynomial complexity. Other schemes such a Schnorr signatures provide far lower costs, however the timeline on which they may be introduced into the Bitcoin protocol is uncertain.

Since the ultimate security of the deposits rests with a number of bonded validators, one other option is to reduce the multi-signature key-holders to only a heavily bonded subset of the total validators such that threshold signatures become feasible (or, at worst, Bitcoin’s native multi-signature is possible). We can achieve this using a M-of-N P2SH multisignature address according to BIP-13 [9]. The Bitcoin reference implementation has validations rules limiting the P2SH redeem script to be at most 520 bytes. The redeem script is of the format:

<p style="text-align: center;">*[M pubkey-1 pubkey-2 ... pubkey-N OP_CHECKMULTISIG]*</p>

It follows that the length of all public keys together plus the number of public keys must not be over 517 bytes. For compressed public keys, this means up to N=15.

This of course reduces the total amount of bonds that could be deducted in reparations should the validators behave illegally, however this is a graceful degradation, simply setting an upper limit of the amount of funds that can securely run between the two networks (or indeed, on the % losses should an attack from the validators succeed).

## Foreign Bridge Implementation for EOS/DAGs
- To be continued
