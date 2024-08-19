# Analysis Thoughts

This document offer some thoughts regarding the analysis phase of the research.

Here's a recap of the questions from the [first direction document](direction.md):

**Question 1**: Can we design a trust-minimized native bridge between blockchains (or rollups) that have
been designed from the get-go to support such a bridge? What would the properties of such a bridge
be, and what are the trade-offs in this design space?

**Question 2**: Can native bridges (preferrably trust-minimized, but not necessarily) provide fast
delivery, atomic message-passing, or atomic execution? What are the trade-offs in the design space?

**Question 3**: What bridging models exist? What are their design, properties, and trade-offs? Can
we reuse or improve them (preferred) — or design a new one that fits our needs? Pay particular
attention to the developer and user experience that these models provide, or enable.

**Question 4**: What are the developer and user experience of existing bridges? Can we identify
superior approaches? What are the trade-offs?

**Question 5**: How best to solve the token-pathing problem? What are the security assumptions and
risks? Can we leverage existing token standards (e.g. [ERC-7281 /
xERC20](https://github.com/ethereum/EIPs/pull/7281))?

**Question 6**: Can we flesh out a full incentive-compatible model for action-request bridges?

**Question 7**: How expensive would this kind of bridge be for users such that a robust network of
relayers can be maintained? Are there any existing network of relayers that could be leveraged for
this task?

**Question 8**: How do the solutions uncovered in answer to the previous questions stack up in terms
of implementation complexity, architecture complexity (e.g. the number of moving pieces), worst-case
assumptions, etc...

# Current Thoughts

Here are some thoughts after reviewing some of the existing materials:

**Question 1 (trust-minimized bridging)**: All signs point to either light clients or zero-knowledge
proofs. ZK proofs of state transitions (execution) are the end game but have limitations. Light
client and ZK proofs of consensus are feasible, but have limited scope: they can't handle Ethereum
or rollups. Sharding (or its close cousin, enshrined rollups) can provide a middle-way, but only for
systems that start integrated.

Regarding ZK proofs of execution, Polygon
[claims](https://twitter.com/jbaylina/status/1603144831978831872?t=QyGfOu3Htsc_42c8pw5Szw) to have a
proving system that is fast enough to use in practice (2.5 min proving time on 7$/hour AWS hardware
for 10M gas — Ethereum blocks are 15M/30M gas target/limit every 12s, and Optmism blocks are 5M/30M
every 2s). This relies on running a zkEVM stack, and such stacks are (from the assessment of
independent researchers) really hard or even impossible to run for third-parties (key components not
open-source, etc). In the medium-term using such proofs + a multisig system (to ward against zk
implementation bugs) seems like the ideal option if designing a system from scratch.

Light clients proofs of execution are not feasible (as a chain needs to take the burden of rerunning
a whole other chain on its VM).

ZK proofs and light client proofs of consensus are feasible, but have limited scope: only fairly
simple system (e.g. Tendermint) can be supported. A PoS system like Ethereum's is too expensive and
requires keeping a lot of state. There is still no implemented zk-proving system for Ethereum PoS.
Rollups have no consensus, so this option isn't available to them either.

Sharding / enshrined rollups techniques could be used to enable effective proofs of consensus for
rollups. Most of the complexity can be pushed to (a) shared chain(s) that everyone must run (e.g.
Ethereum + a coordinator rollup). This shared system is responsible to attribut evalidators to each
rollup, the consensus between each rollup's validators could then be proven via a
proof-of-consensus.

We will talk about shared sequencing in the next section, but without the other solutions we just
talked about, it appears to be incompatible with trust-minimization, short of re-introducing
consensus to rollups.

**Question 2 (native bridges):** This is only just an extension of quesiton 1. ZK-proofs and light
client do not need to be native, as long as they operate on *finalized* blocks. For rollups, this
only works with zk proof of execution, and only with data that has been finalized on the DA layer.

If something faster than finality is required, then native systems are required, introducing
cross-dependency between the two chains (e.g. such that one reverts if the other reverts). For
action-request bridges (e.g. token bridges), it's possible to push the risk of reversion to relayers
and liquidity providers (who are compensated via a fee) — without going native.

Sharding / enshrined rollups are native systems by nature.

Shared sequencing is a native system, but it does not help with trust-minimization. However, it
could be a great help for the other properties, as dedicated cross-rollup builders could ensure
instant delivery, atomic message-passing, and atomic execution. The lack of finalization guarantees
could be acceptable if all rollups re-org with the DA layer.

Both sharding and shared sequencing are generally not permissionless (allowing for arbitrary
rollups/shards to be added to the system), for reasons of scalability and trust. A builder-based
shared sequencing solution could accept permissionless chains if there are economic incentives, but
the bridge will only be as secure as the weakest of the two bridge chains, and such a system could
not work within a unified asset model (cf. question 5).

**Question 3 (bridging models)**: Existing research is pretty satisfactory here, we'll build upon
some existing model. Fred is currently working on this. A special area of interest are the
message-passing interfaces of various bridges & standards. These generally look reasonable, and not
really complex, but we could probably point out some of their differences.

**Question 4 (bridging model UX)**: very much TODO

**Question 5 (token pathing)**: A simple way to solve token pathing is to lock a token on its native
chain, and emit a wrapped version on other chains, which can then be minted and burned on all those
chains. The issue is that this system is only as strong as its weakest link (i.e. the least secure
chain). In practice, such a system could end up looking like a "web-and-spokes" model, with a
trusted core web of chains (e.g. the Superchain as it seems to be conceptualized by OP Labs at the
moment), while other chains (spokes) would get a chain-specific version of the token that is not
fungible with the core web version. They will need to go back to a specific bridge on one of the web
chains in order to transfer to token somewhere else (most likely to some kind of liquidity hub
chain). Both in the web and at the spokes, rate-limits on transfers can be used to limit the damage
in case of system failure.

**Question 6 (action-request bridge)**: TODO but seems entirely possible, in fact such bridges
already exist, though they haven't been theorized in quite the same way.

**Question 7 (relayers)**: very much TODO

**Question 8 (complexity of solutions)**: Can't realy be done until the very end, when we come up
with recommendations.

# Next Steps

- Fred is working on question 3.
- I've put a lot of thoughts on question 1 & 2, so I'm eager to
  - get your feedback on my thoughts above
  - lead to effort to flesh these out
- Help greatly needed for question 4 — someone that has experience integrating bridges would be very
  welcome here. We'll need to peruse the docs of existing bridges (and pray to god they're not awful),
  but there are so many of them that being able to make a representative pre-selection would be
  extremely valuable.
- If someone has more ideas on question 5, that'd be welcome.
- Question 6 & 7: These two are pretty related, we mostly need more information from existing
  relayer networks, their difficulty and economics. If someone has more info here, that would be
  pretty welcome. We will probably need to inquire with these actors anyway to get their feedback.

I think this is pretty much "analysis". Beyond that we could go into something more constructive,
like sketching out an ideal cross-chain architecture that is both compatible with current realities
(esp. regarding the superchain) and future-compatible (e.g. zk proofs of execution or even
sharding). This would include high-level descriptions of the components, their interactions, and
some code for the key interfaces.