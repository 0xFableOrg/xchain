# Research on Relayers

Why Research on Relayers?

Relayers play a crucial role in sending and executing messages across different chains. The goal of this research is to explore how the new bridge design for Superchain can benefit from leveraging other relayer networks or how to design relayer infrastructure with improved performance and incentive mechanisms.

This page covers the general roles of relayers, how other bridges use relayers, and examples of similar relayer services employed in other infrastructures like MEV.

## 1. Relayers

What is Relayer?

Every cross-chain message requires two transactions to be delivered: one on the origin chain to send the message and one on the destination chain to receive the messages. Relayers are responsible for sending the second transaction.

**1.1.1 Roles of Relayers**

Relayers play a crucial intermediary role in facilitating cross-chain communication, transaction execution and incentive distribution across independent blockchain networks.

- Message relaying: Relayers relay messages and transactions between different blockchains. This involves receiving messages/transactions from one blockchain and transmitting them to the target blockchain(s).
- Network monitoring: Relayers monitor the blockchain networks they are connected to in order to identify messages that need to be sent to other chains.
- Message verification: Relayers verify the validity and integrity of messages/transactions before relaying them to other chains. This includes checking signatures, balances, etc. to prevent relaying of invalid data.

**1.1.2 Core Responsibilities by Relayers**

- Liveness: Relayers guarantee that messages/transactions will be relayed and delivered within a certain timeframe. This ensures transactions can be processed in a timely manner.
- Safety: Relayers only relay valid and authorized transactions that adhere to the protocols of both source and destination chains. This ensures no invalid or malicious transactions are propagated.
- Integrity: The integrity of relayed messages is maintained through the use of digital signatures and encryption. This prevents tampering or modification of transactions in transit.
- Censorship resistance: Relayers do not have the ability to selectively censor or drop specific transactions. All valid transactions are relayed impartially.
- Latency: Relayers provide latency guarantees around how long it will take for a transaction to be relayed and included in the destination chain. This helps ensure predictable delivery times.
- Availability: The relayer infrastructure and services are designed to be highly available and resilient to failures or attacks. Relayers aim to deliver 99.99% uptime for transaction relaying.

**1.1.3 Reference**

[Relayers in the Evolving Modular Blockchain Stack by Catalyst](https://blog.catalyst.exchange/relayers-in-the-evolving-modular-blockchain-stack/)

[Blockchain Bridges: Navigating a Fragmented Universe](https://www.coinbase.com/institutional/research-insights/research/market-intelligence/blockchain-bridges-navigating-a-fragmented-universe)


## 2. Relayers in Bridges

- Relayer in the Architecture
- incentive structure
- Implementation

### 2.1 IBC

**2.1.1 Relayer in the Overall Architecture**

[ibc/spec/relayer/ics-018-relayer-algorithms/README.md at main · cosmos/ibc](https://github.com/cosmos/ibc/blob/main/spec/relayer/ics-018-relayer-algorithms/README.md)

[Relaying The Message: A Deep Dive Into IBC Relayer Operations | by Interchain | The Interchain Foundation | Medium](https://medium.com/the-interchain-foundation/relaying-the-message-a-deep-dive-into-ibc-relayer-operations-6ff763a2a22f)

[Relayer | IBC-Go](https://ibc.cosmos.network/v7/ibc/relayer)

[IBC relaying guide | Celestia Docs](https://docs.celestia.org/developers/ibc-relayer)

**2.1.2 Incentive Structure**

[Moving Relayer Incentives On-Chain: Fee Middleware, Fee Grant and Budget Modules | by IBC Protocol | The Interchain Foundation | Medium](https://medium.com/the-interchain-foundation/moving-relayer-incentives-on-chain-fee-middleware-fee-grant-and-budget-modules-b6a13eced375)

[ICS29: How do relayer fees work with IBC in the Cosmos ecosystem? | by Kam | Imperator.co | Medium](https://medium.com/imperator-guide/ics29-how-do-relayer-fees-work-with-ibc-in-the-cosmos-ecosystem-e89ae6cf4530)

**2.1.3 Implementation**

[cosmos/relayer: An IBC relayer for ibc-go](https://github.com/cosmos/relayer)

[Introduction - Hermes (IBC Relayer CLI) Documentation](https://hermes.informal.systems/)

[hyperledger-labs/yui-relayer: IBC Relayer implementations for heterogeneous blockchains](https://github.com/hyperledger-labs/yui-relayer?tab=readme-ov-file)

### 2.2 Layerzero

[Introducing LayerZero V2. Today marks the deployment of LayerZero… | by LayerZero Labs | LayerZero Official | Dec, 2023 | Medium](https://medium.com/layerzero-official/introducing-layerzero-v2-076a9b3cb029)

### 2.3 Hyperlane

[Run a Relayer | Hyperlane v3 Docs](https://v3.hyperlane.xyz/docs/operate/relayer/run-relayer)

### 2.4 General Relaying Services: Gelato

Used by Connext, Hyperlane

[Gelato Network | Relay: Gasless transactions](https://www.gelato.network/relay)

## 3. Relayers in Other Infrastructure

### 3.1 MEV Relay

Relayers play a key role in the MEV by facilitating the submission of bids from MEV builders to Ethereum block proposers. Builders submit their transaction bundles and bids to multiple relays in order to maximize the chances of inclusion. The relays then forward these bids to proposers running protocols like MEV-Boost. They are also responsible for delivering the winning block bodies containing bundled transactions to the rest of the network after a block is committed. This ensures the transactions are validated and added to the blockchain. 

However, relays currently operate as a public good without a clear incentive structure. This is unsustainable long-term and risks relay centralization as only a few players will be able to subsidize relay operations indefinitely. Several proposals aim to incentivize relays by allowing them to collect fees, either directly from builders by modifying bids, or via deferred payment contracts. However, aligning the incentives of builders, validators, and relays remains challenging. Future work is still needed to develop relay incentives that balance sustainability, decentralization, and the interests of all parties.

[https://frontier.tech/optimistic-relays-and-where-to-find-them](https://frontier.tech/optimistic-relays-and-where-to-find-them)

[https://ethresear.ch/t/why-enshrine-proposer-builder-separation-a-viable-path-to-epbs/15710](https://ethresear.ch/t/why-enshrine-proposer-builder-separation-a-viable-path-to-epbs/15710)

[MEV-Boost: Merge ready Flashbots Architecture - The Merge - Ethereum Research](https://ethresear.ch/t/mev-boost-merge-ready-flashbots-architecture/11177)

**3.1.1 Incentivization**

[The Pursuit of Relay Incentivization — Ballsyalchemist](https://mirror.xyz/0xE21b1e6f471EDeF18264e9BBe51b7fA7643EE6B5/0Sh7BDW7qgH_nadfqF8bpmnjxnfoYzPFvRdmoIoi9mg)

3.2 Shared Sequencer, Relaying block commitments

**3.2.1 Astria**

[astria/crates/astria-sequencer-relayer at main · astriaorg/astria](https://github.com/astriaorg/astria/tree/main/crates/astria-sequencer-relayer)