# Research Direction

This document is written prior to the research, and is meant to document some starting assumptions
and area of investigations. No doubt that things will evolve as we get into the process.

- [RFP Statement](https://app.charmverse.io/op-grants/page-012176298170861077)
- [RFP Application](https://app.charmverse.io/op-grants/page-2498144202521042)

I will start with a simple blockchain & bridge security model, which will inform the rest of the
document.

Next I will formulate the fundamental questions that we need to answer in this research, presented
in an order that reflects the ease of presentation rather than priority. The questions center around:

- What bridging systems integrated in the blockchain system itself might look like.
- The potential of existing bridging models (roughly, interfaces).
- How weakening the fully-general message-passing model can help us in certain scenario, namely
  regarding token bridges.
- What the implementation and architecture complexity of the various uncovered solutions are.

----------------------------------------------------------------------------------------------------

## Chains & Bridges Security Model

I could write a 20-page essay on the topic. This is, by necessity, a "good-enough" simplification.

Every PoS blockchain is essentially a multisig: a set of validators sign off on a block, and the
real chain is the longest chain of blocks signed by a super-majority of validators. The same goes
from pretty much all blockchain variants, but not for PoW chains, nor rollups.

Rollups, on the other hand, share the security of the settlement (L1) chain, with the additional
assumption that either transaction data (all rollups) or state diffs (zk rollups) are available.
Optimistic rollups add the assumption that there is at least one honest validator (and in the final
vision, the validator set is permissionless).

We define cross-chain bridges as systems that allows message passing between two chains, i.e. a
system that enables a transaction on chain A to trigger (or at least authorize) a transaction on
chain B.

There are bridge designs that only enable the transfer of assets but not of messages. We will talk
about them later, but in this section we focus on fully-general message-passing bridges.

Cross-chain bridges can be classified as external or internal.

**External bridges** are not integrated in the blockchain system — they live at the application
layer. These are most of the bridges we are familiar with.

**Internal bridges** are integrated in the blockchain system — as far as I know this only includes
rollup bridges.

Cross-chain bridges can also be classified as trust-minimized or trusted.

**Trusted bridges** introduce additional security assumptions on top of the those of the bridged
chains. Ultimately, this is also always a multisig, no matter how bridges try to spin things.

**Trust-minimzed bridges** do not introduce additional security assumptions (outside of
implementation correctness — which is itself a big can of worms). Trust-minimized bridges include
bridges from chain A to chain B where the consensus (i.e., the "multisig" for PoS) is proven on
chain B. This could be done by means of a chain-A light client running on chain B, or via a
zero-knowledge proof of chain B's consensus. They also include rollup bridges.

Caveat: trust-minimized bridges are unable to handle some blockchain upgrades ("hard forks"), in
particular those that pertain to the consensus mechanism itself. (This is why they're
"trust-minimized", not "trustless".) This means there needs to exist a mechanism to upgrade the
bridge itself, hence introducing new trust assumptions. At best, the additional trust can be reduced
by including a time delay in the upgrade mechanism which allows funds to be withdrawn from the
bridge. The only exception to this are L1 → L2 rollup bridges, because the L1 chain is the ultimate
source of truth. The same isn't true in the reverse direction (L2 → L1).

Also note that some popular "light-clients" are not trust-minimized. For instance, light-clients
that use the Ethereum light-client protocol rely on signatures from the light-client commitee which
has [much weaker guarantees][altair] than the Ethereum consensus (but is much easier to verify).

[altair]: https://prestwich.substack.com/p/altair

How this taxonomy shakes out today:

- Most bridges today are external and trusted.
- Light client bridges like most IBC implementations are external and trust-minimized (+ caveat).
- Rollup bridges are internal in the L1 → L2 direction, external in the L2 → L1 direction, and
  trust-minimized (+ caveat in the L2 → L1 direction).

All of these provide *eventual delivery* of messages. They guarantee that a transaction can
eventually be made on the destination chain, but provide no time bounds (excepted of L1 → L2
bridging, where L2 transaction inclusion is bounded but the bound is large).

They also do not provide atomicity, which is the property that the transaction on the source chain
succeeds only if the transaction on the destination chain also succeeds. This is a very difficult
property to attain, and can only be achieved in full generality via internal bridges.

Any external bridge must also contend with possible re-orgs of the source chain, meaning that the
minimal safe delay for cross-chain transfers is the finality delay of the source chain. This isn't
an issue for L1 → L2 rollup bridges as the L2 reorgs when L1 reorgs, but the L2 → L1 direction is
actually much slower than the finality delay, as it needs to wait for the challenge period.

----------------------------------------------------------------------------------------------------

## What do we want? — Research Questions

In this section, I explain what we're interested in achieving, and what seems to be the important
questions to answer to help us figure out how to get there.

The various questions are presented in an order that eases presentation, and do not necessarily
reflect priority. There is a certain amount of cross-dependency between some questions anyway. For
instance, to know what a good messaging model looks like, it would help having an idea of how
internal bridges might work.

### Investigate Internal Bridges

Trusted bridges add security assumptions. Trust-minimized external bridges are actually really
trusted because of the need to support chain upgrades.

Caveat: most bridge hacks have not been due to the trust assumption, though the Ronin Network (Axie
Infinity) hack (624M$, #1 on the Rekt leaderboard) and the Multichain hack (126M$, #15 on the Rekt
leaderboard) have been due to the multisig being compromised. I do think that it is a reasonable
argument that such issues can be prevented by ensuring multisig decentralization and ensuring that
this process is transparent. This also shows that bridge implementation has deep security
implications and is not to be ignored, even though the ultimate trust model is identical.

**Question 1**: Can we design a trust-minimized internal bridge between blockchains (or rollups)
that have been designed from the get-go to support such a bridge? What would the properties of such
a bridge be, and what are the trade-offs in this design space?

Beyond trust-minimized bridging, we might want to go beyond the state of the art and achieve the
following properties:

- *Fast* bridging (faster than the finality delay) in the usual case in a way that is still safe.
- Go beyond eventual delivery message-passing and **guarantee** that the message will be delivered
  within a certain block span.
- Provide atomic message-passing — ensure the transaction on the source chain succeeds only if the
  transaction on the destination chain also succeeds.
- Provide atomic execution, which we can conceptualize as recursive atomic message-passing: a
  transaction on the destination chain resulting from atomic message-passing being able to itself
  trigger further atomic message-passing to the source chain (etc). Put differently, enable
  cross-chain transaction that can seemingly call back and forth between both chains.

**Question 2**: Can internal bridge (preferrably trust-minimized, but not necessarily) provide fast
delivery, atomic message-passing, or atomic execution? What are the trade-offs in the design space?

### Bridging Models & User Experience

Designing and taking advantage of internal bridges might be a long-term endeavour. In the
short-term, external bridges are and will be used. There is great interest from OP Labs (and I think
the Optimism community at large) to propose a "canonical" bridging model.

This "model" would take the form of a cross-chain message-passing format. It specifies the interface
for sending a message on the source chain, and the interface for message delivery on the destination
chain. It also specifies an interface for verification via the actual bridge provider. Finally, it
should probably say something of relaying messages, paying gas costs, replaying failed mesages, and
probably more things that I am not thinking of right now.

The goal there is to design this model in such a way that it is able to support existing external
bridges (ideally, most/all of them) but is able to be seamlessly upgraded to an internal and/or
trust-minimized bridge model whenever that becomes available. The same model would also enable
switching bridge providers if required.

Examples of existing models include the [deposits]/[withdrawals] model of the OP Stack, [EIP-7833
(Public Cross Port)][7833], the [Inter-Blockchain Communication protocol (IBC)][IBC] and Hyperlane's
[Mailbox/ISM model][hyperlane]. The OP Stack model is specific to that system, but the two others
allow plugging in different bridge providers. 

(Before anyone gets angry, this is not an exhaustive list, and there are definitely other models out
there, though for those I am aware of it is less clear that those allow arbitrary bridge providers.)

[deposits]: https://github.com/ethereum-optimism/optimism/blob/develop/specs/deposits.md
[withdrawals]: https://github.com/ethereum-optimism/optimism/blob/develop/specs/withdrawals.md
[7833]: https://github.com/ethereum/EIPs/pull/7833/files
[hyperlane]: https://docs.hyperlane.xyz/docs/protocol/permissionless-interoperability
[IBC]: https://github.com/cosmos/ibc/blob/main/spec/core/ics-002-client-semantics/README.md

**Question 3:** What bridging models exist? What are their design, properties, and trade-offs? Can
we reuse or improve them (preferred) — or design a new one that fits our needs? Pay particular
attention to the developer and user experience that these models provide, or enable.

**Question 4:** What are the developer and user experience of existing bridges? Can we identify
superior approaches? What are the trade-offs?

### Token Bridging & Action-Request Bridges

Bridging tokens is of course the most common use case for bridges.

Sometimes it is very straightforward: if a token is native to chain A, then bridging between chain A
and chain B can follow the lock-and-mint model.

If a token is not native to the source chain, things get difficult, as either the token needs to be
bridged back to its native chain to then be bridged to the destination chain, or it needs to be
"re-locked". This could lead to many version of the same token on the same chain, depending on the
"path" taken by the token to arrive there.

**Question 5**: How best to solve the token-pathing problem? What are the security assumptions and
risks? Can we leverage existing token standards (e.g. [ERC-7281 /
xERC20](https://github.com/ethereum/EIPs/pull/7281))?

Departing from a fully-general message passing model enables new possibilities. In particular, it
enables leveraging an existing A → B message-passing bridge to build a B → A bridge for
"permissionless actions".

In this document, we'll call *permissionless actions* any actions that can be undertaken on a chain
without requiring authentication from another chain. We'll call a bridge that only enable
permissionless actions an *action-request bridge*.

As an example, we can create a B → A token bridge as follows: a user puts in a request for bridging
on B, along with the funds to bridge and a fee payment. Liquidity providers then transfer
corresponding tokens on A to the user. They prove they did this via the A → B bridge, and claim the
locked funds & fee. This matches how the Accross bridge [fast
fills](https://docs.across.to/how-across-works/how-across-guarantees-transfers) work.

This is particularly interesting for rollups, because the canonical L1 → L2 bridge is fast, while
the canonical L2 → L1 bridge is very slow.

In general, an action-request bridge allows user to request any permissionless action on B by
putting up a reward on B. Relayers can perform the action on A and prove it via the A → B bridge to
claim the reward.

This model also enables faster bridging for permissionless actions by letting relayers take on
finality risk on a case-by-case basis. They can consider that a re-org after a certain delay or
whenever certain guarantees have been met is unlikely enough that they are willing to take on the
risk to capture the opportunity. This implies a model that rewards relayers for prompt execution.

**Question 6**: Can we flesh out a full incentive-compatible model for action-request bridges?

**Question 7**: How expensive would this kind of bridge be for users such that a robust network of
relayers can be maintained? Are there any existing network of relayers that could be leveraged for
this task?

Note: answering questions 1-4 might uncover a need for relaying there too.

In the case of token bridging, we note that there are significant issues with liquidity in this
model: relayers must maintain inventory on all possible destination chains. In a world of many small
chains (e.g. appchains), this is obviously a problem. This underscores that the message-passing
bridges need to be fast enough to enable prompt liquidity rebalancing whenever demand for certain
bridging routes picks up.

### Implementation Simplicity

As we mentionned before, implementation correctness is the major factor in bridge security. All else
being equal, we should favor simpler systems that are easier to understand, review, and reason
about. In some cases, it might be good to trade good theoretical properties for implementation
simplicity, as it translates into effective security.

**Question 8**: How do the solutions uncovered in answer to the previous questions stack up in terms
of implementation complexity, architecture complexity (e.g. the number of moving pieces), worst-case
assumptions, etc...

----------------------------------------------------------------------------------------------------

## Roadmap

### Step 1: Information Gathering

The first step is to gather as many quality resources as possible on the topic, while trying to
avoid needless duplication (we don't need every Medium article on the topic). Primary sources (e.g.
from Bridge docs and repositories) are preferred, but secondary sources are also welcome when they
shed light on particular aspects of the problem.

A few things we should try to collect:

- A list of bridge, and where we can learn about their properties.
    - If available, evaluations of them, or comparisons between them, especially with respect to user
      experience and developer experience.
- Existing investigations & implementations of internal bridge systems. This will include work on
  shared sequencing and execution sharding, amongst other things.
- Existing bridging models in the sense of question 3.
- Any kind of existing set of criteria or taxonomies for comparing and classifying bridges and
  similar system (e.g. the [L2Beat Bridge Risk Framework][risk]).

[risk]: https://gov.l2beat.com/t/l2bridge-risk-framework/31

Additionally, we should try to identify axioms that can guide or constrain the research. For
instance, in this document, I have posited the following things (amongst others):

- Trust-minimized bridges don't really exist in the presence of hard forks, though mitigations are
  possible.
- If safety is to be guaranteed, bridging speed is bound by the finality delay of
  the source chain, unless the destination chain re-orgs whenever the source chain re-orgs (e.g.
  rollups).
- Fully-general atomic message-passing is not achievable with external bridges.

We could also try to identify weaker hypotheses that we can try to disprove, or treat as true in the
absence of discomfirming evidence.

We can also try to provide our own tentative distinctions and taxonomies. For instance, in this
document I distinguished internal vs external bridges, trusted vs trust-minimized bridges, and
message-passing vs action-request bridges.

### Step 2: Analysis

Take the information gathered in step 1, and try to answer the questions posed in the previous
section. Some questions require synthesis, while other are more open creative endeavours.

In any case, we will need to try to articulate our answers around taxonomies and design trade-offs
(this is where the axioms from Step 1 come in handy).

### Step 3: Recommendations & New Designs

In step 2 we highlighted the various solutions and their design trade-off. In this step, we give an
opinion on the relative value of various properties, and consequently on which trade-offs are worth
taking.

This is also where we can try to flesh new designs that we might have come up with a little more.

----------------------------------------------------------------------------------------------------