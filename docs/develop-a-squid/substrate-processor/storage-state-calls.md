---
sidebar_position: 41
description: >-
   Query the storage and access the historical state with gRPC requests to the node
title: State queries
---

# Storage calls and state queries

It is sometimes impossible to extract the required data with only event and call data without querying the runtime state.
The context exposes a lightweight gRPC client to the chain node accessible via `ctx._chain`. 
It exposes low-lever method for accessing the storage. However, the recommended way to query the storage is by generating type-safe access classes with [Substrate typegen](/develop-a-squid/typegen). 

To enable the gRPC client, **one must provide a `chain` data source to the processor**:

```typescipt
const processor = new SubstrateBatchProcessor()
    .setDataSource({
        archive: lookupArchive("kusama", {release: "FireSquid"})
        chain: 'wss://kusama-rpc.polkadot.io'
    })
```

:::tip
We recommend using private endpoints for better performance and stability of your squids. A standard way is to use environement variables and set them via [secrets](/deploy-squid/env-variables#secrets) when deploying to Aquarium.
:::

## Type-safe storage access with typegen

The Substrate typegen generates access classes in `src/types/storage.ts` which take into account the historical runtime upgrades by exposing versioned getters (like `getAsVXXX()`). The generated access methods support both single key and batch queries. 

Note that the generated getters **always query the historical block height of the "current" block derived the context**. This is the recommended way to access the storage.

To generate the storage classes with typegen:

* Set an archive GraphQL endpoint to `specVersions`. For a list of public archives, check the [Aquarium page](https://app.subsquid.io/aquarium/archives) and pick the data source URL at the archive page.
* List the fully qualified names of the storage items to the `storage` section of the typegen config. The format is `${PalleteName}.${KeyName}`.
* Rerun the typegen with

```ts
make typegen
```

Here's an example of `typegen.json` for generating an access class for `Balances.Reserves`:

```json title=typegen.json
{
  "outDir": "src/types",
  "specVersions": "https://kusama.archive.subsquid.io/graphql", 
  "storage": [
    "Balances.Account"
  ]
}
```
To generate all the available storage calls, set `"storage": true`.


Inspect the generated storage access classes in `src/types/storage.ts`. It should look similar to this:

```typescript title=src/types/storage.ts
/**
 * This code is generated by typegen
 */
export class BalancesAccountStorage {
  //.. constructuors

  // get a single account data
  async getAsV1050(key: Uint8Array): Promise<v1050.AccountData> {
    assert(this.isV1050)
    return this._chain.getStorage(this.blockHash, 'Balances', 'Account', key)
  }

  // get account data for multiple accounts in a single batch -- speeds up processing a lot 
  async getManyAsV1050(keys: Uint8Array[]): Promise<(v1050.AccountData)[]> {
    assert(this.isV1050)
    return this._chain.queryStorage(this.blockHash, 'Balances', 'Account', keys.map(k => [k]))
  }

}
```

The generated access class provide methods for:

- accessing a single storage item with `getAsXXX()`
- accessing multiple storage items in a batch call with `getManyAsXXX()`
- accessing all storage keys with a given prefix with `getKeys()` (supports paged output) (experimental)
- accessing all storage key-value pairs at a given key prefix with `getPairs()` (support paged output) (experimental)
- accessing all values at a given key prefix with `getAll()` (experimental)


## Access Storage items within a handler

As previously mentioned, the storage items are always retrieved at the "current" block height of `StorageContext`. To instantiate a storage access class, pass the current block hash wrapped as `{ hash: string }` as the second constructor argument.

### Example

```typescript title=processor.ts
import {BalancesAccountStorage} from "./types/storage"

const processor = new SubstrateBatchProcessor()
    .setBatchSize(50)
    .setBlockRange({ from: 9_000_000, to: 9_100_000})
    .setDataSource({
        // Lookup archive by the network name in the Subsquid registry
        archive: lookupArchive("kusama", {release: "FireSquid"}),
        // use a private endpoint for production
        chain: 'wss://kusama-rpc.polkadot.io'
    }).includeAllBlocks({ from: 9_000_000, to: 9_100_000})


processor.run(new TypeormDatabase(), async ctx => {
    for (const block of ctx.blocks) { 
      // block.header is of type { hash: string }
      let storage = new BalancesAccountStorage(ctx, block.header)
      let aliceAddress = ss58.decode('5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY').bytes
      let aliceBalance = (await storage.getAsV1050(aliceAddress)).free
      ctx.log.info(`Alice free account balance at block ${block.header.height}: ${aliceBalance.toString()}`)
    }
})
```