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
It leads to create new transaction types for token creation, transfer and meta-data updates.

## Motivation

This concept is born on Ethereum with the [EIP 721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) where the discussion was: `How can we standardize smart-contracts implementing non-fungible tokens, from the representation and transfer point of views ?`. The main goal was to ease tool creation (like exchanges) wherein multiple  NFT classes could be traded, without the need to adapt to each token specificities. 

Now NFTs are very popular on blockchain platforms, [Ethereum](https://opensea.io/), [Qtum](https://github.com/qtumproject/QRC721Token), [Stellar](https://github.com/future-tense/stellar-nft), [Neo](https://github.com/Splyse/neo-nft-template), [EOS](https://github.com/unicoeos/eosio.nft).

This AIP differs from EIP 721, in the way that we want framework users to be able to configure NFT class, create, update and transfer on their custom chain, without editing core code, but simply with configuration files. It's continuity of `blockchain in one click` baseline.

## Specifications

Here are the terms used in this document:
- **NFT**: stands for `non-fungible token`
- **NFT class**: like in *[OOP](https://en.wikipedia.org/wiki/Object-oriented_programming)*, it defines all tokens properties, characteristics and behaviour (like distribution model, type of data attached to a token, visibility, price,...).
- **NFT**: it's a unit, an instance of the NFT class. It's an indivisible and unique asset, verifying all NFT class definitions. 
- **mint**: token creation/instantiation process.

*Note: vocabulary is arbitrary and can be updated with discussions*

In this part, we focus on specifications for simple NFT class, allowing mint (First-in-first-served and free), transfer and properties value updates. 
Then, a non-exhaustive list of token features we have to keep in mind during the design process are described. 

### Transactions 

In order to be able to handle NFTs, at least two new transactions must be defined: 

#### Transfer

- `mint` or `transfer` token ownership
- payload:
    - token id - positive integer (`bignum`)
    - *(optional)* recipient id - address
- if `recipientId` is set, the transaction updates the owner of the token identified by `tokenId`
- else the owner of the given `tokenId` is the transaction sender
- transaction fails if the sender is not the token owner.
- Notice here that token id is computed off-chain. It's a design choice easing the minting process. The same process is used in popular NFTs on other chains (like Cryptokitties on Ethereum).

**Payload**

| Description                    | Size (bytes) | Example                                      |
| ------------------------------ | ------------ | :------------------------------------------- |
| type                           | 1            | 0x09                                         |
| token id                       | 8            | 0x0000000000000001                           |
| transfer type                  | 1            | 0x00 (mint) or 0x01 (transfer)               |
| recipient address *(optional)* | 21           | 0x171dfc69b54c7fe901e91d5a9ab78388645e2427ea |

- `token id` size (8 bytes) is an arbitrary choice. The idea was to limit transaction size (the same way than `delegate` username is limited to 20 characters). As a consequence, the maximum number of mintable NFTs is 2^64 units. Which seems huge, but really depends on use-case. 
- `transfer type` payload is only used by de-serializer to know if following 21 bytes must be read or not. This choice has been made to limit new transaction types. As an alternative, we could simply split transaction in two types `mint` and `transfer`.
- payload size is between 9 and 30 bytes ([type is a mandatory field of transaction header ](https://github.com/ArkEcosystem/AIPs/blob/master/AIPS/aip-11.md))

#### Update

- update token meta-data(s)
- payload:
    - token id
    - map of `propertyName` / `propertyValue` 
- transaction fails if token owner differs from sender

**Payload**

| Description           | Size (bytes) | Example                                                            |
| --------------------- | ------------ | :----------------------------------------------------------------- |
| type                  | 1            | 0x0A                                                               |
| token id              | 8            | 0x0000000000000001                                                 |
| properties length     | 1            | 0x01                                                               |
| property N key length | 1            | 0x3F                                                               |
| property N key (utf-8)| 1-255        | 0x70726f706572747931                                               |
| property N value      | 32           | 0x3C9683017F9E4BF33D0FBEDD26BF143FD72DE9B9DD145441B75F0604047EA28E |

- `properties length` is the number of updated properties. It must be positive. Size of the payload has been limited to 1 byte. As a consequence, the maximum number of updatable properties in a single transaction is (2^8)-1=255 properties. 
- `property key` is encoded in utf8 and can contain a maximum of 255 characters. 
- `property value` must be an output of the sha-256 function. It's a design choice used to limit the size of token meta-data. We don't want blockchain to store a lot of crappy data, but prints of these crappy data stored elsewhere.
- payload size is between 43 and 73449 bytes (255 properties with 255 characters key each).



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

We need to store tokens properties values in the database in order to be accessible at any time through API. 
So maybe add a new repository and queries in `core-database-postgres`.

### API

We need new routes fetching NFTs from current chain state. 
Here are some new routes:

- [GET] `/nfts/`: to index all created NFTs
- [GET] `/nfts/{id}`: to show NFT instance with given identifier and all its properties

Existing routes:
- [GET] `/wallets/{id}`: must be updated to return `list of owned tokens id`
- [POST] `/transactions/`: is used to submit a new NFT transaction (transfer or update).

### Transaction validation 

In order to be able to verify `mint` transactions, a validator must look into current chain state. Indeed, one of the core principles of minting is that you can't duplicate tokens or create a token with the same identifier than an already owned one. So, if given token id is already affected to an existing wallet, the transaction must be rejected. 

There are two validation processes: `transaction pool` and `block processor`. Each of them occurs at different steps. Respectively at new transaction received on a node and new block received. Each of these processes validates transactions applying them on two distinct wallet managers (respectively `core-transaction-pool/src/pool-wallet-manager` and `core-database/src/wallet-manager`).

By design `transaction handlers` in `crypto` package can't access to other wallets properties (other wallets than sender), we have to add a specific rule for our NFT transactions in `wallet managers` like:

```typescript
if ( isNFTMintTransaction && isTokenOwned ) {
    return new Error("reject")
}           
```

These modifications may be outdated due to works on [aip-29 - Generic Transaction Interface](https://github.com/ArkEcosystem/AIPs/blob/master/AIPS/aip-29.md). 

### NFT dedicated plugin

In order to isolate NFT support feature, a new core plugin has been created, dedicated to NFT management.
This plugin has many roles:
- define models
- parse configuration
- build token logic and validation rules from configuration
- keep and manage the state of NFTs at runtime
- ...

### Advanced token features

Here is a non-exhaustive list of behaviours and characteristics token class could specify, to keep in mind during the design process. Some items could have a dedicated AIP in the future. 

- advanced token distribution 
    - complex mint process, 
    - dynamic/static price, 
    - limit available tokens, 
    - auction, 
    - renewable ownership,
    - ... 
- security mechanisms with roles controlling token life-cycle (
        - unusable, 
        - kill, 
        - forced release,
        - ...
- multiple NFT class on the same chain
- composability (NFT linked to each other)
- ... 

## Reference Implementation

I'm working on [`@unik-name`](https://www.unik-name.com/) solution at [Spacelephant](https://www.spacelephant.org/) and our product is based on NFTs. 
So, in our lab, we've developed a proof-of-concept of Ark's NFT. 

Project sources are available [here](https://github.com/spacelephantlabs/core).

**What has been done:**

- 2 new transaction types 
    - constants
    - builders
    - validators
    - (de)serializers
    - handlers
- a dedicated package (`core-nft`) handling NFT class configuration and management. 
- API routes to access NFTs properties
- a very simple CLI to build NFT transactions for a demo.
- some other packages adaptations

**Future works:**

- fix existing bugs
- persist in database
- estimate and set default fees amount. How?
- write some tests 

