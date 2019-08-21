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

In order to be able to handle NFTs, at three new transactions must be defined (types `9` and `10` and `11`).

*Note*: types number are arbitrary choices. They'll be increased to prevent conflicts with future types released before nft transactions. 

### > Mint (type 11)

The first thing to do to get a NFT is to create and broadcast a `mint` transaction. 
This transaction is responsible of writing on chain that a given wallet is the first owner of a token identified by a never used value.

**Transaction dedicated parameters**
- token id: a 32 characters string
- *(optional)* a list of properties


#### > Transfer (type 9)

Because under the hood they are very similar, `NFT Transfer` transaction type gathers two behaviors:
- **token mint:** seal existence of unique token id and its owner.
- **token ownership transfer:** seal the new owner of a token.

**Transaction dedicated parameters**

- token id: a large number
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

1. `token id` size (32 bytes) is an arbitrary choice inherited from [ERC721 specifications](https://github.com/ethereum/EIPs/blame/master/EIPS/eip-721.md#L270-L274) identifying NFTs as the largest unsigned integer of the EVM: `uint256` (256 bits). It also allows a wide variety of applications because UUIDs and sha3 outputs are of this size. 
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

Payload size is between **67** (single property with 1 character key) and **73473 bytes** (255 properties with 255 characters key each).

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
    public id: string;
    public properties: { [_: string]: string };
    constructor(id: string) {...}
    public updateProperty(key: string, value: string) {...}
}
```

Existing `Wallet` model must be extended to reference `owned tokens id`:

```typescript
class Wallet {
    [...]
    public tokens: string[];
    
    [...]
}
```

### Events

In order to be compatible with existing applications handling NFTs (exchanges, wallets,... ), we must provide a simple way to detect updates on NFTs. 
Similarly to [ERC721](https://github.com/ethereum/EIPs/blame/master/EIPS/eip-721.md#L49-L54) proposal, we can broadcast events at several steps of NFTs life-cycle:

- token creation
- ownership update
- properties update

`Core-webhooks` is the module forwarding events to applications. 

It's listening `event-emitter` events, so new [blockchain events](https://github.com/ArkEcosystem/core/blob/master/packages/core-blockchain/src/blockchain.ts#L579-L604) must be added:

- `nft.created`
- `nft.transferred`
- `nft.updated`

### Database 

We need to store tokens and their properties values in the database because of scalability. Indeed, store nfts and their properties in memory (like wallets) could cause performance issues and memory bursting.

A new repository and queries in `core-database-postgres` are required.

### API

We need new routes fetching NFTs from current chain state. 
Here are some new routes:

- [GET] `/nfts/`: to index all sealed NFTs. This route follows the same logic as other root routes (like `/transactions` or `/blocks`). It's paginated and returns all properties for each NFT.
- [GET] `/nfts/{id}`: to show NFT instance with given identifier and all its properties
- [GET] `/nfts/{id}/properties`: to get all properties of a NFT (paginated)

Existing routes:
- [GET] `/wallets/{id}`: must be updated to return `list of owned tokens id`
- [POST] `/transactions/`: is used to submit a new NFT transaction (transfer or update).


### NFT dedicated plugin

In order to isolate NFT support feature, a new core plugin has been created, dedicated to NFT management.
This plugin has many roles:
- define models
- parse configuration
- build token logic and validation rules from configuration
- keep and manage the state of NFTs at runtime
- ...

### NFT configuration file

A NFT must be configurable through `network.json` file because its properties are specific to a network and must be the same for all nodes validating nft transactions.

A new property of `network` is created. Here is a configuration sample: 

```JSON
{
    "nft":{
        "name": "UNIK",
        "properties":{
            "a":{
                "constraints":[
                    "immutable",
                    {
                        "name":"type",
                        "parameters": {
                            "type": "number",
                            "min": 1,
                            "max": 2
                        }
                    }
                ]
            }
        }
    }
}
```

A NFT has a name and properties. In this configuration file, you could set constraints on a properties value.

A constraint would be :
- a name/identifier
- optional parameters to customize constraints depending on property.

As an example, we could have an immutability constraint (`immutable` as identifier) which causes a property to not be updatable after set.

As an other example, we could have more abstract constraints like `type` which causes a property value to be of a specific type (`number`, `string`,... ).
In that case, we could add constraints depending property type and value. In the example above, we set that property `a` must be a `number` between 1 and 2.

Constraint logic is implemented into `core-nft` plugin as a class which must implement the `Constraint` interface:

```typescript
interface ConstraintApplicationContext {
    propertyKey: string;
    propertyNewValue: string;
    transaction: ITransactionData;
}

interface Constraint {
    apply(context: ConstraintApplicationContext, parameters?: any): Promise<void>;
    name(): string;
}
```

Each constraint must set a name and implement a check on nft property update transaction validity. 
To check validity, a constraint has several information:
- a context 
    - the updated property key 
    - its new value 
    - and the overall transaction ( can be used to set complex rules ( based on sender, fees, or any other nft property update value ) )
- possible constraint parameters as described previously. 
- and they can do asynchronous operations like get property current value from database.

Bellow is an example of how `type` constraint could be implemented (voluntarily simplified):

```typescript

class TypeConstraint implements Constraint {
    public async apply(context: ConstraintApplicationContext, parameters?: any){
        const { type, min, max } = parameters;
        if( type === "number" && ( context.propertyNewValue < min || context.propertyNewValue > max )){
            throw new ConstraintError();
        }
    }

    public name() {
        return "type";
    }
}

```

Constraint application must be made inside the `canBeApplied` method of nft-update transaction handler. 

Current proposal makes nft property constraint relatively easy to customize : register a new custom constraint logic to constraint manager and reference it directly from `network.json` file.

***Note 1:*** `network.json` is validated before used using schemes. When adding custom constraints, you must update `network.json` scheme accordingly. 
***Note 2:*** a limitation exists due to current implementation of handlers in Ark. You can't apply constraint from `canBeApplied` method because it's synchronous.

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