# Cross-Chain Interoperability Report

By [Norswap](https://twitter.com/norswap),  
with the help of [Frederik Lührs](https://www.linkedin.com/in/frederik-luehrs/), [Heechang Kang (xpara)](https://x.com/xparadigms), [Naman Garg](https://x.com/namn_grg)  
and the advice and insights of many others!

---

<!-- TOC -->

- [Cross-Chain Interoperability Report](#cross-chain-interoperability-report)
  - [Context](#context)
  - [Outline](#outline)
  - [Superchain Models](#superchain-models)
  - [Bridge Taxonomy](#bridge-taxonomy)
  - [Bridge Properties](#bridge-properties)
    - [Safety](#safety)
    - [Liveness](#liveness)
    - [Latency](#latency)
    - [Incentives & Costs](#incentives--costs)
      - [Relayers](#relayers)
      - [Handling Variable Destination Chain Fees](#handling-variable-destination-chain-fees)
      - [Validators, Provers and Challengers](#validators-provers-and-challengers)
    - [Implementation Complexity](#implementation-complexity)
    - [Summary](#summary)
  - [Finality](#finality)
    - [Definitions & Problem](#definitions--problem)
    - [Cross-Chain Contingent Blocks](#cross-chain-contingent-blocks)
    - [Shared Validity Sequencing (Dependency Mechanism)](#shared-validity-sequencing-dependency-mechanism)
    - [Limitations and Attack Vectors](#limitations-and-attack-vectors)
  - [Atomicity & Synchronicity](#atomicity--synchronicity)
    - [Message-passing vs Bundling](#message-passing-vs-bundling)
    - [Atomic Inclusion](#atomic-inclusion)
      - [Shared Sequencing](#shared-sequencing)
      - [Builder-Proposer Separation (PBS)](#builder-proposer-separation-pbs)
      - [Reclaiming Safety](#reclaiming-safety)
      - [Applications to the Superchain (Atomic Inclusion)](#applications-to-the-superchain-atomic-inclusion)
    - [Atomic Execution](#atomic-execution)
      - [Running Multiple Nodes](#running-multiple-nodes)
      - [State Locking](#state-locking)
      - [Crossbar System](#crossbar-system)
      - [Shared Validity Sequencing (SVS) + PBS](#shared-validity-sequencing-svs--pbs)
      - [Applications to the Superchain (Atomic Execution)](#applications-to-the-superchain-atomic-execution)
      - [Domain-Specific Solutions](#domain-specific-solutions)
    - [More Powerful Models](#more-powerful-models)
  - [Messaging Formats & Interfaces](#messaging-formats--interfaces)
    - [Landscape](#landscape)
    - [Design Considerations](#design-considerations)
    - [Token Layer](#token-layer)
  - [Conclusions & Recommendations](#conclusions--recommendations)
  _ [Bridging Mechanisms](#bridging-mechanisms)
  _ [Latency](#latency-1)
  _ [Implementation Complexity](#implementation-complexity-1)
  _ [Atomicity](#atomicity) \* [Summary](#summary-1)
  <!-- TOC -->

---

## Context

This report has been [commissioned][rfp] by the Optimism Governance, with the goal of providing an
overview of the current state of cross-chain interoperability solutions and their applicability to
the Optimism Superchain in particular, as well as making recommendations regarding the path forward.

[rfp]: https://app.charmverse.io/op-grants/page-012176298170861077

When we talk about cross-chain interoperability, we are talking about the ability to transmit
information between blockchains, and therefore also let actions in one blockchain trigger actions on
another blockchain. We will often use the term "message" to refer to a generic piece of information.

For the most part, cross-chain interoperability is concerned with bridging messages and assets. We
depart from some connotations of "bridging" by not confining ourselves to the app layer, as we also
consider chain-level architecture possibilities — for instance, rollup sequencing.

While we talk about "cross-chain interoperability", the real goal of this report is to uncover
possibilities for cross-**rollup** interoperability. In the process, we will however say a good deal
that applies to any kind of chain. Similarly, we will use the classical architecture of EVM chains
as a basis for conversation and nomenclature, though a lot of the discussion generalizes.

We will consider that you are aware why cross-chain interoperability is important, and skip the
corresponding motivational section. We will later define some desirable properties for cross-chain
interoperability.

The reader of this report is expected to have solid fundamentals in blockchain architecture, and
readily understand terms such as "rollup", "L1/L2", "finality", "bridge" and "sequencing".

Finally, we note that this report is not meant to be a comparison of existing solutions, though some
are mentionned. Instead, we try to abstract away from specific implementations and focus on the
general solutions and their properties.

If you keen to read comparisons of bridging providers in particular, here are a few articles and
reports that have taken this approach:

- [Uniswap Foundation Bridge Assessment Report](https://uniswap.notion.site/Bridge-Assessment-Report-0c8477afadce425abac9c0bd175ca382)
- [Navigating Arbitrary Messaging Bridges: A Comparison Framework](https://blog.li.fi/navigating-arbitrary-messaging-bridges-a-comparison-framework-8720f302e2aa)
- [Navigating the Web of Interoperability: A deep dive into Arbitrary Message Passing Protocols](https://longhashvc.medium.com/navigating-the-web-of-interoperability-a-deep-dive-into-arbitrary-message-passing-protocols-43a469b9e7d)

## Outline

To enable bridging arbitrary messages from a source to a destination chain, the only real hard
requirement of consequence is a mechanism to authenticate the messages sent from the source chain on
the destination chain. This problem generalizes to the problem of authenticating blockhashes or
state roots from the source chain.

We discuss these mechanisms in the (voluminous) [Bridge Taxonomy](#bridge-taxonomy) section, where
we classify bridges as follows:

- embedded bridges
- light client bridges
- validator bridges
- slashing bridges
- optimistic bridges
- reverse bridges
- rollup bridges
  - proof-of-execution bridges
  - fault proofs bridges

In the [Bridge Properties](#bridge-properties) section, we will then define a few key properties of
bridges (safety, liveness, latency, incentive & costs, and implementation complexity), and assess
all bridges in the taxonomy in the light of these properties.

Additionally, two important things can be added to the authentication mechanism.

First, a mechanism to remove finality risk from the bridge. Bridging cannot be safe as long as the
source chain did not finalize. The current minimum time to finality for a rollup using Ethereum as
the data availability (DA) layer is 15 minutes: 14 minutes for Ethereum to finalize in the best
case, and a [1-minute average time](https://l2beat.com/scaling/liveness) for the rollup to post
transactions to Ethereum. This issue can be tackled by making sure that if a source chain block
sending a message rolls back, then the destination chain block receiving the message must also roll
back. We discuss this at length in the [Finality](#finality) section.

Second, a mechanism to ensure atomic inclusion of transactions across chains, or atomic execution of
a transaction and the messages it sends. Atomic inclusion can easily be achieved with shared
sequencing, but offer few hard guarantees of interest (it is very interesting for MEV extraction,
however). Atomic execution is a much more powerful property, but also much harder to achieve in
practice. This is the topic of the [Atomicity & Synchronicity](#atomicity--synchronicity) section.

The [Messaging Formats & Interfaces](#messaging-formats--interfaces) section will discuss how
cross-chain transactions are represented on the source and destination chain, and highlight a few
design trade-offs for these interfaces.

Finally, the [Conclusions & Recommendations](#conclusions--recommendations) section will summarize
the findings of the report and give some opinions and recommendations for the future.

## Superchain Models

In considering the interoperability of two rollup systems, it is important to consider their
relationship. For instance, is the interoperability possible permissionlessly, or does it require
collaboration (or even software changes) from one or both sides?

The Optimism Superchain [is][superchain] "a horizontally scalable network of chains that share
security, a communication layer, and an open source development stack." OP Labs has unveiled some of
its plans for cross-rollup interoperability, but many avenues remain open in the future.

[superchain]: https://app.optimism.io/superchain

In our report, we will consider three models:

- **The small model**, where the inclusion of a chain in the Superchain is a permissioned process,
  that requires a number of guarantees from the candidate chain. This seems to be the current "social"
  model, embodied by [The Law of Chains][law-of-chains].

- **The large model**, where any chain can join the Superchain, with minimal oversight and overhead.

- **The graph model**, where each chain independently decides which chain it can accept messages
  from. For a new chain, accepting messages from other chains is permissionless, while in the other
  direction the other chains need to opt into accepting messages from the new chain. This is the
  current "technical" model, reflected in the [Optimism interop specifications][interop-specs].

[law-of-chains]: https://gov.optimism.io/t/final-law-of-chains-v0-1
[interop-specs]: https://specs.optimism.io/interop/overview.html

It is important to note that no model is superior, they simply have different trade-offs. The small
model can enable solutions where some components of the system like sequencers are more "trusted"
because there are decentralization or financial guarantees in place. The large model has the
advantage of allowing anyone to run their rollup however they want, but it excludes solution where
the misbehaviour of a rollup would negatively impact other rollups in the Superchain. The graph
model is a hybrid that lets every chain defines its own trust set.

These solutions are also not exclusive. In fact, in the context of the Superchain, the graph model
can be seen as unifying the small and large models: a core set of chains would be fully
interoperable (trust each other) while a broader set of sattelite chains would trust other core or
satellite chains, but not be trusted by the core set. It's also possible that new techniques will
make the large model more practical.

## Bridge Taxonomy

We propose a taxonomy of bridges that divides them into seven categories:

- embedded bridges
- light client bridges
- validator bridges
- slashing bridges
- optimistic bridges
- reverse bridges
- rollup bridges
  - proof-of-execution bridges
  - fault proofs bridges

Let's define each of those. We keep the descriptions short here, as we will discuss their respective
properties later.

**Embedded bridges** are systems where the destination chain embeds a full node of the source chain
— all validators are thus expected to also follow and validate the source chain (or trust someone
else to do so). This is the model used by the rollup bridges in the L1 to L2 direction.

**Light client bridges** are systems where the destination chain runs or verifies the execution a
light client of the source chain. By _light client_, we mean a system that verifies the consensus of
the source chain, but does not verify the execution of transactions (1). This is, after a certain
point of view (2), no less secure, because a chain's consensus usually also posesses the authority
to upgrade the chain (and so to "allow" any execution outcome). This category also includes bridges
using zero-knowledge proofs of consensus.

(1) Let's note that the Ethereum light client protocol (introduced in the Altair hard fork) does not
fall under this definition, as it relies on an attestation commitee whose security assumptions are
considerably weaker than the Ethereum consensus. [Read here][altair] for more details.

[altair]: https://prestwich.substack.com/p/altair

(2) The key difference between a light client and full client is that a full client also verifies
execution, and as such, is capable of participating in "social consensus": it may choose to follow
the set of rules (the state transition function) that it considers legimitate.

**Validator bridges** are systems where a set of validators are commissioned to validate the source
chain, and must attest to the authenticity of the information (whether it is a state root,
transactions, or anything else) being bridged, and the destination chains verifies these
attestations. This is very similar to light client bridges — validator bridges verify the consensus
of validators — with the difference that the validators do not need to participate in the actual
consensus of the source chain, and their attestations do thus not carry the same guarantees.

**Slashing bridges** are systems where a permissioned set of relayers can relay
messages from the source chain to the destination chain. If a relayer posts a message on the
destination chain that did not originate from the source chain, anybody can post a copy of this
(signed!) message to the source chain, causing the relayer to be slashed.

This system is only economically secure, and must be used in a way where the system can function
with forged messages, as well as ensure that the loss incurred by a forged message is lower than the
relayer's bond. In practice, this probably entails rate-limiting relayers in terms of economic
value.

There is also a question of how to ensure relayers actually deliver messages. At a minimum, we could
allow users to challenge relayers, forcing them to post a signed relaying message on the source
chain, such that the user (or a benevolent third-party) is at least able to relay the message
manually if needed.

We're not aware of any pure slashing bridge deployment, but it is a sound design within its
constraints, and a slashing mechanism is often added to other types of bridges.

**Optimistic bridges** work like the slashing bridges, but additionally add a period during which
messages can be challenged on the destination chain before they are considered valid. A set of
permissioned actors (this could be the relayer set itself) are responsible for challenging, and
challenges immediately invalidate the relayed message. Invalid challenges can themselves be
challenged on the source chain, in the same manner as invalid messages. This system is secure as
long as there is an honest actor that can challenge a forged message within the challenge period.
Slashing of relayers and challengers is required to avoid griefing attacks.

**Reverse bridges** are bridges that rely on the existence of another bridge (which we'll call
"forward bridge") in the reverse direction (destination to source). A user posts a message on the
source chain, and a relayer relays it to the destination chain. Then, using the forward bridge, the
relayer proves that he did relay the message and unlocks payment from the user.

Reverse bridges are different from the other bridges in that they make no claim about the
authenticity of relayed messages. As such, these messages are limited to those that could have been
sent regardless of their initiation from the source chain. These messages can contain end-user (EOA)
signatures for validation, but there is no way to establish that a smart contract on the source
chain emitted a message. Use cases include cross-chain asset transfers, triggering permissionless
actions on another chain, and operating a smart wallet controlled by an EOA from another chain.

The reverse bridge inherits security assumptions from its forward bridge, though the risks for the
user and the relayer are altered.

**Rollup bridges** are bridges that enable bridging in the L2 to L1 direction in rollup systems (in
the L1 to L2 direction, rollups use embedded bridges), as well as between rollups that share the
same architecture and bridging mechanism.

The specificity of rollups is that they do not reach consensus on the state of the blockchain via a
majority vote of validators. Instead, they prove on the L1 chain that the state (represented by a
state root) is valid, given a commitment to a state transition function and the included
transactions. If the source of truth (i.e. the [data availability layer][dal]) for the L2 chain
transaction data is not the L1 chain, this additionally require bridging these commitments over,
with the additional trust assumptions that this entails.

[dal]: https://www.alchemy.com/overviews/data-availability-layer

Once the validity of a L2 chain block's state root has been established, messages sent within that
block can be relayed on the L1 chain (by proving the message against the state root). So the
L2-to-L1 bridge comes almost for free with the state consensus mechanism.

Rollup bridges can also be used to bridge between rollups, the mechanism here being specific to the
flavour of rollup bridge.

Rollup bridges come in two flavours, specific to either validity/ZK rollups (proof-of-execution
bridges) or optimistic rollups (fault proof bridges).

**Proof-of-execution bridges** are systems where the L1 chain verifies a zero-knowledge (ZK) proof
of the correct execution of a transition on the L2 chain. This is also known as a validity proof,
since the cryptographic proving mechanism is used predominantly for its succintness property (it's a
lot cheaper to verify a proof than to execute the computation) than its privacy property.

Cross-rollup bridging using proof-of-execution is fairly straightfoward: the destination rollup
chain can simply verify the validity proof. For this, it needs access to the L1 and possibly the DA
blockhashes, which it is already tracking, being a rollup itself. These hashes commit to all the
inputs (mostly transactions) that go into making a block, and are thus the only public inputs to the
validity proof besides the proven source chain hash or state root. The inputs (transactions)
themselves can be private inputs to the proof, which are proven by the proof against the public
input hashes.

**Fault proof bridges** are systems where the L1 chain waits for a challenge period before accepting
a proposed L2 chain block's state root as valid. If a challenge is made, a challenge game takes
place on the L1 chain to determine the challenge's validity.

Explaining challenge games is outside the scope of this document, but if interested, refer to [this
introductory thread][cannon-thread] and [this video][cannon-video] for the general idea, as well as
[this article][game-article] and [this video][game-video] for the finer details.

[cannon-thread]: https://twitter.com/norswap/status/1502085000967061504
[cannon-video]: https://www.youtube.com/watch?v=2RjkkoGUTx0
[game-article]: https://blog.oplabs.co/dispute-games/
[game-video]: https://www.youtube.com/watch?v=nIN5sNc6nQM

Cross-rollup bridging using fault proof bridges requires sequencers to deliver messages from other
rollups. A shared fault proof system ensures that blocks that deliver a message that wasn't sent on
the source rollup is considered invalid.

---

We believe this taxonomy appropriately captures all bridge designs that are have been used or
proposed. Variations are of course possible, though we believe they do not usually fundamentally
alter the bridge's properties.

Hybrid bridges are sometimes possible. For instance, slashing can be added to most other types of
bridges, though it usually does not fundamentally alter their properties. An interesting example is
adding a validator set to a zero-knowledge bridge (whether light client or rollup), in order to
guard against possible proving system exploits.

## Bridge Properties

We identify 5 key bridge properties:

- safety
- liveness
- latency
- incentives & costs
- implementation complexity

**Safety** characterizes the conditions under which the bridge might malfunction, either
by delivering messages that weren't sent, or by _never_ delivering sent messages. It can also be
characterized in terms of the possible economic losses incurred by the bridge's malfunction.

**Liveness** characterizes the conditions under which the bridge might stop functioning, either by
stopping to accept new messages, or by temporarily not delivery messages that were sent.

**Latency** characterizes the time it takes for a message to be delivered on the destination chain
after it is included in a block on the source chain.

**Incentives & costs** are about the cost incurred and profits accrued by various actors in the
bridging system, and looking at how these incentivize their behaviour, especially in relation to the
three previous properties (safety, liveness and latency).

**Implementation complexity** is about the complexity of the bridge system's implementation.
Complexity comprises the size of the codebase, the number of independent software components and
actors, use of advanced technology and algorithms (in particular cryptography), or other niche
technology where expertise is less widespread. Most bridge hacks have resulted from implementation
problems rather than fundamental design flaws.

---

Let's make some observations on the properties of various bridge types in our taxonomy.

The "Summary" section will recapitulate the properties of each bridge type.

### Safety

We define safety in terms of "safety violations", which are situations that violate the main bridge
invariant that every sent message must be delivered, and only sent messages must be delivered:

1. Lost message: A message was sent on the source chain, but is never delivered on the destination
   chain.
2. Forged message: A message is delivered on the destination chain that was never sent on the source
   chain.

Most bridges don't allow either of those outright (except for slashing bridges, which allow forged
messages, albeit it penalizes them). However, there always exist conditions where those safety
violations become possible.

Safety is therefore a spectrum, and must be characterized by the additional assumptions that are
required for the bridge to be safe (on top of the safety — the correct execution — of the source and
destination chain).

A popular distinction is to set apart trust-minimized bridges, which do not add security assumption
other than the correctness of the implementation. They comprise embedded bridges, proof-of-execution
brdiges and light client bridges.

Fault proof bridges (used in optimistic rollups) do add a single honest challenger assumption: the
rollup is safe as long as there is a single (permissionless) actor that will challenge an incorrect
state root within the challenge window (which is usually large — traditionally 7 days).

Embedded bridges are the safest type of bridge, but also the least scalable and realistic,
because they require a blockchain system to embed another. They are only suitable for rollups.

Both rollup bridges and light client bridge need to be updatable, because the source chain might be
updated. So there is a trust assumption in the update mechanism, whether it's a multisig or a
governance vote. Update delays can be added to mitigate the risk from a rogue update, although this
may hamper the ability to quickly react to a rollup vulnerability.

Light client bridge can get around this by enabling the chain's consensus to update the bridges.
This can work if the chain is light client aware (and upgrades include a generic "update light
client" transaction signed by the validators), or if the validators control the bridge deployment
(though this precludes permissionless light client bridges).

Slashing bridges are weird. They're in a sense completely unsafe, since there is no way to prevent a
forged message, but the idea is that the loss incurred by the forged message, as well as the profit
incurred by the forger, will be lower than the amount slashed from the forger. This equivalency
must be calibrated correctly, ideally using hard in-protocol accouting.

Another risk for slashing bridges comes from the liveness of the source chain. If the source chain
is unavailable for long enough, a malicious relayer might be able to withdraw its bond before
incurring a slashing penalty. This can be mitigated by making the withdrawal delay long enough that
this is unlikely to occur.

Optimistic bridges rely on the existence of a single honest challenger that is able to challenge
within the challenge period. Compared to fault proof bridges, the window is usually much smaller
(e.g. 30 minutes vs 7 days for fault proof bridges), which exposes the system to more potential
attack on the liveness and censorship resistance of the destination chain. The safety here could be
characterized by the cost of making the chain unavailable for the duration of the challenge period,
or the cost of censoring challenges for that period.

Validator bridges derive their safety from the decentralization of the validator set, which can be
analyzed in the same way as one analyzes the decentralization of a blockchain.

Slashing bridges and optimistic bridges require relayers to lay down a bond, while validator bridges
require relayers to be part of a permissioned set. Unless new actors are voted in by the existing
ones, the mechanism to select the permissioned actors (e.g. a governance vote) adds its assumptions
to the bridge's safety.

Reverse bridges derive their safety from the underlying forward bridge, although it alters some of
the properties. For other bridges, the safety risk lies in message being forged on the destination
chain. Because reverse bridge can only enact permissionless actions on the destination chain, this
risk does not exist here.

However, the permissionless action might be costly to the relayer (for instance, transfer an asset),
and so a malfunction in the forward bridge that would prevent the relayer from unlocking payment (or
the execution of an action of interest) on the source chain is a safety risk.

Similarly, if a message is forged in the forward bridge, the user in the source chain might be
forced to pay (or execute some action) without the desired action having taken place on the
destination chain.

A liveness failure in the source chain could also delay payments (or logic execution) for the
relayer. This is not theoretically a safety issue, but since reverse bridges are often used for
asset transfers, the relayer might incur losses due to longer-than-expected price exposure.

Finally, we note that some bridges might want to enable the delivery of messages faster than the
finality period of the source chain, as doing this poses additional risks, which we will discuss in
the [Latency](#latency) section.

For further reading on safety beyond this report, see [the Crosschain Risk
Framework](https://crosschainriskframework.github.io), which has a bigger focus on operational
security, and the [L2Bridge Risk Framework](https://gov.l2beat.com/t/l2bridge-risk-framework/31),
which also touches on liquidity networks.

### Liveness

Liveness is defined in terms of "liveness violations", which are situations during which either:

1. Delayed messages: Sent messages are not delivered for an extended period of time. This is a
   subjective appreciation, but we will generally consider this an especially undesirable issue
   when the delivery could be made (e.g. the destination chain is live and uncongested) but isn't.
2. Bridge down: New messages cannot be sent.

The liveness of a bridge may depend on the following factor:

- liveness of the source and destination chain
- censorship-resistance of the source and destination chain
- built-in mechanism (e.g. an emergency pause function)
- the existence, incentives and liveness of relayers
- the liveness of validators (for validator bridges)

In general, liveness and censorship-resistance have the same effects: censorship can be seen as a
targeted liveness failure for some types of transactions, so we'll treat liveness failures as
encompassing censorship in this discussion.

A liveness failure in either the destination or source chain automatically causes respectively a
type 1 (delayed messages) or type 2 (bridge down) liveness failure in the bridge, but this is
independent of the bridge design.

Additionally, in optimistic bridges, an extended liveness failure of the source chain could enable a
malicious relayer to challenge messages on the destination chain without incurring slashing, hence
causing a liveness issue, but not a safety issue. This is similar to the safety issue of a slashing
bridge, and the same mitigation applies: make the relayer bond withdrawal delay long enough that
this is unlikely.

Bridges might include built-in mechanisms to pause the bridge. This is generally desirable, to help
combat hacks and other operational failures. However, care must be taken to ensure that this
mechanism cannot be abused. Ideally, message senders should not be held hostage by this mechanism,
and should be able to cancel yet-to-be-delivered messages.

Finally, the existence of relayers is a key factor in the liveness of bridges. There are two cases
to consider: the permissonless case, where users can at worst become their own relayer, and the
permissioned case, where relayers must belong a permissioned set. Bridges that require a bond from
relayers but are otherwise permissionless sit in the middle, as most users might not have enough
capital to become a relayer, but a benevolent third-party could step in.

Embedded bridges are unique in not requiring relayers, as the relayers are effectively the
validators and/or sequencers of the chain.

Relayers can be made fully permissionless in light client bridges, reverse bridges, validator
bridges, and rollup bridges (although that is not the case for some rollups today, on which posting
the state root is the purview of the sequencer, whereas users "relay" messages by posting proofs
against a finalized state root).

In the permissioned case, liveness requires relayers to be up and running. For validator bridges
(whether permissoned or permissionless), a voting quorum of validators must be live.

Designs using permissionless but bonded relayers can be considered to be more live than designs with
permissioned relayers, though in general a failure of relayers to show up whenever expected can
still be considered to be a liveness failure.

Some designs can be improved by allowing users to cancel a message request.

In general, liveness is improved by decentralizing the relayer and validator sets, in the same ways
that a chain's validator set can be decentralized: geographical diversity, jurisdictional diversity,
elimination of shared effective controls, client diversity, ...

Finally, technically operational relayers still have to "show up" to relay messages, which they may
not do in the absence of proper incentives. This will be discussed in the [Incentives &
Costs](#incentives--costs) section.

### Latency

We define latency as the time it takes for a message to be delivered on the destination chain after
it is included in a block on the source chain.

Latency can theoretically be 0, in the case of atomic composability (see later discussion) or
whenever a single entity is able to build block on both chains and is willing to carry whatever
finality risks may exist. Since such a system can also be used to ensure atomic composability, we
will also cover it in that section.

More commonly, latency will be at minimum the time requires for relayers to observe a block on the
source chain and make a transaction to deliver it on the destination chain, which is the block time
of the destination chain.

In general, there is no way to guarantee that a message will be delivered in the next block given
the presence of congestion on the destination chain as well as networking issues. In fact,
theoretically, there is no way to guarantee a bounded delivery time, though in practice this is
rarely an issue.

Optimistic and fault proof bridges have intrinsic latency delays, which are due to their challenge
period.

Proof-of-execution bridges must allow for proving time, which, at the time of writing, cannot
measure up to block production time. Provers are now able to prove an Ethereum block in [a few tens
of minutes][zkevm2] (and [as fast as a few minutes][zkevm3] if substituting Ethereum's Keccak hash
function for a prover-friendly hash function like Poseidon).

I can't resist a quick sidebar into the world of zero-knowledge proof generation.

zkEVMs run circuits to prove the execution of the EVM, whereas general zkVM prove the execution of a
general-purpose instruction set like RISC. To prove the execution of an EVM chain's state transition
function, zkVMs have to prove a much larger amount of computation, as they are proving the execution
of an EVM interpreter, whereas zkEVMs prove the EVM execution directly.

Both types of prover (who share the underlying technology) have improved rapidly over the past few
years. [In December 2002][zkevm3], Polygon zkEVM could prove 10M gas for ~0.30$ in 2.5 minutes (with
Poseidon, and note that full Ethereum blocks are 30M gas and that this isn't a reproducible
benchmark). Over the past year, zkVM have been catching up. In August 2024, Succinct SP1 (a zkVM)
was able to prove 10M gas (still with Poseidon) for ~0.14$ in about 11 minutes.

More recently, Polygon zkEVM has announced [type 1 proving][zkevm], which is able to prove real
Ethereum blocks that use the Keccak hash function. Their [self-reported benchmarks][zkevm2] report
proving a 10M gas block in 44 minutes for ~0.23$ and a 12M gas block in 78 minutes for ~0.41$. I
have heard rumblings that new "precompiles" for Keccak could bring those time down significantly
(this wasn't Polygon-specific, but the underlying technology can speed up both kind of provers).

Note that the cost and latency of proving are two faces of the same coin, as a lot of the
computation that goes into proof generation is highly parralelizable, meaning one can trade off cost
for latency by throwing more hardware at the problem.

The technology is improving rapidly, and it doesn't seem far-fetched to expect performance gains of
one or two orders of magnitude within the next year or two.

[zkevm]: https://polygon.technology/blog/upgrade-every-evm-chain-to-zk-introducing-the-type-1-prover
[zkevm2]: https://docs.polygon.technology/cdk/architecture/type-1-prover/testing-and-proving-costs/
[zkevm3]: https://x.com/norswap/status/1751756866844082247
[sp1]: https://blog.succinct.xyz/sp1-benchmarks-8-6-24/
[sp2]: https://docs.google.com/spreadsheets/d/18BKAMMhM2uHf_ATO70wett7-75iXxWlQfbuL9UofghM/

Back to the discussion on latency, bridges must contend with the possibility of re-organizations
(re-orgs) of the source chain. It is not possible for a bridge to be safe if it delivers a message
before the source chain block where the message was sent has reached finality (at which point
re-orgs are no longer possible). Otherwise, the message could be delivered, but the transaction that
originated the message could be re-orged away, which would cause a safety violation.

The solution of that issue, short of only using chains with instant finality, is to make sure that
the destination chain rolls back (re-orgs) if the source chain rolls back, ensuring no message can
be delivered that weren't sent.

We will deal with this issue in the [Finality](#finality) section. This is one of the major issues
in interop for rollups, as they have a 15-minutes time-to-finality at best (Optimism takes 1 minute
to post blocks to Ethereum on average, though it can sometimes take as long as half an hour, and it
takes 14 minutes for Ethereum blocks to finalize in the best (but usual) case).

### Incentives & Costs

The various types of bridges have different types of actors that may need to be present for the
bridge to work, and thus be incentivized to do so. These are relayers, validators, challengers, and
provers.

We will first tackle relayers, as they are the most common actor, and some of the discussion will
generalize to other actors. We then tackle the issue of dynamic destination chain gas fees, which
can cause the user to specify a fee that would not cover the relayer's costs. We then discuss the
remaining actors.

#### Relayers

Relayers are the most common actors, needed in almost in every type of bridge, excepted in embedded
bridges. In some designs the choice can be made to let users relay their own messages. In particular
this is possible for any bridge that could enable permissionless relaying, i.e., validator bridges,
light client bridges as well as rollup bridges.

In validator bridge, it is only necessary to obtain a signature from the validators to be able to
relay. Alternatively, the validators may only decide to relay state roots, in which case a relayed
message must include a proof-of-inclusion against the state root (which anybody should be able to
generate). The same principle applies to rollup bridges: someone needs to relay the chain's state
root. For some rollups like OP-stack-based rollups, this is permissionless, in some others it must
be the sequencer. The state root needs to be validated (by submitting a zk proof or waiting for the
challenge window), then anybody can relay a message by proving it against the state root.

For proof-of-execution bridges, a prover must post a proof for the state root. This can be
permissionless, though it is not expected that users will take on this burden.

For light client bridges, someone needs to relay all the consensus-relevant information (usually an
aggregated signature) to the destination chain. This can also be permissionless, though it is not
expected to be the case.

Nevertheless, it is often desirable to have relayers that alleviate the relaying burden from the
user. For slashing bridges and optimistic bridges, relayers must be bonded, hence this role cannot
be taken permissionlessly by users.

In the case where relaying is permissioned, care should be taken to ensure that every sent message
is eventually delivered (avoiding a safety violation). This could take the form of a challenge where
the message sender requests relayers to post a signed message on the source chain that the user can
then relay manually, otherwise relayers will be slashed (potentially gradually, until the requested
message is posted).

Reverse bridges are a bit of a special case, because while in theory the users could relay their own
messages, their usefulness often lay in guarantee that messages are automatically relayed, or that
the relayer performs a costly action (e.g. transfer a token on the destination chain, in exchange
for receipt of a token on the source chain).

Whenever relayers are expected, they need to be paid as they are taking on costs. An important part
of these costs consists of transaction fees on the destination chain. In addition, relayers expect
to make some profit. In the next section we will describe the issue that arise from the variability
in the transaction fees on the destination chains.

#### Handling Variable Destination Chain Fees

Relayers must be paid a fee. Because transaction fees fluctuate on the destination chain, it is
necessary to ensure that the fee is high enough to both cover the relayer's fees and incentivize his
continued operation (profit).

It may at first seem that it is easier to pay the relayer's fee on the source chain. That is
certainly the easiest (and often the only) way to collect the fee from the user. However, this
doesn't solve the problem of getting to the fee to the specific relayer that actually relayed the
message and thus accrued the associated costs. (The exception being reverse bridges, where this
determination is easy.)

In practice, it is easier to pay the relayer's fee on the destination chain. This is because the
payment can be unlocked as part of the transaction that delivers the message. The bridge must thus
make sure that the destination chain has liquidity to pay the relayer. If the bridge focuses on
token bridging, the relay transaction may deduce the destination chain fee from the tokens being
bridged.

If fees are collected on the source chain and paid on the destination chain, fee estimation is
required. This cannot be done on the source chain, and must thus be carried off-chain, and included
in any source chain transaction that might trigger a message to be sent.

In the case where the relayer payment is set too low because the destination chain fees have gone
up, it should be possible to increase the fee on the source chain. Care should be taken to avoid
replay attacks (delivering the same message twice).

As an aside: such a mechanism could also double as a way to replay delivered but failing messages.
This could happen if the message triggers execution that is only transiently failing (e.g. because
of contract state), or if the user set a gas limit that is too low.

In reality, bridge operator _sometimes_ charge a flat fee that they know to be profitable in the
foreseeable future, or sometimes even operate relayers without explicit payments, if the destination
chain fees are low and expected to remain so — revenue being made at the application layer, e.g. by
fees for token transfers. It is also possible for bridge protocol to incentivize relayers with their
protocol token, or any other kind of offchain deal.

It is also possible to use third-party relaying services, which can be paid either onchain (e.g.
[Chainlink][chainlink], [Gelato][gelato], [Keep3r][keeper]) or offchain (e.g. [Gelato][gelato]).
Again, the issue of variable destination fee exists, and third-party relayers usually enable
adjusting the fee if paid on the source chain.

[gelato]: https://docs.gelato.network/web3-services/relay
[chainlink]: https://docs.chain.link/chainlink-automation
[keeper]: https://docs.keep3r.network/

If the relayer set is permissionless, it is generally safe to operate a fairly centralized relayer,
or to lean on a third party. This also enables coordinating the relayers offchain as to level the
playing field between them and ensuring a minimum amount of redundancy.

If the relayer set is permissioned however, relayers should not rely on any centralized entity to be
able to carry out their task and get paid, otherwise they might stop operating and cause a safety
violation.

Finally, let's consider the special case of embedded bridges (i.e. L1-to-L2 rollup brides). No
relayers are required because the L2 nodes monitor the L1 chain for updates, and **must** relay the
L1-to-L2 messages sent there. However, these messages still need to incur some kind of cost,
otherwise the L2 chain would expose itself to spam attack from users of the L1 chain, who could
write an L1 contract to send messages to the L2 chain for free.

Optimism opts to burn L1 gas to pay for these messages, and implements a separate fee market for
L1-to-L2 messages. Because it does this, it is able to ensure that there is always enough gas to at
least register the transaction on L2, where the user is able to replay it if its execution fails.

Arbitrum uses a "retryable ticket" system, where the user pays a fee to the L1 chain, and the user
is able to retry the transaction if it fails, for a certain time.

There are a few differences between the two approaches: Optimism's system does not collect a fee,
but this enables it to be called by contract functions that aren't `payable` in Solidity. Because it
stores failed messages in the L2 storage, nodes do not need to keep an index of replayable
transactions in the L2 node, and the transactions can be replayed forever, whereas Arbitrum only
allows replays for a week (probably as a security consideration).

#### Validators, Provers and Challengers

Validators (in validator bridges) incur cost by running blockchain nodes to verify the state of the
source chain. They can be incentivized in exactly the same way as relayers, although the situation
is easier since there they do not incur a highly variable cost like transaction fees.

Provers (in proof-of-execution bridges) incur cost by generating proofs — these proofs are
expensive, as they incur an overhead on the order of 10.000x compared to the original computation.

Because of the cost, it is better to have a way to select the prover (or the provers if some
redundancy is desired). This could be implemented by an on-chain round-robin between "preferred"
provers. This does not need to preclude permissionless proving, in case the preferred provers
somehow don't deliver a proof.

Optimistic bridges, slashing bridges and proof-of-execution bridges require challengers. Challengers
are watchguards who challenge incorrect messages, state roots, or other challenges. For slashing
bridges and fault proof bridges, challenging is permissionless as it cannot cause liveness or
latency issues.

We haven't seen much discussion on challenge incentivization in rollups, but it is expected that the
many "natural" node operators (exchanges, RPC providers, block explorers) could take on this role.

As said earlier, we do not know of any pure slashing bridge deployment.

In optimistic bridges, challengers are needed to avoid the delivery of forged messages on the
destination chain. As such, the role must be permissioned so that the challenger may be bonded on
the source chain, and slashed if he makes an invalid challenge. Since a valid challenge leads to the
slashing of a relayer, the challenger can be paid from the relayer's bond.

It is still unlikely that a relayer would cheat, which means that challengers could expect to never
make a profit. This problem can be neatly solved by making the relayer and challenger set identical,
which lets relayers verify other's relayer behaviour at little extra cost. Additionally, the safety
of the bridge is in the interest of the relayers, for whom it is a source of income.

### Implementation Complexity

We will say little about implementation complexity, because it is pretty subjective.

Let's reiterate some key areas to watch for when looking at implementation complexity:

- size and quality of the codebase
- number of independent software components and actors
- use of advanced technology and algorithms (in particular cryptography)
- maturity of the tech stack (e.g. compilers, libraries, etc.)

In general though, simplicity is key. Simpler systems are easier to reason about, and admit
less tricky corner cases.

A related factor to pay attention to is the presence of quality documentation and specification
for the system. It's generally not a good sign when those are not publicly available.

### Summary

This section tersely summarizes the properties of the various bridge types, although it's important
to read the preceding questions for nuance.

Regarding latency, the indications assumes that no one is willing to carry the finality (reorg)
risk. Otherwise, latency could be as low as the party taking the risk desires and is able to afford
given the current level of fees.

We haven't included a summary of the incentives discussion, as we felt it generally wasn't a key
factor in bridge design, resists summarization, and is redundant with the liveness and safety
sections (which identifies the actors that do need to be incentivized).

- embedded bridges
  - safety: trust-minimized (1)
  - liveness: no extra assumptions
  - latency: > source chain finality
  - complexity: simple
- light client bridges
  - safety: trust-minimized (1)
  - liveness: relayers
  - latency: > source chain finality
  - complexity: depends on source chain consensus
- validator bridges
  - safety: quorum of validators is honest
  - liveness: validators and relayers
  - latency: > source chain finality
  - complexity: simple
- slashing bridges
  - safety: only economically secure with single honest challenger assumption, source chain liveness risk
  - liveness: relayers (bonded or permissioned)
  - latency: > source chain finality
  - complexity: medium (bidirectional)
- optimistic bridges
  - safety: single honest challenger assumption
  - liveness: relayers (bonded or permissioned)
  - latency: > source chain finality + challenge period
  - complexity: medium (bidirectional)
- reverse bridges
  - safety: only permissionless actions, forward bridge must be safe or risk to relayers
  - liveness: relayers
  - latency: > source chain finality
  - complexity: medium (bidirectional)
- rollup bridges
  - proof-of-execution bridges
    - safety: single honest challenger assumption
    - liveness: relayers
    - latency: > source rollup finality (2) + challenge period
    - complexity: complex (fault proof)
  - fault proofs bridges
    - safety: trust-minimized (1)
    - liveness: relayers
    - latency: > source rollup finality (2)
    - complexity: complex (zero-knowledge cryptography)

(1) Trust-minimized means there is no additional safety assumption than those of the source
and destination chain, and implementation correctness.

(2) The rollup finality time is: DA submission delay + DA finality + DA bridge finality when DA !=
L1.

## Finality

As stated at the end of the [Latency](#latency) section, bridges must contend with the possibility
of re-organizations (re-orgs) of the source chain. If a bridge delivers a message before the source
chain block where it was sent reached finality, the message could be delivered, but the transaction
that originated the message could be re-orged away, which would cause a safety violation.

### Definitions & Problem

Block finality is the point when a block in the chain can be confidently considered to be part of
the canonical chain: re-orgs may no longer substitute another block at the same height.
Time-to-finality is the time it takes for a block to reach finality after having been produced.

Some chain have instant (aka "single-slot") finality. That is notably the case for Cosmos chains,
which use the Tendermint consensus algorithm. Proof-of-work chains have no finality, or rather
probabilistic finality: the more blocks are built on top of a block, the more unlikely (i.e. costly)
it is to re-org the block away.

The time-to-finality for a rollup is the sum of the time it takes for a block to be posted to its
data availability (DA) layer + the time for the DA layer to reach finality.

Amongst the leading data availability solutions, Celestia has instant finality with a block time of
around 12s ([1][celestia1]), EigenDA relies on Ethereum finality ([1]), and Avail usually finalizes
within two Avail blocks (~1 minute — [1][avail1], [2][avail2]).

[celestia1]: https://docs.celestia.org/developers/integrate-celestia
[eigenda1]: https://docs.eigenlayer.xyz/eigenda/overview
[avail1]: https://docs.availproject.org/docs/learn-about-avail/consensus/grandpa
[avail2]: https://blog.availproject.org/avails-core-features-explained/

Ethereum's time-to-finality is at minimum (and usually) the time it takes for 2 epochs (64 blocks)
to be produced, which is about 14 minutes. More precisely, the requirement is that the first block
in an epoch gets a 2/3 majority of votes in two separate round of votes, making it (and all blocks
in the epoch before it) final (1). It is possible for blocks to continue to be produced while
finalization fails to occur!

(1) See [here][two-rounds] to learn why two rounds of votes are required.

[two-rounds]: https://twitter.com/stonecoldpat0/status/1671303684016291848

The time to post blocks to the DA layer vary between rollups stack and configurations. For
optimistic rollups, it is purely a matter of cost: submitting multiple blocks at once allows to
amortize the fixed transaction costs, as well as compress multiple blocks together, which increases
the compression rate, hence decreasing the data cost. This also applies to ZK rollups, though they
additionally might want to wait for a proof to be generated, as this avoids a situation in which a
block is posted to DA but cannot be proved because of an issue in the proving system.

You can consult the [L2Beat liveness dashboard][liveness] to view the 30-day average block (tx data)
submission time for various rollups.

[liveness]: https://l2beat.com/scaling/liveness

We care about finality because relaying a message submitted on a non-finalized block could lead to
a safety violation: if the block is re-orged away, the message might be delivered, although it was
never sent on the canonical source chain.

Waiting for source chain finality is always safe, but it can be slow. A message betweens rollups
using Ethereum as their DA layer would take about 15 minutes to be safe to relay.

However, in cases where the latency is less than the time-to-finality, someone is taking the
finality risk. In asset bridging protocol, this will usually be the bridging protocol, and in
particular its liquidity providers. Similarly, in reverse bridges where relayers act as just-in-time
liquidity providers, they are the one taking the risk.

Such behaviour is not necessarily reckless — Ethereum has never experienced a reorg bigger than 7
blocks (~ 1m24s). And in the case a re-org did occur, it would be possible to recoup some of the
losses by resubmitting the transactions that submitted the message on the source chain.

It is however not a sound risk to take for general message-passing bridge that wishes to be usable
for any kind of message.

Beyond using a DA with fast finality, there is only one other solution to this problem: if the
source and destination chain are guaranteed to re-org together, then the problem disappears.

### Cross-Chain Contingent Blocks

The straightforward way to achieve this is to include on the destination chain the blockhash of a
block of the source chain, and enforce that if the source chain block fails to finalize, then the
destination chain block must not finalize (roll back) as well. We can call this a [cross-chain
contingent blocks][contingent] (2).

[contingent]: https://prestwich.substack.com/p/contingency

(2) James Prestwich actually termed it a "cross-ORU block" (ORU = Optimistic RollUp), but there is
no reason the same principle can't be applied to other types of chains.

However, with rollups that share the same DA layer, we can make the mechanism a lot more general.
Instead of commiting to a block in the source chain, we commit to a DA layer block. In this way, if
the DA layer re-orgs and makes the message disappear from the source chain, the message won't be
delivered on the source chain either (as we commited to a DA layer block that no longer exists).

Additionally, this makes the mechanism much more general: a destination rollup can now accept
messages from any source rollup that uses the same DA layer, as long as bridge operators can
independently verify the execution of the blocks of the source rollup.

This generalized mechanism is already applicable today — for instance OP stack rollups carry an
Ethereum blockhash in every L2 block. The sequencer has leeway in updating this blockhash, which is
guaranteed to be no more than 30 minutes old. However, on recently sampled Optimism blocks, the L1
block hash is about 2 minutes old, which is considerably smaller than the 15 minutes Ethereum
finality time.

### Shared Validity Sequencing (Dependency Mechanism)

An alternative but similar design is proposed in the [shared validity sequencing][svs] design, where
instead of the entire state root of the source chain being carried to the destination chain, only a
Merkle root of sent messages is carried over, and the destination chain will roll back if this root
is different from the one finalized on the source chain (3). The advantage of this design is that by
not copying the whole blockhash, it enables bidirectional bridging between two chains at blocks at
the same height if used in conjunction with shared sequencing (cf. section on [Atomic
Inclusion](#atomic-inclusion)). The idea being that the messages root can be included in the
destination chain's block at the same height at which it was updated on the source chain.

Optimism adopted a generalization of this mechanism for their experimental [interoperability
specification][interop-specs]. There, no merkle root of messages needs to be forwarded, instead a
single fault proof program would be responsible for validating all the block that form a closed
dependency graph with respect to their cross-rollup messages.

[svs]: https://www.umbraresearch.xyz/writings/shared-validity-sequencing

(3) The mechanism is the same as in cross-chain contingent blocks, but is here given a name: "shared
fault proofs".

### Limitations and Attack Vectors

These two solutions requires the destination chain to be aware of the source chain, so they are
necessarily permissioned and limited in scope.

Please note that those solutions are not full bridging mechanism: they say nothing on the validity
of the source chain blocks. They only ensure that bridges do not suffer from finality risk by
ensuring the destination chain rolls back if the message was not actually sent on the source chain.

Unfortunately, there are some possible attack vectors whenever the source blockhashes or state root
cannot be authenticated by the destination rollup (so depending on DA blockhashes is fine!). Let's
take a validator bridge as an example in the following discussion.

The simplest attack vector is that if the chains have any "fast bridges" that take on finality risk,
a malicious validator set could use those to double-spend: they would create an invalid block by
authorizing an incorrect state root, exit via the fast bridges, wait for the re-org to occur via the
shared fault proof, then spend their balance again.

An even more complex attack would be to exploit the mayhem that result from a chain rollback to
replay transactions of interest and extract a maximum of MEV from the chains. Since nobody except
the attacker would really be prepared for this scenario, they would have a big edge in capturing
this MEV.

These scenarios require a lot of sophistication and are not especially likely in our estimation, but
they show that the re-org mechanism cannot be dismissed as almost irrelevant, as there are
incentives at play.

And of course, any rollback is quite disruptive to users, who generally expect pre-confirmed
transactions to be included no matter what.

## Atomicity & Synchronicity

_Atomicity_ for a set of operations means that either all the operations occur, or none of them
occur.

In the context of cross-chain messaging, atomicity can apply to either transaction inclusion or
successful transaction execution.

Atomic transaction inclusion is when transactions for multiple chains are included in their
respective chains at the same time. It is necessary to define "at the same time" carefully, as the
chains might not have the same block time, or block production can be delayed (the block time might
not correspond to the wallclock time of actual block production).

Atomic transaction execution is when transactions for different chains are successfully executed on
their respective chains. In practice, this generally also needs to happen at the same time otherwise
it could cause unwanted rollbacks, though certain designs propose to lock part of the state to avoid
this happening, which can make this requirement somewhat looser.

We use _synchronicity_ in the sense used in many programming language: an asynchronous operation is
fired off and its result is not immediately available, whereas a synchronous operation is one whose
result we wait on.

Based on these definitions, we can distinguish five atomicity models:

- Non-atomic (asynchronous) execution (asynchronous message passing)
- Atomic inclusion
- Non-atomic instant asynchronous execution (non-atomic synchronous execution + atomic inclusion)
- Atomic asynchronous execution
- Atomic synchronous execution (aka "universal synchronous composability")

Note that non-atomic synchronous execution is impossible by definition, so we'll just say
"non-atomic execution".

So far, this report has assumed non-atomic execution, which is less constrained than the atomic
execution models. Every constraint that applies to non-atomic execution also applies to atomic
execution.

Atomic inclusion is a bit different, because it does not guarantee the success of any of the
transactions, only that they will all be included. As such it is not a messaging model at all.

If one combines atomic inclusion with non-atomic execution, one gets non-atomic instant asynchronous
execution. This model is different from the atomic execution models because the success of the
action triggered by message delivery is not guaranteed. Indeed, a message is often an instruction to
perform an action on the destination chain (e.g. buying an NFT), and that cannot be guaranteed to
succeed (for instance if the NFT has already been sold to someone else).

On the other hand, atomic execution guarantees that if the action on the destination chain is
unsuccessful, then the transaction on the source chain won't be executed either.

The canonical example for this is a transaction that swaps fungible tokens on the source chain (say
USDC to ETH), bridges the swapped token over to the destination chain, and then uses the bridges
token to buy an NFT on the destination chain. In this example, the message sent to the destination
chain is a transaction that transfers/mints the second token on the destination chain, and uses it
to purchase an NFT. If the NFT cannot be purchased, we don't want the token swap and the bridging to
occur.

In atomic asynchronous execution, we simply send the message but can't get data back from the other
chain. In atomic synchronous execution on the other hand, we wait for the result of the message on
the destination chain, then can resume execution on the source chain using the result. We're
actually making a kind of function call instead of just firing a message.

Both type of atomic execution can further be differentiated based on their depth and breadth
characteristics. For depth, can message delivery trigger the sending of further messages? And how
far can this recurse? For breadth, can a single transaction trigger the sending of multiple
messages? To multiple different destination chains?

For atomic execution, our discussion will focus on the simplest case — atomic asynchronous execution
with a single message sent to a single chain and no recursion. As we will see, this is plenty hard
enough. We will then add some qualifications for the other cases.

For a good discussion on how atomicity relates to other topics in blockchain engineering (that we
really don't have time to cover in this report), I recommend reading ["We're all building the same
things"][same-thing] by Jon Charbonneau.

[same-thing]: https://dba.xyz/were-all-building-the-same-thing/

### Message-passing vs Bundling

So far we've discussed bridging in terms of message-passing: a transaction is sent on the source
chain which sends a message to the destination chain, usually by writing into some kind of inbox on
the source chain, then the message is relayed to the destination chain

A different way to achieve bridging is to create an atomic bundle containing one transaction for the
source chain and one for the destination chain, where all transactions that must be included or
successfully executed on their respective chains.

Atomic inclusion, _requires_ the use of bundles.

Atomic execution can work with message passing or bundles. The message-passing model is more
powerful, because it can dynamically adapt the message being sent based on the state of the source
chain — the message doesn't need to be known before executing on the source chain.

Nevertheless, while discussing execution we will use the concept of bundle to simplify our
explanations, adding qualifications wherever necessary.

### Atomic Inclusion

As written above, atomic transaction inclusion is when transactions for multiple chains are included
in their respective chains "at the same time".

Because atomic inclusion does not guarantee the successful execution of the transactions, it is not
sufficient to enable safe bridging, but can be used to make bridges faster by lowering the latency
to 0 (what we previously called "non-atomic instant execution"). They also enable better extraction
of cross-chain MEV by lowering the risk for MEV searchers, enabling for instance atomic cross-chain
arbitrages.

It's important to note that, just like how atomic inclusion does not ensure safe bridging, it also
does not by itself remove finality risk. A dedicated mechanism or a fast-finality DA layer is still
required.

#### Shared Sequencing

The main way to implement atomic inclusion is to use shared sequencing. In shared sequencing,
multiple chains (usually rollups) share a common sequencer, which is responsible for ordering
transactions on these chains. Because propagating and ordering transactions is a lot cheaper than
block production (it does not require execution nor keeping the state of the chains), shared
sequencing can scale to a vast number of chains.

Existing shared sequencing solution include Astria (using Celestia as the DA layer), Espresso (using
its own Tiramisu DA layer), NodeKit (using their own custom DA layer), and Radius.

#### Builder-Proposer Separation (PBS)

Another conceivable way to implement atomic inclusion is to rely on a proposer-builder separation
(PBS) system for rollups. In such a system, there is a set of proposers selected to propose blocks,
and a set of block builders that proposers delegate the building task to (this is usually done to
optimize MEV extraction).

With such a system, if a proposer is selected to build blocks in multiple rollups, he can delegate
to a builder that will be able to perform cross-block atomic inclusion.

The main limitation of the PBS system is that atomic inclusion is only possible whenever a proposer
gets to propose blocks for multiple rollups, or alternatively when a builder is selected by multiple
proposers for different chains. The availability of atomic inclusion is therefore not guaranteed.

Shared sequencing could also adopt a PBS system — while this do not yield additional benefits for
atomic inclusion, such a system could be used for atomic execution. This is not a silver bullet, as
we will discuss later. In the case where a single builder is selected to build all the blocks, see
the [Running Multiple Nodes](#running-multiple-nodes) section, and if not, refer to the [Shared
Validity Sequencing (SVS) + PBS](#shared-validity-sequencing-svs--pbs) section.

#### Reclaiming Safety

In both cases, the atomic inclusion guarantee will depend on the properties of the protocol (shared
sequencing or PBS). These are generally distinct from the security properties of the chains
themselves, and thus add new safety assumptions. These will generally be a bespoke consensus
(similar to validator bridges, or fully-fledged blockchains) or an economic guarantee (similar to
slashing bridges).

There is a way to reclaim the safety property however, by implementing shared validity sequencing,
as discussed in [the section on finality](#finality). This ensures that the atomic inclusion
guarantee is respected, as blocks in different chains containing transaction from an atomic bundle
reference the same Merkle root of messages. If one of the blocks doesn't finalize, then none do
(shared fault proofs). This also gives you non-atomic instant asynchronous execution.

Note that shared validity sequencing is not expressed in terms of bundles of transaction, but of
transactions in a source chain adding messages to an inbox that gets Merkelized. However, the design
can work with bundles too, it just requires Merleizing the bundles and making sure the Merkle root
is included on both the source and destination chain, and that bundled transactions are checked
against it.

However, shared validity sequencing relies on the shared sequencing system to truthfully propagate
the message Merkle roots between chains, making the shared sequencing system an enshrined validator
bridge (1). The shared sequencer cannot break safety (because of shared fault proof), but it can
temporarily get the chain to produce invalid blocks, which can cause the issues we describe in the
[Limitations and Attack Vectors](#limitations-and-attack-vectors) section.

(1) Normally, a shared sequencer only orders transactions, and therefore can't cause any safety
issue nor cause block reversion. By requiring it to attest to the messages (or bundles) Merkle root,
it could cause invalid blocks to be executed before they are rolled back by the shared fault proof.

#### Applications to the Superchain (Atomic Inclusion)

Shared sequencing can be used with the large Superchain model: new chains can permissionlessly opt-in
to the shared sequencer infrastructure, and since execution is not required, the scalability of
the shared sequencer is only limited by the speed of the underlying DA layer.

A PBS model without shared sequencing is also applicable, but builders become the gatekeeper of
atomic inclusion. We will have much more to say about this in the [Atomic
Execution](#atomic-execution) section, where the model coupling shared validity sequencing with PBS
is even more relevant.

### Atomic Execution

As discussed above, we consider atomic execution (synchronous and asynchronous) with a single
message sent to a single destination chain and no recursion.

The main drawback of atomic execution is that even in the atomic asynchronous execution model,
synchronous cross-chain node communication is required.

Let's take our swap-bridge-buy-NFT example from before (swap tokens on chains A, bridge them, use
them to buy an NFT on chain B). You must ensure that the NFT is still for sale on chain B to commit
the swap on chain A, which requires locking the relevant state on chain B.

#### Running Multiple Nodes

The simplest way to do this is to require all nodes for chain A to also run a node for chain B. This
doesn't scale well, as you are not splitting your resources between multiple chains. Effectively,
your set of chains is as resource-constrained as a single chain. This still has some advantages:
it's possible for every chain to be meaningfully customized (e.g. different gas tokens, execution
VMs, ...).

You could make a case for distributing this system, i.e. having nodes from different rollups talk to
each other. We don't think this is practical: the amount of synchronous networking communication
would introduce very significant latency to block production. Nevertheless, this seems what was
implemented in [this demo][radius-demo] by the Radius team.

[radius-demo]: https://ethresear.ch/t/cross-rollup-synchronous-atomic-execution/20193

#### State Locking

If we want scalability, things get harder. A naive solution entails locking state on both chains.
Given a bundle of transactions to execute atomically accross chains, determine the state that they
need to access (1), have both chain lock that state (no other transactions can write it until the
bundle resolves), then execute the transactions, commit them if they all succeed, or abort and
unlock the state otherwise.

(1) This is not a trivial problem itself, and in practice probably requires a form of strict state
access list.

This is not a very good solution: it requires multiple network roundtrips between the chains (and
potentially some coordination system sitting in the middle). While other transactions can continue
to be processed, locking the state can still be a significant bottleneck when the state is very hot:
for instance locking the balances of an AMM pool, or an NFT minting contract.

Interestingly, this is exactly the solution proposed by [Polygon AggLayer][agglayer]. The
documentation [used to][agglayer-old] mentions slashing actors that submit bundles that they
knowingly know will fail. But if that is an issue, we also need to consider that there are common
scenarios where most transactions fail, like NFT mint events, or just period of high price
volatility. If the system is restricted to transactions that always work, it's not more powerful
than atomic inclusion. They also mention griefing by locking a lot of state, but fail to touch on
the issue of locking _valuable_ state.

**CORRECTION:** The discussion of AggLayer in this document is based on previously published
documents. I have since then been informed that the design changed and AggLayer will not feature
state locking. At the time of publication, there isn't any documentation available that would enable
a meaningful discussion of AggLayer. Everything that is said about the AggLayer does however apply
in full generality to state locking system, and as such is still relevant.

[agglayer-old]: https://web.archive.org/web/20240228124632/https://docs.polygon.technology/learn/agglayer/
[agglayer]: https://web.archive.org/web/20240815013104/https://docs.polygon.technology/cdk/agglayer/overview/

[Chimera chains][chimera] are an alternative that also uses state locking. In this case, the state
is partitioned into a "base state partition" and a set of "chimera state partitiosn" which is meant
to be combined with the chimera state partitions of other chains to create a new "chimera chain"
where cross-chain transactions can be conducted on the various chimera partitions.

Chimera chains present a more pragmatic solution to the problem at hand: instead of pretending that
locking any state is practical, smart contracts would deliberately portion off the state that can be
locked and written only by a specific chimera chain (it will still be readable by the base chains).
While this has interesting use cases, it does not allow full atomic execution touching arbitrary
state — our canonical example of bridging a token to buy an NFT is unlikely to work, unless some
NFTs are specially reserved to be sold only on a specific chimera chain.

Another question is who should be responsible for execution on the chimera chains. In the context of
the Anoma ecosystem — where Chimera chains were proposed — the answer was to use an intersection of
the validators of the various chains participating in the chimera chain. This could be extended to
rollups by having all rollups (i.e. their sequencer) "run a node" for all the chimera chains they
participate in, and having them rotate the block proposer for the chain.

This would necessarily be a permissioned system, as it is unlikely sequencers would want to run a
chimera chain with an unknown rollup who could grief them by comitting a ton of bogus state or
transactions to their chimera chain.

[chimera]: https://anoma.net/blog/chimera-chains

#### Crossbar System

An alternative is to colocate the execution of the atomic bundles in a "crossbar system" as
[proposed here][crossbar]. This could part of the L1 chain, or a different chain (or rollup) that
all participating chains are expected to follow (i.e. implement an embedded bridge for). This
crossbar system would perform stateless execution of atomic cross-chain bundles.

[crossbar]: https://archive.devcon.org/archive/watch/6/rollups-shards-and-fractals-the-dream-of-atomically-composable-horizontal-scaling/?tab=YouTube

The crossbar system receives atomic transaction bundles, along with state access lists for each
transaction, as well as state proofs for each piece of accessed state, made against state roots. It
then outputs a set of transactions that each participating chain _must_ include at the start of
their next block.

There are again no silver bullet here, and there needs to be a bridging mechanism for the crossbar
system to receive the state roots. A good option is to use the cross-chain contingent blocks system
discussed in [the finality section](#finality). Note that, for rollups, the DA-mediated version of
that system can only be applied here if the DA layer is at least as fast as rollup blocks.
Alternatively, we could restrict cross-chain transaction to only occur in one out of X blocks for
slower DA layers.

This system particularly shines with rollups, because the cross-chain transactions can be posted to
the DA and are considered part of the canonical chain. This means that if a sequencer ignores them,
the state roots it would derive from executing further transactions would be considered invalid and
would fail to prove. This is similar to the "force inclusion" mechanism used by rollup to force
sequencers to periodically include transactions posted directly to the DA layer (which prevents
total transaction censorship by the sequencer set).

This system still has the property of cross-chain contingent blocks that if one of the posted
blockhashes is invalid, then all blocks in the other chains contingent on that blockash must be
rolled back as well. Because the crossbar system creates transitive dependencies between chains, it
probably means that most or all chains using the crossbar system need to be rolled back.

The main drawback of the crossbar system is that it sits in the execution path of all the chains
that use it: to produce a block, they must wait for the crossbar rollup to finish processing, which
reduces chain throughput.

The multi-node solution and the PBS solution (that we'll present right after) have a weaker version
of this problem: whenever a cross-chain transaction is processed, the block building for all the
chains that the transaction touches is temporarily interrupted. The crossbar rollup solution is
worse, because it blocks all the rollups while processing _all_ the cross-chain transactions.

The crossbar system helps with scaling a cross-chain system but does not fundamentally solve the
issue: by delegating the execution of all cross-chain transaction to a single system (the crossbar
chain or rollup), the cross-chain ecosystem is able to scale in proportion to the relative volume of
cross-chain transactions compared to regular transaction.

For instance, in a system with 10 rollups, the naive solution would have every chain's node operator
operate nodes for all 10 chains. If no more than 10% of the transaction on every chain are
cross-chain, then adding a crossbar rollup to the mix enables the same result, but only requires
running a node the rollup of interest and a node for the crossbar rollup.

We note that some state locking solutions share this issue as well.

#### Shared Validity Sequencing (SVS) + PBS

Finally, we can use [shared validity sequencing (SVS)][svs] to ensure that the atomic execution
guarantee is respected.

SVS combines shared sequencing (presented earlier in the [Shared Sequencing](#shared-sequencing)
section) with a mechanism to get around the finality delay (presented earlier in the [Shared
Validity Sequencing (Dependency Mechanism)](#shared-validity-sequencing-dependency-mechanism)
section): by Merkelizing the sent messages on the source chain, we can guarantee that if the
destination chain includes this Merkle root, a dependency on the source chain is created, causing
the destination chain to roll back if the source chain roll back via shared fault proofs.

We've also seen in the [Atomic Inclusion](#atomic-inclusion) section that SVS can accomodate atomic
inclusion, and therefore non-atomic instant asynchronous execution.

We can also modify SVS to obtain atomic execution, by modifying the contracts on the destination
chain such that messages are only added to the Merkle tree there whenever the action they trigger is
successful. If that is the case, then the message Merkle roots can only be similar on the source and
destination chains if both sides of the cross-chain transaction are successful.

However, creating blocks that guarantee this property require actually executing the blocks. This is
where Proposer-Builder Separation (PBS) comes into play: block creation is auctioned off to
sophisticated block builders running nodes for the various chains. These builders would only a
message to be sent on a source chain if they can verify that the action caused by message delivery
succeeds on the destination chain.

Builders are incentivized to be honest, as dishonesty would cause the blocks to be rolled back via
the shard fault proofs and their profits to be lost. It might also be desirable to add a bond and a
slashing condition, as builders could profit from the kind of attacks described in the [Limitations
& Attack Vectors](#limitations-and-attack-vectors) section.

The drawback of the SVS + PBS solution is that it offers no guarantees as to the availability of
atomic execution: builders will need to set up nodes for every chain for which they wish to build
cross-chain atomic transaction. Forcing them to do so means ensuring censorship resistance of
cross-chain transactions. Plain censorship resistance is already a hard problem (2) but the
cross-chain nature of things makes this even harder as one would have to prove that the cross-chain
transaction would have succeded to establish censorship.

SVS + PBS creates centralization, with consequences for censorship resistance. On Ethereum, [three
builders][mevpics] produce most of the blocks on Ethereum (because they are better at extracting
MEV). If a similar situation arises in the cross-chain system, then maintaining
censorship-resistance (not only for cross-chain transaction this time) becomes an issue, just as it
is on Ethereum.

[mevpics]: https://mevboost.pics/

Another issue with SVS + PBS is that it is hard to verify the work of builders. Nodes for a specific
rollup are not able to easily verify whether the messages that are part of an atomic bundle are
actually associated with successfully executed transactions on other chains. They must either trust
other nodes for these remote chains, or run nodes for all the chain that the rollup can interact
with. Using zero-knowledge proofs of execution is the only way to avoid this issue that is both
scalable and trustless.

(2) See for instance the work on inclusion lists for Ethereum ([one][inclusion1],
[two][inclusion2], [three][inclusion3]) to understand some of the complexity of the task.

[inclusion1]: https://ethresear.ch/t/no-free-lunch-a-new-inclusion-list-design/16389
[inclusion2]: https://ethresear.ch/t/unconditional-inclusion-lists/18500
[inclusion3]: https://ethresear.ch/t/inclusion-list-eip-7547-end-to-end-workflow/18810

Let's mention here [NodeKit Javelin][javelin] ([docs][javelin-docs]), which provides a cross-chain
builder implementation. Currently, NodeKit does not have a shared re-org mechanism, opting instead to
have builders post a stake that can be slashed in case of misbehaviour. While this form of economic
security isn't extremely scalable, the project has the merit of being actually implemented. There is
an idea to add validity proofs to the system, which could form the basis for a shared re-org
mechanism.

[javelin]: https://x.com/nodekitorg/status/1775161664645578808
[javelin-docs]: https://nodekit.gitbook.io/nodekit-documentation/architecture/javelin

#### Applications to the Superchain (Atomic Execution)

None of these solutions work really well in the large Superchain model.

- Running all nodes is obviously very permissioned and would only work for a relatively small amount
  of chains.
- Locking state also requires permissioning because of the potential for participating chains to
  grief the system.
- The crossbar system is susceptible to rollbacks if a single chain posts invalid blockhashes.
- In SVS + PBS, the builders act as gatekeepers and do not guarantee the availability of atomic execution.

In practice, the crossbar and SVS+PBS systems share the same issue that prevents them from being
totally permissionless and thus work in the large Superchain model: a new unknown rollup could
include malicious code allowing it to roll back the chain under some conditions, which would cause
all other rollups to also roll back.

We already discussed one possible solution in the [Applications to the Superchain (Atomic
Inclusion)](#applications-to-the-superchain-atomic-inclusion) section: if we can guarantee that a
rollup does not include any custom code and only makes customization choices among an audited set of
options (in a way that can be verified on L1), then these can be more easily trusted. While this is
hopeful, it is also rather sad, as experimentation is one of the best reason to launch a new rollup.

Finally, all advanced solutions can theoretically become permissionless by having some actors post a
bond (bundle submitters for state locking, sequencers for the crossbar system and SVS+PBS), but that
bond must be commensurate to the potential damage (liveness failure for state locking, rollbacks for
the crossbar system). Given that all chains are affected, the bond will necessarily be substantial
and smaller chains will be priced out.

All the models are easily amenable to the small and graph Superchain models.

#### Domain-Specific Solutions

None of the solution presented above is without trade-offs. But it is conceivable that by limiting
ourselves to certain use-cases (instead of general message passing) we could achieve atomic
transactions without paying too high a cost. These domain-specific solutions could come with their
own architecture or be built upon a general atomic execution solution, alleviating its pitfalls.

In particular, the main use-case for cross-chain transactions is token (both fungible and NFT)
bridging and swapping. Think of our running swap-bridge-buy-NFT example here. I believe these
transactions can be made atomic without too much downside.

We can outline a solution for this that leverages synchronous state locking, as long as locking is
only required on the destination chain.

The main issue of state locking is lock contention when multiple chains are trying to write to the
same state. We can alleviate this issue by using escrow contracts holding tokens on chain X that we
wish to exchange for some other tokens on chain Y.

The escrow contract is set up in such a way that the buyer's token are locked for a fixed period of
time, and within this period, the only way to withdraw the tokens is for the escrow contract to
receive a message from chain Y indicating that the requested tokens have been received there. A
standard contract could be set up on every chain that enables sending token balances to other
chains.

When using this setup with state locking (requiring locks on only one chain), it becomes possible
for a "solver" (aka liquidity provider) to acquire the requested tokens on chain Y, send them to the
buyer, then call the balance-sending contract to message chain X, signaling that the buyer's
balance on chain Y has now been updated with the requested tokens.

The one-sided locking property is important: it enables the solver to use contentious state on chain
Y (e.g. an AMM pool, an NFT marketplace) to acquire the token without incuring lock contention. On
chain X's side, the escrow contract is not contended at all, as the only people trying to write to
it are the solvers trying to fill the request — only one of which may succeed anyway.

Interestingly, atomicity protects the solver here. If we use the same setup without atomicity (only
asynchronous message-passing), then two solvers could fill the buyer's order (sending him tokens on
chain Y), but only the originator of the first message to arrive on chain X would collect the tokens
from the escrow contract.

Our solution is still susceptible to griefing attacks by submitting cross-chain transactions that
take the lock on the escrow contract but fail, which is a problem to be solved separately (e.g. by
letting user own locks, and granting permission to use them via a signature).

This solution might be implementable on Polygon's AggLayer, if it allows for one-sided locking and
can solve the griefing issue.

One domain-specific project that seem to implement something very similar to this proposed solution
is [OneBalance][oneb], which enables cross-chain transfers and swaps by deploying a smart contract
wallet for the user on every chain of interest. Instead of escrow contracts, it uses "resource
locks" to lock tokens earmarked for an order. It however does not exploit atomicity and as such has
to rely on extra security assumption or at the very least, on economic security to avoid griefing
attacks (1), depending on the implementation of what they call "credible commitment machine" (which
is yet to be fully determined).

(1) e.g. if OneBalance lets solvers "lock the resource locks" by announcing they will fullfill an
order, which can cause griefing without an economic incentive.

[oneb]: https://docs.onebalance.io/

### More Powerful Models

As explained before, we can go further than the plain asyncronous atomic execution.

In atomic synchronous execution, we wait for the result of the message on the destination chain,
then resume execution on the source chain. In the state locking solution, this requires more
roundtrip between chains, causing state to be locked for longer. In the crossbar solution, this
means that more "legs" of the transaction have to be prepared in advance, which isn't a huge deal,
but one will have to be careful with the costs that a single transaction on a source chain can start
incurring (both to the source and destination chain). SVS+PBS is also capable of this, although it's
once again guarded by economic incentives.

Expanded depth (enabling message delivery to trigger the sending of further messages, whether
synchronous or asynchronous) has the same effects.

Expanded breadth (enabling a single transaction to trigger the sending of multiple messages to the
same or different chains) is again the same, excepted that if these messages are sent in parallel,
then latency won't be increased for the state locking solution, and enable more parallelization for
PBS builders.

## Messaging Formats & Interfaces

Our discussion so far has mostly stayed high-level, but let's now dive into the nitty-gritty of
message passing between chains.

### Landscape

By necessity, arbitrary message passing requires smart contracts on both the source and destination
chain, sometimes designated as respectively an "outbox" and "inbox". A few standard have been
proposed for this purpose, listed here in chronological order:

- [ERC-5164: Cross-Chain Execution][5164]
- [ERC-6170: Cross-Chain Messaging Interface][6170]
- [ERC-7533: Public Cross Port][7533]

[5164]: https://eips.ethereum.org/EIPS/eip-5164
[6170]: https://eips.ethereum.org/EIPS/eip-6170
[7533]: https://eips.ethereum.org/EIPS/eip-7533

Besides proposed standard, most bridges, rollups, and other interoperability projects have their own
messaging interface. There are too many to list but let's mention a few that have the vocation to
be pluralistic (i.e. not tied to any specific provider). This list is not intended to be exhaustive.

- [Hyperlane's General Message Passing (GMP) Interface][hyperlane]
- [Multi Message Aggregation (MMA)][mma]
- [Hashi][hashi]

[hyperlane]: https://docs.hyperlane.xyz/docs/reference/messaging/messaging-interface
[mma]: https://github.com/MultiMessageAggregation/multibridge
[hashi]: https://crosschain-alliance.gitbook.io/hashi

Let's also mention Optimism's [bridging spec][op-bridge] and [interop spec][op-interop] as examples
of messaging standards between L1 and L2, and accross L2 (respectively)

[op-bridge]: https://specs.optimism.io/protocol/bridges.html
[op-interop]: https://specs.optimism.io/interop/overview.html

Finally, some standard or proposed solutions focus specifically on bridging tokens, for instance:

- [ERC-7281: Sovereign Bridged Token (aka xERC20)][7281]
- [Optimism Interop Token Standard (SuperchainERC20)][superchainerc20]

[7281]: https://github.com/ethereum/ERCs/pull/89/files
[superchainerc20]: https://specs.optimism.io/interop/token-bridging.html

Despite the proliferation of standards and solutions, none has come close to becoming a de facto
standard nor even gain significant traction, with perhaps the exception of xERC20 (though its
success is still moderate). We will speak more about it later.

### Design Considerations

We could go into the minutiae of each proposal and how they compare, but I don't think that would be
a wise use of our time. Let's instead look at general considerations for message passing interfaces,
trade-offs.

As mentioned earlier, interfaces for arbitrary message passing typically have something analogous to
an outbox on the source chain, and an inbox on the destination chain.

The messages posted to the outbox need two have three usual transaction properties: a sender (filled
automatically), a destination address and some data ("calldata"). There are multiple ways that the
data can be stored or even represented:

1. It could simply be posted as-is (or potentially compressed) and stored in the outbox.
2. Alternatively, only a hash of the data can be stored in the outbox. The actual data can then either:
   1. be emitted in an event.
   2. live in the source chain calldata.
   3. not be posted onchain at all, and only communicated offchain to a relayer, or relayed by the
      submitter himself. It could also be temporarily stored on an alternative data availability (DA) layer.

The first alternative consumes precious blockchain storage space, which is usually permanently
occupied (at least until blockchains adopt data expiry).

The second alternative has consequences for messaging initiated by smart contract, as only the first
option (emitted in an event) makes it easy to retrieve the message by following only the source
chain. Storing the data in calldata is only suitable for messaging initiated directly by users and
not mediated by intermediate smart contracts (and retrieving the calldata does require some advanced
node data querying — not simple JSON-RPC calls). The third option requires extra coordination
mechanisms to be present.

Besides the sender, the destination address and the data, it is also necessary to specify the
destination chain. There are two main ways to do this: either specify the chain explicitly, or
deploy a separate outbox for each possible destination.

Another common field found in messages is a nonce (a sequence number, either unique to the sender or
globally unique), which prevents replay attacks (relaying the same message multiple times).

While preventing replay attacks is desirable, note that the mechanism to do so does not necessarily
need to live in the messaging interface: it is possible to implement it on top of a general
messaging standard without replay protection, by ensuring that any replayed messages would fail
(e.g. by providing nonces directly inside the data). This is the approach taken by the Optimism
bridge, which offer two interface layers: a low-level interface without nonces, and a higher-level
interface build on top, which has replay protection.

As a side note, it is also technically possible to leave out the destination chain from the
low-level layer, which allows for broadcast messages that can be sent to any chain. A secondary
layer can enable specifying (a) specific destination(s). This layer must be standardized, otherwise
relayers would have to understand application-specific data to know where to relay non-broadcast
messages.

Once the message is in the source chain's outbox, it must make its way to the destination chain's
inbox. Many standards have something to say about how message should be relayed, and how their
authenticity should be proven.

We have already spoken at length about these topics, including how the authenticity of messages can
be established, the safety and liveness characteristics, the risks of bridging faster
than the source chain's finality, as well as the relayer's role and their incentives.

More interestingly, some standard explicitly decide to eschew these considerations, and leave the
message verification to the implementation, or to an external contract. This is notably the case of
[EIP-5164][5164], [EIP-6170][6170], [Hyperlane][hyperlane] and [xERC20][7281].

While this makes things less "standard" (messages will be relayed and verified differently depending
on the source, destination, and inbox/outbox deployments), it offers more flexibility, which might
benefit their adoption.

In particular, it seemed to have worked well for xERC20 and Hyperlane. For xERC20, it is attractive
for the governing body of a token to be able to use multiple bridging providers (or to change
provider) without having to perform a token migration or a complex contract ugprade. xERC20 has the
added benefit that it enables a single token representation in the presence of tokens bridges from
multiple chains, instead of a multitude of non-fungible wrappers. Hyperlane is attractive to new
blockchains for essentially the same reasons: it allows received messages from many chains without
enshrining a bridge provider.

Note that these standards are not always incompatible: for instance it's perfectly possible to have
an xERC20 token, who authorizes minting from a Hyperlane inbox or MMA receiver, which itself
delegates to one or multiple bridge-specific interfaces.

Amongst the standards that have something to say about message verification, we can generally
distinguish those that want to prove messages against a state root from the source chain, and those
that expect messages to be relayed directly.

Sometimes, the root is not the chain's state root itself, but a "message root" specific to the
inbox. It is also possible to deliver messages in batches, and even to even include message from
multiple source chains in the same batch. See [EIP-7533][7533], which prescribes both of these
things.

### Token Layer

Let's talk a bit more about the specificities to token bridging.

Beyond everything we've talked about so far, tokens face an additional challenge in the form of
"path-dependency": whenever a token is bridged from a source to a destination chain, the token is
typically locked or burned on the source chain and unlocked or minted on the destination chain, in
what is typically called a "wrapper token" contract.

This works well when the source chain happens to be the chain where the canonical token
representation lives. However, trying to bridge between two other chains causes can lead to the
proliferation of mutually non-fungible token wrappers, some of which are "wrappers of wrappers".

This can be fixed by adopting a single token contract on every chain, burning the token on the
source chain and minting on the destination, however this approach (which isn't supported by most
bridge providers) is not without tradeoffs: the token contract on every chain is now susceptible to
the security risks of all the bridges that are allowed to mint into it.

[xERC20][7281] tackles this risk by enabling token governance to set rate limits on the token flow
from any given bridge. Of course, the classical solution to the "wrapper of wrapper" problem is to
route every token through its canonical chain, though the UX of this "solution" is pretty
horrendous. Another non-solution is introducing liquidity pools between all the wrappers. This can
be made to sort-of work in the presence of liquidity providers that take upon themselves the task of
rebalancing token liquidity accross chains.

The mint-and-burn into a single contract approach also doesn't support permissionless bridge
deployment. This is almost a fundamental trade-off, but it can be alleviated in a very homegeneous
chain and bridging environment like our small Superchain model: if it is known that all chains have
similar execution environment and that the bridges between any of these chains have guaranteed
identical security properties, it could be allowed to permissionlessly or automatically deploy
mirror token contracts on all of these chains. Note that this is not generally possible in the graph
Superchain model.

## Conclusions & Recommendations

This report has covered a lot of ground: we have established a bridge taxonomy focused on message
authentication mechanisms, and have analyzed their properties (safety, liveness, latency, incentives
& costs, and implementation complexity).

We have discussed how chain finality is the main obstacle to a bridging experience that is both fast
and safe, and highlighted one key property: to bridge faster than chain finality, a destination
chain must re-org whenever a source chain re-orgs. We have then discussed mechanisms to make this
happen (shared validty sequencing, cross-chain contingent blocks).

We have looked at how to tightly couple execution on the source and destination chains, with the
main prize being atomic synchronous execution (aka universal synchronous composability). We reviewed
the state-of-the-art solutions to the problem (state locking, crossbar system, SVS + PBS) and
explored the possibility of less general domain-specific solutions (like atomic token swaps).

Finally, we discussed the various messaging formats and their trade-offs.

Now, we offer some opinions, recommendations and wishes for the future.

### Bridging Mechanisms

There is no question that for cross-rollup bridging, proof-of-execution bridging (aka ZK bridging or
validity bridging) is the safest solution — it is the only trustless solution that works for
rollups, and has better latency characteristics than other near-trustless solution like fault proof
bridging.

However, their adoption has so far been hampered by the fact that generating proofs was still
relatively costly and slow compared to less secure bridges.

As we discussed in the [Latency](#latency) section (with a breakdown of proving times and costs),
the technology has been improving rapidly, and it is not unreasonable to expect it to match the
speed of DA layers within the next few years.

As proving time comes down to seconds instead of minutes, we can expect zero-knowledge bridges to
start blooming. At this point, focusing on other mechanisms feels like a mistake for ambitious
interop projects. Much to our chagrin, our bridge taxonomy is fated to become a historical
chronicle.

That being said, zero-knowledge proofs don't solve everything in interoperability.

### Latency

For one, for chains using the Poseidon hash for Merkleization, their proving time today is already
lower than most rollup's finality time. As we have pointed out, bridging faster than the source
chain's finality is not safe.

Besides waiting for Ethereum to move to single-slot finality — which could take years — the
alternative is to form network of rollups that re-org together.

We saw two ways of doing this: Cross-Chain Contingent Blocks (CCCB) and Shared Validity Sequencing
(SVS). These are essentially the same, the only difference being that CCCB commits to state roots
while SVS works on message roots. Continuing in the same direction, the Optimism interop protocol
does not even require commitments beyond the inclusion of the transaction themselves.

Any of those solutions are fine, though CCCB does not permit atomicity. Optimism's solution is the
only one that is actually being implemented, and its design is very thoughtful, notably the
realization of the graph model of cross-chain dependencies and an actual mechanism for shared
re-orgs via a shared fault proof. The next step there will be to extend the shared re-org mechanism
to also support zk/validity proofs.

There is one other alternative, which is to adopt fast-finality DA layers like Celestia. These avoid
re-orgs altogether. However, this is not a panacea either: DA layers do not add safety risks to
chains that use them, but — paradoxically — they add safety risks when a chain receives messages
from another chain using a different DA layer. If the source chain's DA layer is compromised and a
shared re-org mechanism is not implemented, then the destination chain faces a bridge safety
violation.

Fast proofs of execution can help here too: if the proof generation time can match the DA block
time, then fast trustless bridging becomes possible.

### Implementation Complexity

The other problem of proof-of-execution is that they have the highest implementation complexity of
all mechanism. Some will disagree and say that fault proofs are complex too. Fair enough, but the
highly specialized knowledge required to be able to audit a zero-knowledge protocol is a
particularly onerous barrier.

Because of this, it is probably a good idea to couple a zero-knowledge bridge with a validator
bridge, requiring both a valid proof and a quorum of signatures. Both types of bridges are fast, and
having the validator bridge ensures that creating a safety violation in the bridge would require
both compromising a majority of validators and exploiting a bug in the fault proof — an unlikely
scenario.

### Atomicity

When it comes to atomicity, the waters are muddier.

Do we even need atomicity? That itself is unclear. Most of the use cases invoked are financial in
nature and involve conducting complex transactions without putting capital at risk (like in our
running swap-bridge-buy-NFT example). "Liquidity fragmentation" is often pointed out as the reason
why atomicity is necessary, but it doesn't seem crazy to me that this problem could be satisfyingly
solved (if not quite as elegantly) via fast non-atomic message passing. When capital can safely move
between chains at the speed of block production, atomicity is merely a cherry on top.

But okay, atomicity _is_ cool, and it fullfills one of the original promises of Ethereum: it gives
us the world's computer, a uniform computing layer. It's a worthy goal to strive towards. How then?

We have looked at three solutions: state locking, crossbar system, SVS + PBS. These all have
trade-offs.

State locking struggles with lock contention: if multiple chains want to access the same state, a
lot of synchronous network communication will be required, adding latency. Locks being leased for
longer period causes a degration of liveness for operations that wish to manipulate this state. They
are also open to griefing attacks where a malicious actor keeps requesting lock for failing
transactions.

The crossbar system helps with scaling (by not requiring an actor to run every node) but reduces the
throughput of all rollups as a result of the need to wait for the crossbar rollup to finish its
work.

SVS + PBS (Shared Valididty Sequencing + Proposer Builder Separation) (1) relies on the presence of
economically-incentivized block builders to enable atomic execution. This could work well but makes
the block builders into powerful gatekeepers of interoperability, and possibly adds a centralizing
force with consequences for censorship-resistance.

There seems to be no silver bullet for general atomic execution at this stage, and we believe that
domain-specific solutions are more likely to be adopted in the near future. Those could have their
own architecture or alleviate the issues in one of the general architecture. In particular, it seems
clear that some form of state locking can be made to work quite well for cross-chain token swaps.

However, if I had to issue a recommendation for the fully general atomic execution, I would go with:

- The crossbar system in the small Superchain model, avoiding centralized actors at the cost of
  slightly reduced throughput compared to SVS + PBS.
- SVS + PBS in the graph or large Superchain models, though noting that block builders become the
  gatekeeper to chain participation. The crossbar system doesn't supor these models.

(Until proven otherwise, state locking is fundamentally flawed as a general approach
because of lock contention and griefing vectors.)

(1) Optimism's interop protocol can be subtitude to SVS here, as it enables shared re-orgs and allow
concurrent cross-chain transaction execution at the same height/time.

### Summary

After expanding so many words on the topic, the conclusion to this report is deceptively simple:

1. Zero-knowledge proofs enable trustless bridging and are poised to become the superior form of
   bridging in the medium-term, as proving time and cost further decrease.

2. Time-to-finality remains an issue in any case to enable fast bridging. Adopting a common DA layer
   with fast finality alleviates this issue. Since Ethereum is the de facto DA of most rollups, this
   points to single-slot finality being a priority. Absent a fast-finality DA layer, a mechanism for
   shared re-orgs must be used. [Shared validity sequencing][svs] and the [Optimism interop
   protocol][interop-specs] are both good solutions to that problem.

3. Every known solution for atomic transaction execution has significant trade-offs. However,
   domain-specific solutions, in particular for token swaps seem achievable with minimal downsides.
   We showed that this could be built on top of a general state locking mechanism. Domain-specific
   architectures might also be possible.

With this, we conclude this report.

If you've read this far, thanks a lot for your attention! You're a real research chad.
