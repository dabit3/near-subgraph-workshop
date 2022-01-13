## Building an NFT API on NEAR with The Graph

In this workshopp we'll build a subgraph for querying NTF data from the [Misfits](https://explorer.near.org/accounts/misfits.tenk.near) smart contract, implementing queries for fetching NFTs as well as their owners, and building relationships between them.

### Prerequisites

To be successful in this tutorial, you should have [Node.js](https://nodejs.org/en/) installed on your machine. These days, I recommend using either [nvm](https://github.com/nvm-sh/nvm) or [fnm](https://github.com/Schniz/fnm/blob/master/docs/commands.md) to manage Node.js versions.

### Creating the Graph project in The Graph dashboard

To get started, open The Graph [dashboard](https://thegraph.com/hosted-service/dashboard) and either sign in or create a new account.

Next, click on __Add Subgraph__ to create a new subgraph.

Configure your subgraph with the following properties:

- Subgraph Name - __Nearmisfits__
- Subtitle - __A subgraph for querying NFTs__
- Optional - Fill the description and GITHUB URL properties

Once the subgraph is created, we will initialize the subgraph locally using the Graph CLI.

### Initializing a new subgraph using the Graph CLI

Next, install the Graph CLI:

```sh
$ npm install -g @graphprotocol/graph-cli

# or

$ yarn global add @graphprotocol/graph-cli
```

Once the Graph CLI has been installed you can initialize a new subgraph with the Graph CLI `init` command.

There are two ways to initialize a new subgraph:

1 - From an example subgraph

```sh
$ graph init --from-example <GITHUB_USERNAME>/<SUBGRAPH_NAME> [<DIRECTORY>]
```

2 - From an existing smart contract

If you already have a smart contract deployed to Ethereum mainnet or one of the testnets, initializing a new subgraph from this contract is an easy way to get up and running.

```sh
$ graph init --from-contract <CONTRACT_ADDRESS> \
  [--network <ETHEREUM_NETWORK>] \
  [--abi <FILE>] \
  <GITHUB_USER>/<SUBGRAPH_NAME> [<DIRECTORY>]
```

In our case we'll be starting with the [Misfits NFT contract](https://explorer.near.org/accounts/misfits.tenk.near) so we can initialize from that contract address by passing in the contract address using the `--from-contract` flag:

```sh
$ graph init --from-contract misfits.tenk.near --contract-name Token

? Protocol › near
? Subgraph name › your-username/nearmisfits
? Directory to create the subgraph in › Foundationsubgraph
? Ethereum network › Mainnet
? NEAR network › near-mainnet
? Contract account › misfits.tenk.near
? Contract Name · Token
```

This command will generate a basic subgraph based off of the contract address passed in as the argument to `--from-contract`. By using this contract address, the CLI will initialize a few things in your project to get you started (including fetching the `abis` and saving them in the __abis__ directory).

The main configuration and definition for the subgraph lives in the __subgraph.yaml__ file. The subgraph codebase consists of a few files:

- __subgraph.yaml__: a YAML file containing the subgraph manifest
- __schema.graphql__: a GraphQL schema that defines what data is stored for your subgraph, and how to query it via GraphQL
- __AssemblyScript Mappings__: AssemblyScript code that translates from the event data in Ethereum to the entities defined in your schema (e.g. mapping.ts in this tutorial)

The entries in __subgraph.yaml__ that we will be working with are:

- `description` (optional): a human-readable description of what the subgraph is. This description is displayed by the Graph Explorer when the subgraph is deployed to the Hosted Service.
- `dataSources.source`: the address of the account that the subgraph sources.
- `dataSources.source.startBlock` (optional): the number of the block that the data source starts indexing from. In most cases we suggest using the block in which the contract was created.
- `dataSources.mapping.entities` : the entities that the data source writes to the store. The schema for each entity is defined in the the schema.graphql file.
- `dataSources.mapping.receiptHandlers`: lists the receipts this subgraph reacts to and the handlers in the mapping — __./src/mapping.ts__ in the example — that transform these events into entities in the store.

### Defining the entities

With The Graph, you define entity types in __schema.graphql__, and Graph Node will generate top level fields for querying single instances and collections of that entity type. Each type that should be an entity is required to be annotated with an `@entity` directive.

The entities / data we will be indexing are the `Token` and `User`. This way we can index the Tokens created by the users as well as the users themselves. We'll also enable full text search for searching by account name as well as the `kind` of Misfit NFT.

To do this, update __schema.graphql__ with the following code:

```graphql
type _Schema_
  @fulltext(
    name: "tokenSearch"
    language: en
    algorithm: rank
    include: [{ entity: "Token", fields: [{ name: "ownerId" }, { name: "kind" }] }]
  )

type Token @entity {
  id: ID!
  owner: User!
  ownerId: String!
  tokenId: String!
  image: String!
  metadata: String!
  kind: String!
  seed: Int!
}

type User @entity {
  id: ID!
  tokens: [Token!]! @derivedFrom(field: "owner")
}
```

### On Relationships via `@derivedFrom` (from the docs):

Reverse lookups can be defined on an entity through the `@derivedFrom` field. This creates a virtual field on the entity that may be queried but cannot be set manually through the mappings API. Rather, it is derived from the relationship defined on the other entity. For such relationships, it rarely makes sense to store both sides of the relationship, and both indexing and query performance will be better when only one side is stored and the other is derived.

For one-to-many relationships, the relationship should always be stored on the 'one' side, and the 'many' side should always be derived. Storing the relationship this way, rather than storing an array of entities on the 'many' side, will result in dramatically better performance for both indexing and querying the subgraph. In general, storing arrays of entities should be avoided as much as is practical.

Now that we have created the GraphQL schema for our app, we can generate the entities locally to start using in the `mappings` created by the CLI:

```sh
graph codegen
```

In order to make working smart contracts, events and entities easy and type-safe, the Graph CLI generates AssemblyScript types from a combination of the subgraph's GraphQL schema and the contract ABIs included in the data sources.

## Updating the subgraph with the entities and mappings

Now we can configure the __subgraph.yaml__ to use the entities that we have just created and configure their mappings.

To do so, first update the `dataSources.mapping.entities` field with the `User` and `Token` entities:

```yaml
entities:
  - Token
  - User
```

Next, update the configuration to add the startBlock and change the contract `address` to the [main proxy contract](https://etherscan.io/address/0x3B3ee1931Dc30C1957379FAc9aba94D1C48a5405) address:

```yaml
source:
  account: "misfits.tenk.near"
  startBlock: 53472065
```

Next, we'll want to enable a couple of features like __Full Text Search__ as well as the __IPFS API__.

To do this, add the following feature flags as top level configurations (after the s`chema` declaration):

```yaml
features:
  - ipfsOnEthereumContracts
  - fullTextSearch
```

Finally, update the `specVersion` to be 0.0.4:

```yaml
specVersion: 0.0.2
```

The final __subgraph.yaml__ should look like this:

```yaml
specVersion: 0.0.4
schema:
  file: ./schema.graphql
features:
  - ipfsOnEthereumContracts
  - fullTextSearch
dataSources:
  - kind: near
    name: Token
    network: near-mainnet
    source:
      account: "misfits.tenk.near"
      startBlock: 53472065
    mapping:
      apiVersion: 0.0.5
      language: wasm/assemblyscript
      entities:
        - Token
        - User
      receiptHandlers:
        - handler: handleReceipt
      file: ./src/mapping.ts

```

## Assemblyscript mappings

Next, open __src/mappings.ts__ to write the mappings that we defined in our subgraph subgraph `eventHandlers`.

Update the file with the following code:

```typescript
import { near, JSONValue, json, ipfs } from "@graphprotocol/graph-ts"
import { Token, User } from "../generated/schema"
import { log } from '@graphprotocol/graph-ts'

export function handleReceipt(
  receipt: near.ReceiptWithOutcome
): void {
  const actions = receipt.receipt.actions;
  for (let i = 0; i < actions.length; i++) {
    handleAction(actions[i], receipt)
  }
}

function handleAction(
  action: near.ActionValue,
  receiptWithOutcome: near.ReceiptWithOutcome
): void {
  if (action.kind != near.ActionKind.FUNCTION_CALL) {
    return;
  }
  const outcome = receiptWithOutcome.outcome;
  const functionCall = action.toFunctionCall();
  const ipfsHash = 'bafybeiew2l6admor2lx6vnfdaevuuenzgeyrpfle56yrgse4u6nnkwrfeu'
  const methodName = functionCall.methodName

  if (methodName == 'buy' || methodName == 'nft_mint_one') {
    for (let logIndex = 0; logIndex < outcome.logs.length; logIndex++) {
      let outcomeLog = outcome.logs[logIndex].toString();

      log.info('outcomeLog {}', [outcomeLog])

      let parsed = outcomeLog.replace('EVENT_JSON:', '')

      let jsonData = json.try_fromString(parsed)
      const jsonObject = jsonData.value.toObject()

      let eventData = jsonObject.get('data')
      if (eventData) {
        let eventArray:JSONValue[] = eventData.toArray()

        let data = eventArray[0].toObject()
        const tokenIds = data.get('token_ids')
        const owner_id = data.get('owner_id')
        if (!tokenIds || !owner_id) return

        let ids:JSONValue[] = tokenIds.toArray()
        const tokenId = ids[0].toString()

        let token = Token.load(tokenId)

        if (!token) {
          token = new Token(tokenId)
          token.tokenId = tokenId
          
          token.image = ipfsHash + '/' + tokenId + '.png'
          let metadata = ipfsHash + '/' + tokenId + '.json'
          token.metadata = metadata
  
          let metadataResult = ipfs.cat(metadata)
          if (metadataResult) {
            let value = json.fromBytes(metadataResult).toObject()
            if (value) {
              const kind = value.get('kind')
              if (kind) {
                token.kind = kind.toString()
              }
              const seed = value.get('seed')
              if (seed) {
                token.seed = seed.toI64() as i32
              }
            }
          }
        }

        token.ownerId = owner_id.toString()
        token.owner = owner_id.toString()

        let user = User.load(owner_id.toString())
        if (!user) {
          user = new User(owner_id.toString())
        }

        token.save()
        user.save()
      }
    }
  }
}

```

These mappings will handle events for when a new token is created, transferred, or updated. When these events fire, the mappings will save the data into the subgraph.

### Running a build

Next, let's run a build to make sure that everything is configured properly. To do so, run the `build` command:

```sh
$ graph build
```

If the build is successful, you should see a new __build__ folder generated in your root directory.

## Deploying the subgraph

To deploy, we can run the `deploy` command using the Graph CLI. To deploy, you will first need to copy the __Access token__ for your account, available in [The Graph dashboard](https://thegraph.com/hosted-service/dashboard):

![Graph Dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/820lwqh8yo3iyu7fsbhj.jpg)

Next, run the following command:

```sh
$ graph auth https://api.thegraph.com/deploy/ <ACCESS_TOKEN>

$ yarn deploy
```

Once the subgraph is deployed, you should see it show up in your dashboard:

![Graph Dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q548jeq4kuhgddrjv0dv.jpg)

When you click on the subgraph, it should open the Graph explorer:

![The Foundation Subgraph](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9qremconu1io72z3g6pa.png)

## Querying for data

Now that we are in the dashboard, we should be able to start querying for data. Run the following query to get a list of tokens and their metadata:

```graphql
{
  tokens {
    id
    ownerId
    tokenId
    image
    metadata
    image
    kind
    seed
  }
}
```

We can also configure the order direction:

```graphql
{
  tokens(
    orderBy:id,
    orderDirection: desc
  ) {
    id
    ownerId
    tokenId
    image
    metadata
    image
    kind
    seed
  }
}
```

Or choose to skip forward a certain number of results to implement some basic pagination:

```graphql
{
  tokens(
    skip: 100,
    orderBy:id,
    orderDirection: desc
  ) {
    id
    owner
    tokenId
    image
    metadata
    image
    kind
    seed
  }
}
```

Or query for users and their associated content:

```graphql
{
  users {
    id
    tokens {
      id
      ownerId
      tokenId
      image
      metadata
      image
      kind
      seed
    }
  }
}
```

We can also with full text search:

```graphql
{
  tokenSearch(text: "Normies") {
      id
      ownerId
      tokenId
      image
      metadata
      image
      kind
      seed
  }
}
```
