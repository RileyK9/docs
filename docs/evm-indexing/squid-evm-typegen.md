---
sidebar_position: 35
description: Type-safe EVM tx, log and state data
---

# EVM typegen

**Since `@subsquid/evm-typegen@2.0.0`**

The `squid-evm-typegen(1)` tool generates TypeScript facades for EVM transactions, logs and `eth_call` queries.

The generated facades are assumed to be used by squids indexing EVM data.

The tool takes a JSON ABIs as an input. Those can be specified in three ways:

1. as a plain JSON file(s):

   ```bash
   npx squid-evm-typegen src/abi erc20.json
   ```
   If you use this option, you can also place your JSON ABIs to the `abi` folder and run.
   ```bash
   sqd typegen
   ```
   Script is available in all EVM templates.

2. as a contract address (to fetch the ABI from Etherscan API). Once can pass multiple addresses at once.

   ```bash
   npx squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413
   ```

3. as an arbitrary URL:

   ```bash
   npx squid-evm-typegen src/abi https://example.com/erc721.json
   ```

In all cases typegen will use `basename` of the ABI as the root name for the generated files. You can change the basename of generated files using the fragment `(#)` suffix.

```bash
squid-evm-typegen src/abi 0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413#my-contract-name
```

**Arguments:**

|                        |                                                           |
|------------------------|-----------------------------------------------------------|
|  `output-dir`          | output directory for generated definitions                |
|  `abi`                 | A contract address, an URL or a local path to the ABI file. Accepts multiple contracts. |


**Options:**

|                           |                                                          |
|---------------------------|----------------------------------------------------------| 
|  `--multicall`            | generate a facade for the MakerDAO multicall contract. May significantly improve the performance of contract state calls by batching RPC requests (see below)   |
|  `--etherscan-api <url>`  | etherscan API to fetch contract ABI by a known address. By default, `https://api.etherscan.io/`   |
|  `--clean`                | delete output directory before the run                   |
|  `-h, --help`             | display help for command                                 |


## Usage

### Events 

The EVM log data is provided by the `event` object with wrappers for each event defined in the ABI.

**Example**

Subscribe to two topics:

```ts
// generated by evm-typegen
import { events } from "./abi/weth";

const processor = new EvmBatchProcessor()
  .setDataSource({
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog([
    '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'
  ], {
    filter: [[
      events.Deposit.topic, events.Withdrawal.topic
    ]],
    data: {
      evmLog: {
          topics: true,
          data: true,
      },
    } as const,
  })
```

Decode event data:
```ts
processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if (i.kind === 'evmLog' && i.evmLog.topics[0] == events.Deposit.topic) {
        // type-safe decoding of the Deposit event data
        const amt = events.Deposit.decode(i.evmLog).wad
      }
    }
});
```

### Transactions

Similar to `events`, transaction access is provided by the `functions` object for each contract method defined in the ABI. 

### Contract state calls

The typegen creates a wrapper `Contract` class for each contract. Create a `Contract` instance using the processor context and the block height at which the state should be queried.

**Example**

```ts
// generated by evm-typegen
import { Contract } from "./abi/weth";

let contractAddress = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2'.toLowerCase()

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    // call totalSupply() at the current block
    let blockHeader = c.header
    let supply = await (new Contract(ctx, blockHeader, contractAddress).totalSupply())
  }
});
```

### Batching contract state calls using the Multicall contract

Use the `--multicall` flag to generate the `Multicall` facade class for the [MakerDAO Multicall contract](https://github.com/makerdao/multicall). 
See [Batch state queries](/evm-indexing/query-state/#batch-state-queries) for more context.

The `Multicall` facade exposes the method
```ts
tryAggregate<Args extends any[], R>(
    func: Func<Args, {}, R>,
    calls: [address: string, args: Args][],
    paging?: number
  ): Promise<MulticallResult<R>[]>
```
The arguments are as follows:
- `func`: the contract function to be called
- `calls`: an array of tuples `[contractAddress: string, args]`. Each specified contract will be called with the specified arguments.
- `paging` an (optional) argument for the maximal number of calls to be batched into a single JSON PRC request. Note that large page sizes may cause timeouts.

A typical usage is as follows:
```ts
// generated by evm-typegen
import { functions } from "./abi/mycontract";
import { Multicall } from "./abi/multicall";

const MY_CONTRACT='0xac5c7493036de60e63eb81c5e9a440b42f47ebf5'

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
        // some logic
    }
  }
  const lastBlock = ctx.blocks[ctx.blocks.length - 1];
  // Multicall address for Ethereum is 0x5ba1e12693dc8f9c48aad8770482f4739beed696
  const multicall = new Multicall(ctx, lastBlock, '0x5ba1e12693dc8f9c48aad8770482f4739beed696')
  // call MY_CONTRACT.myContractMethod("foo") and MY_CONTRACT.myContractMethod("bar")
  const args = ["foo", "bar"]
  const results = await multicall.tryAggregate(functions.myContractMethod, args.map(a => [MY_CONTRACT, a]) as [string, any[]], 100);

  results.forEach((res, i) => {
    if (res.success) {
      ctx.log.info(`Result for argument ${args[i]} is ${res.value}`);
    }
  }) 
});
```

## Migration from `evm-typegen@1.x`

* In `evm-typegen@2.0`, the `--abi`, and `--output` flags are replaced by the positional arguments. For example, the command
  ```sh
  npx squid-evm-typegen --abi src/abi/ERC721.json --output src/abi/erc721.ts
  ```
  is replaced with
  ```sh
  npx squid-evm-typegen src/abi src/abi/ERC721.json
  ```

* The `events` object generated by `evm-typegen@2.0` exposes events by name, not by the full signature. For example,
  ```ts
  events['UpdatedGravatar(uint256,address,string,string)'].decode(evmLog)
  ```
  should be replaced with a more readable
  ```ts
  events.UpdatedGravatar.decode(evmLog)
  ```
