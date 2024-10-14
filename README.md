## Deployed at

https://api.goldsky.com/api/public/project_cm0bmkexjt0n601ww47obci68/subgraphs/fungible/1.0.5/gn

## Prerequisites

Before you begin, ensure you have the following installed on your machine:

-   **Node.js** and **Yarn**
-   **Git**
-   **The Graph CLI**

Install The Graph CLI globally:


`npm install -g @graphprotocol/graph-cli` 

----------

## Step 1: Install Goldsky CLI and Log In

### Install Goldsky CLI



`curl https://goldsky.com | sh` 

### Log In to Goldsky

#### Create an API Key:

-   Go to your Goldsky project's **Settings** page.
-   Create an **API key**.

#### Log In via CLI:

In your terminal, run:

bash

Copy code

`goldsky login` 

Paste your API key when prompted.

#### Verify Installation:

Run the Goldsky CLI to ensure it's working:

bash

Copy code

`goldsky` 

----------

## Step 2: Set Up Your Subgraph

### Clone the Repository

Clone your fork of the subgraph repository:


```sh
git clone https://github.com/cdirks4/v3-subgraph.git
cd v3-subgraph 
```

### Install Dependencies

`yarn install` 

### Create `subgraph.yaml`

Create a `subgraph.yaml` file in the root directory of your project. Here's a base example tailored for the contract at `0x1E6678e0AF0Aa0220a01fCA979Fa3612e3E4e2ce` on `story-testnet`:


```specVersion: 0.0.4
description: Subgraph for the Uniswap V3-like contract on story-testnet.
repository: https://github.com/<your-username>/v3-subgraph
schema:
  file: ./schema.graphql
features:
  - nonFatalErrors
  - grafting

dataSources:
  - kind: ethereum/contract
    name: Factory
    network: story-testnet
    source:
      abi: Factory
      address: '0x1E6678e0AF0Aa0220a01fCA979Fa3612e3E4e2ce'
      startBlock: <START_BLOCK_NUMBER>
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      file: ./src/mappings/factory.ts
      entities:
        - Pool
        - Token
      abis:
        - name: Factory
          file: ./abis/Factory.json
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: PoolCreated(indexed address,indexed address,indexed uint24,int24,address)
          handler: handlePoolCreated

templates:
  - kind: ethereum/contract
    name: Pool
    network: story-testnet
    source:
      abi: Pool
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      file: ./src/mappings/pool/index.ts
      entities:
        - Pool
        - Token
      abis:
        - name: Pool
          file: ./abis/Pool.json
        - name: Factory
          file: ./abis/Factory.json
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: Initialize(uint160,int24)
          handler: handleInitialize
        - event: Swap(indexed address,indexed address,int256,int256,uint160,uint128,int24)
          handler: handleSwap
        - event: Mint(address,indexed address,indexed int24,indexed int24,uint128,uint256,uint256)
          handler: handleMint
        - event: Burn(indexed address,indexed int24,indexed int24,uint128,uint256,uint256)
          handler: handleBurn
        - event: Collect(indexed address,address,indexed int24,indexed int24,uint128,uint128)
          handler: handleCollect` 
```
#### Customize `subgraph.yaml`

-   **Replace `<START_BLOCK_NUMBER>`** with the block number where your contract was deployed.
-   **Update `repository`** with your repository URL.
-   **Ensure all ABIs referenced exist** in the `./abis` directory.
-   **Adjust `entities` and `eventHandlers`** as per your contract's specifications.

### Define the Schema

Create a `schema.graphql` file in the root directory. Define your entities based on the data you want to index. For example:


```type Pool @entity {
  id: ID!
  token0: Token!
  token1: Token!
  feeTier: BigInt!
  # Add other fields as necessary
}

type Token @entity {
  id: ID!
  symbol: String!
  name: String!
  decimals: BigInt!
  # Add other fields as necessary
}` 
```
### Implement Mappings

Mappings are TypeScript functions that transform Ethereum events into entity data. Implement the handlers specified in your `subgraph.yaml`.

#### Example: `src/mappings/factory.ts`
```
import { PoolCreated } from '../../generated/Factory/Factory';
import { Pool, Token } from '../../generated/schema';
import { Pool as PoolTemplate } from '../../generated/templates';
import { fetchTokenSymbol, fetchTokenName, fetchTokenDecimals } from '../utils/token';

export function handlePoolCreated(event: PoolCreated): void {
  // Load or create Token entities
  let token0 = Token.load(event.params.token0.toHexString());
  if (token0 == null) {
    token0 = new Token(event.params.token0.toHexString());
    // Set token0 fields
    token0.symbol = fetchTokenSymbol(event.params.token0);
    token0.name = fetchTokenName(event.params.token0);
    token0.decimals = fetchTokenDecimals(event.params.token0);
    token0.save();
  }

  let token1 = Token.load(event.params.token1.toHexString());
  if (token1 == null) {
    token1 = new Token(event.params.token1.toHexString());
    // Set token1 fields
    token1.symbol = fetchTokenSymbol(event.params.token1);
    token1.name = fetchTokenName(event.params.token1);
    token1.decimals = fetchTokenDecimals(event.params.token1);
    token1.save();
  }

  // Create a new Pool entity
  let pool = new Pool(event.params.pool.toHexString());
  pool.token0 = token0.id;
  pool.token1 = token1.id;
  pool.feeTier = event.params.fee;
  // Set other Pool fields as necessary
  pool.save();

  // Create a new Pool template instance
  // This allows indexing of events from the new Pool contract
  PoolTemplate.create(event.params.pool);
}` 
```
#### Example: `src/mappings/pool/index.ts`

```
`import { Swap, Mint, Burn, Collect, Initialize } from '../../../generated/templates/Pool/Pool';
import { Pool, Transaction } from '../../../generated/schema';

export function handleInitialize(event: Initialize): void {
  // Implement logic for Initialize event
}

export function handleSwap(event: Swap): void {
  // Implement logic for Swap event
}

export function handleMint(event: Mint): void {
  // Implement logic for Mint event
}

export function handleBurn(event: Burn): void {
  // Implement logic for Burn event
}

export function handleCollect(event: Collect): void {
  // Implement logic for Collect event
}` 
```
----------

## Step 3: Generate Code

Run the following command to generate TypeScript types from your schema and ABIs:



```sh 
graph codegen
```

This command generates code in the `generated/` directory.

----------

## Step 4: Deploy to Goldsky

### Deploy the Subgraph

Use the Goldsky CLI to deploy your subgraph:
	
```sh
goldsky subgraph deploy <your-subgraph-name>/<version>
 ```

-   Replace `<your-subgraph-name>` with a name for your subgraph.
-   Replace `<version>` with a version identifier, like `v1.0.0`.

### Monitor Deployment

-   Log in to your Goldsky dashboard.
-   Navigate to your project and monitor the indexing status.
-   Use the GraphQL playground provided by Goldsky to test queries.

----------

## Additional Resources

-   **Goldsky Documentation**: [https://docs.goldsky.com/](https://docs.goldsky.com/)
-   **The Graph Documentation**: [https://thegraph.com/docs/](https://thegraph.com/docs/)
-   **AssemblyScript Documentation**: [https://www.assemblyscript.org/](https://www.assemblyscript.org/)

----------

## Conclusion

By following this guide, you should be able to set up a subgraph, define the necessary schema and mappings, and deploy it to Goldsky. This will enable efficient querying of blockchain data for your applications.

