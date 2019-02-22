---
  AIP: 33
  Title: Non fungible token management
  Authors: Guillaume Nicolas <guillaume.nicolas@spacelephant.org>
  Status: Draft
  Discussions-To: https://github.com/arkecosystem/AIPS/issues
  Type: Standards Track
  Category: Core
  Created: 2019-02-11
  Last Update: 2019-02-11
---

## Abstract

While fungible tokens can be exchanged with each other without loss of value, non-fungible tokens are unique pieces with value based on their properties and scarcity.
It's a new way to digitize assets ownership like sports cards, game stuff, art pieces, houses, identities, ... 

This AIP proposes to add a new feature for Ark framework: the non-fungible token support. 
It leads to create new transaction types for token creation, transfer and properties values updates.

## Motivation

This concept is born on Ethereum with the [EIP 721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) where the discussion was: `How can we standardize smart-contracts implementing non-fungible tokens, from the representation and transfer point of views ?`. The main goal was to ease tool creation (like exchanges) wherein multiple  NFT classes could be traded, without the need to adapt to each token specificities. 

Now NFTs are very popular on blockchain platforms, [Ethereum](https://opensea.io/), [Qtum](https://github.com/qtumproject/QRC721Token), [Stellar](https://github.com/future-tense/stellar-nft), [Neo](https://github.com/Splyse/neo-nft-template), [EOS](https://github.com/unicoeos/eosio.nft),...

This AIP differs from EIP 721, in the way that we want framework users to be able to configure NFT class, create, update and transfer on their custom chain, without editing core code, but simply with configuration files. It's continuity of `blockchain in one click` baseline.

## Specifications

Here are the terms used in this document:
- **NFT**: stands for `non-fungible token`
- **NFT class**: like in *[OOP](https://en.wikipedia.org/wiki/Object-oriented_programming)*, it defines all tokens properties, characteristics and behaviours (like distribution model, type of data attached to a token, visibility, price,...). They can't be updated by user transactions. They're used to identify NFT classes (supposing multiple ones co-exist on the same chain).
- **NFT**: it's a unit, an instance of the NFT class. It's an indivisible and unique asset, verifying all NFT class properties. 
- **mint**: token creation/instantiation process.

*Note: vocabulary is arbitrary and can be updated with discussions*

In this part, we focus on specifications for a simple NFT class, allowing mint (first-in-first-served and free), ownership transfer and properties values updates. We assume that NFT class is living alone on a dedicated chain (*i.e* the chain does not handle multiple ones). 
Then, we describe a non-exhaustive list of token features we have to keep in mind during the design process. 

### Transactions 

The advantage of NFTs is that you cryptographically own a token. As a consequence, a transaction is valid only if sender is the token owner. 

In order to be able to handle NFTs, at least two new transactions must be defined (types `9` and `10`): 

#### > Transfer (type 9)

Because under the hood they are very similar, `NFT Transfer` transaction type gathers two behaviors:
- **token mint:** seal existence of unique token id and its owner.
- **token ownership transfer:** seal the new owner of a token.

**Transaction dedicated parameters**

- token id: positive integer (`bignum`)
- *(optional)* recipient id: address (`string`)

If a `NFT Transfer` transaction is created without `recipient id` field, it's a `token mint` transaction. Else, it's a `token ownership transfer` transaction.

A `token mint` transaction is valid as soon as given token id has not been sealed previously. Then, owner of this new token is the transaction sender.

A `token ownership transfer` transaction is valid if:
- token identified by given token id has been previously sealed
- transaction sender is the owner of given token id

**Transaction payload**

| Description                    | Size (bytes) | Example                                                            |
| ------------------------------ | ------------ | :----------------------------------------------------------------- |
| type                           | 1            | 0x09                                                               |
| token id                       | 32           | 0x0000000000000000000000000000000000000000000000000000000000000001 |
| transfer type                  | 1            | 0x00 (mint) or 0x01 (transfer)                                     |
| recipient address *(optional)* | 21           | 0x171dfc69b54c7fe901e91d5a9ab78388645e2427ea                       |

Payload size is between **33** and **54 bytes** ([type is a mandatory field of **transaction header** ](https://github.com/ArkEcosystem/AIPs/blob/master/AIPS/aip-11.md))

**Design choices**

1. `token id` size (32 bytes) is an arbitrary choice inherited from [ERC721 specifications](https://github.com/ethereum/EIPs/blame/master/EIPS/eip-721.md#L270-L274) defining NFT ids as the largest unsigned integer of the EVM: `uint256` (256 bits).
2. `transfer type` payload is only used by de-serializer to know if following 21 bytes must be read or not. This choice has been made to limit the number of new transaction types. As an alternative, we could simply split transaction in two types `mint` and `transfer`.
3. `Transfer` type is only used to create a new blank token or update ownership property. It means that we need at least two transactions to create a new token with user-defined properties.
4. Token id is computed off-chain. It's a design choice easing the minting process. The same process is used in popular NFTs on other chains (like Cryptokitties on Ethereum).
5. It's not possible to mint a token for someone else. Two transactions are needed (`mint` then `transfer`). It's a questionable design choice. 

#### > Update (type 10)

This transaction type is used to update token properties values.

**Transaction dedicated parameters**

- token id: which token to update
- map of `propertyName` / `propertyValue`: list of new token properties value. 
    
Transaction is valid if:
- token identified by given token id has been previously sealed
- transaction sender is the owner of given token id

**Transaction payload**

| Description           | Size (bytes) | Example                                                            |
| --------------------- | ------------ | :----------------------------------------------------------------- |
| type                  | 1            | 0x0A                                                               |
| token id              | 32           | 0x0000000000000000000000000000000000000000000000000000000000000001 |
| properties length     | 1            | 0x01                                                               |
| property N key length | 1            | 0x3F                                                               |
| property N key (utf-8)| 1-255        | 0x70726f706572747931                                               |
| property N value      | 32           | 0x3C9683017F9E4BF33D0FBEDD26BF143FD72DE9B9DD145441B75F0604047EA28E |

Payload size is between **67** (single property with 1 character key) and **73219 bytes** (255 properties with 255 characters key each).

**Design choices**

1. `properties length` is the number of updated properties. It must be positive. Size of the payload has been limited to 1 byte. As a consequence, the maximum number of updatable properties in a single transaction is (2^8)-1=255 properties. 
2. `property key` is encoded in utf8 and can contain a maximum of 255 characters. 
3. `property value` must be an output of the sha-256 function. It's a design choice used to limit the size of token properties value. We don't want blockchain to store a lot of crappy data, but prints of these crappy data stored elsewhere.
4. No controls are made on-chain on new properties value.
5. `Token id` and `token owners` can't be updated with this transaction type. `Token id` can't be updated at all.`Token owner` can be updated with a `transfer` transaction.

### Model

A new model must be created to represent an NFT instance:

```typescript
class NFT {
    public id: Bignum;
    public properties: { [_: string]: string };
    constructor(id: Bignum) {...}
    public updateProperty(key: string, value: string) {...}
}
```

Here we can see a design choice: `NFT.id` is of `Bignum` type (a big number). Identify tokens with numbers is a choice based on ERC721 specifications defining NFT ids as the largest unsigned integer of the EVM: `uint256` (256 bits).

Existing `Wallet` model must be extended to reference `owned tokens id`:

```typescript
class Wallet {
    [...]
    public tokens: Bignum[];
    
    [...]
}
```

### Database 

We need to store tokens properties values in the database in order be accessible at any time through API. 
A new repository and queries in `core-database-postgres` are required.

### API

We need new routes fetching NFTs from current chain state. 
Here are some new routes:

- [GET] `/nfts/`: to index all sealed NFTs. This route follows the same logic as other root routes (like `/transactions` or `/blocks`). It's paginated and returns all properties for each NFT.
- [GET] `/nfts/{id}`: to show NFT instance with given identifier and all its properties

Existing routes:
- [GET] `/wallets/{id}`: must be updated to return `list of owned tokens id`
- [POST] `/transactions/`: is used to submit a new NFT transaction (transfer or update).

### Transaction validation 

In order to be able to verify `mint` transactions, a validator must look into current chain state. Indeed, one of the core principles of minting is that you can't duplicate tokens or create a token with the same identifier as an already owned one. So, if given token id is already affected to an existing wallet, the transaction must be rejected. 

There are two validation processes: `transaction pool` and `block processor`. Each of them occurs at different steps. Respectively at new transaction received on a node and new block received. Each of these processes validates transactions applying them on two distinct wallet managers (respectively `core-transaction-pool/src/pool-wallet-manager` and `core-database/src/wallet-manager`).

By design `transaction handlers` in `crypto` package can't access to other wallets properties (other wallets than sender), we have to add a specific rule for our NFT transactions in `wallet managers` like:

```typescript
if ( isNFTMintTransaction && isTokenOwned ) {
    return new Error("reject")
}           
```

*These modifications may be outdated due to works on [aip-29 - Generic Transaction Interface](https://github.com/ArkEcosystem/AIPs/blob/master/AIPS/aip-29.md).*

### NFT dedicated plugin

In order to isolate NFT support feature, a new core plugin has been created, dedicated to NFT management.
This plugin has many roles:
- define models
- parse configuration
- build token logic and validation rules from configuration
- keep and manage the state of NFTs at runtime
- ...

### Advanced token features

Here is a non-exhaustive list of behaviours and characteristics token class could specify, to keep in mind during the design process. They're inspired by existing ERC721 tokens. Some items could have a dedicated AIP in the future. 

- advanced token distribution 
  - complex mint process, 
  - dynamic/static price, 
  - limit available tokens, 
  - auction, 
  - renewable ownership,
  - ... 
- security mechanisms with roles controlling token life-cycle
  - unusable, 
  - kill, 
  - forced release,
  - ...
- multiple NFT class on the same chain
- composability (NFT linked to each other)
- ... 

## Reference Implementation

I'm working on [`@unik-name`](https://www.unik-name.com/) solution at [Spacelephant](https://www.spacelephant.org/) and our product is based on NFTs. 
We've developed a proof-of-concept of Ark's NFT in our labs.

Project sources are available [on the dedicated repository](https://github.com/spacelephantlabs/ark-core_non-fungible-token).

**What has been done:**

- 2 new transaction types 
    - constants
    - builders
    - validators
    - (de)serializers
    - handlers
- a dedicated package (`core-nft`) handling NFT class configuration and management. 
- API routes to access NFTs properties
- update of `tester-cli` package to build `nft` transactions (mint and transfer).
- various packages adaptations. 

**Future works:**

- fix double transaction execution
- persist in database
- implement a way to revert update transactions.
- estimate and set default fees amount.
- rename /nft API to /nfts
- fix old tests.

See the [README of the project](https://github.com/spacelephantlabs/ark-core_non-fungible-token/blob/feat/nft/README.md) for an updated version of the todo/done tasks.
