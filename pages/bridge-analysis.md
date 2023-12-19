# Arbitrary Message Passing - Analysis

In this paper, AMBs are classified and analyzed on a more architectural basis. Projects implement specific architectures and depending on the implementation evaluations of key metrics can vary vastly. However, we will see that fundamental types of AMBs show strengths and weaknesses in different metrics which can not be turned upside down by a specific implementation.

## Design space of the analysis
Nowadays there exist lots of different types of bridges like token bridges, liquidity layers, optimistic bridges secured by underlying bridges or request-action bridges.
The suggestion of analysis solely focusses on "bottom layer" type bridges namely arbitrary message passing bridges (AMBs).
As described later, AMBs in the purest form are used to prove the state (or certain parts of the state) of other chains.
When there are two isolated chains, it can be easily seen that external actors are needed to provide data of different chains.
All other types of bridges mentioned above can actually be build on top of AMBs.

## Key metrics
We will have a look at different key metrics which are typically of interest when it comes to message passing between two networks.
Note, that we are not covering atomic message passing which would require fundamental changes in the underlying network architectures.
To date, all AMBs are not atomic and thus asynchronous. Which means there is no native way of binding two actions from two different networks together.
That is the reason, why cross chain bridges try to address this problem and bind the two actions together by introducing protocols on top, which are executed and followed by external actors (off-chain actors).

Key metrics to look at are: 

#### Safety
How safe is the protocol in order to prevent invalid or forged messages to be executed on the target side? How would an attack look like where a message is tried to be executed which was not sent originally.
What is the boundary or the threshold where it actually becomes viable for the trusted external actors to misbehave and forge messages.

#### Liveness
What does it take, to prevent the execution of valid messages? Can attackers stop the bridge from functioning or stop the execution of specific messages.

#### Latency (speed)
What are the latency aspects of messages from sending the message on the source chain until execution of the message on the target chain.


#### (Cost) efficiency
Cost efficiency describes the economic value needed to maintain the same level of safety, liveness and latency. While it is tightly coupled to the three above and not completely independent, it is worth to have a look at the economics, because it ultimately sets boundaries to fee structures and cost of providing AMB services.

#### Implementation complexity
Implementation complexity increases the likelihood of implementation bugs. Looking at the most relevant bridge hacks, one can see that the main reason was due to implementation bugs and not the underlying architectures.


## Messages
It should be mentioned that arbitrary message passing should not be confused with liquidity layers or request-action bridges.
However, those bridges could be built on top of general message passing architectures and serve as optimizations for specific use cases (i.e. instant token bridging by offloading latency to the market maker's settlement).


### Arbitrary messages
Since we do not know what data are send we must assume that it could contain a very high economic value. This leads to the fact, that whenever a message is validated and executed it is final and could potentially bring massive economic losses.
For this reason it needs to be taken into consideration when analyzing cross chain messaging. If the analysis is made for messages with very high economic value, the real metrics of the underlying protocol are uncovered.
IOW, arbitrary message passing effectively means proving the state of another chain.

In the time of the analysis, a concrete message example is chosen for a simpler understanding.

### Message example
In the example message, the use case is the following. A user wants to synchronize the owner key of his safes on different chains. He only wants to change the owner key on his preferred main chain (Chain A), and from there messages are being sent to the other chains that the owner should be changed to a new address. To make this example even stricter, the owner on the main safe can only be changed by a DAO contract and not by an EOA. The reason for this restriction is to remove the option of signatures created by EOAs.
In the next section I will explain briefly why this restriction is needed.
Chain B's safe holds assets worth of 1 billion USD. 

### Message data
We need to get an understanding of the message data which are going to be sent from one chain to another. Since it only makes sense for data which can only come from the chain itself, the space of data can be reduced to actual state of the chain. Any other information would come from off-chain, and then could be actually send directly to the target chain instead of via the source chain.
This is specifically the reason why we restrict the example above that the owner of the main safe can only be changed by a DAO quorum. It's a simplification but we could argue that the DAO quorum is represented as state of chain A. If it were easy to recreate the DAO quorum off-chain then the DAO could simply also send it to chain B, thus there would be no need for an AMB.

#### Pushing vs Pulling data
It is crucial to understand that when it comes to AMBs it actually only makes sense to use them to bridge/prove part of the state of Chain A (Examples: I own X amount of tokens, I own a specific NFT, I am a whitelisted member of Y, the DAO quorum to  change the owner key of the safe).
If we understand this, then we can come to the conclusion that there is no essential need of sending a request on chain A towards chain B (pushing).
We could actually pull the state on chain B directly. An illustration would be, the bridge operators could have other communication channels than on chain requests, but they could be asked off-chain instead to transport the state to chain B. Or in the case of Beamer, the user himself could send the message on chain B, what he claims to be the state of chain A.

### Delegation of admin rights
If we think about our example of synchronizing the owner of safes, what we actually would technically need to do is delegating the admin rights to the bridge operator. And this is the part where we see the need for security unleashes.
While at the main safe the DAO is the admin to change the owner key, on all the other chains the message is received from the incoming side of the bridge (operators). This means the bridge will have the right to change the owner key. This demonstrates the sensitivity and responsibility of the bridge operators. There is no way to really prevent that they are technically able to forge messages and change the owner key to an arbitrary address.
We can clearly see how much of a burden that can be to a bridge protocol and how attack vectors open when there is a high economic stake in the message itself.




## Architectures
We can differentiate between three basic architectures. These are validator, optimistic proof and light client bridges. Other types like POA or centralized intermediares are not of interest here. The different architecture describe the actors and roles in the system and how to make it work to transport state from one chain to another. Specifically here, we will focus only on the arbitrary message example and neglect token messages.

### Validator bridges
In this architecture, a selected validator set is chosen to validate (and relay) messages from one chain to another. Once the validator set validated the messages, the messages are executed on the target chains.
The synchronization of the validators and then executing the messages typically comes with a faster execution than other models.
In order to trust the validator set, typically the sets are either PoA or PoS based. Any misbehavior would then harm their reputation or in the case of PoS result in a slashable event for the stake.

### Optimistic proof bridges
Optimistic proof bridges, first sent the message on the target chain and initiate a fraud proof window before the messages get executed. The messages are validated by the time passed and if none challenges, the window passes and the message becomes finalized, valid and is ready to be processed.
One can see that the execution is likely to be slower than in other architectures. If the challenger opportunity is permissionless, it allows for a secure environment and follows an honest minority assumption where only one honest party is necessary for guaranteeing safety of the bridge. Typically these assumptions are considered to be safe.


### Light clients
Light client bridges use light clients of the source chain on the target chain to retrieve the source chain's state. Typically a light client protocol implements a form of verification of chain A's state on chain B. 
A typical light client protocol verifies that a certain threshold of validators of a PoS chain attested a specific block. Incoming new block headers, from where messages can be derived, will be validated by verifying the consensus of Chain A. For PoS chains this means that it is a type of a validator set, however the validator set is the same as chain A, so no external trust assumptions are added.
However, light client bridges have high operational costs, as verifying the consensus may be quite costly when it gets executed on another chain and needs to be done frequently to prevent long range attacks.
Also the implementation risk is higher compared to other architectures.


## Improvements by using zk proofs
Using zk proofs in respect to computational integrity can improve the security of bridge protocols potentially. Fundamental improves can be achieved especially in light client protocols. The reason for it is that light clients have the biggest operational cost in terms of on chain execution which already let viability of light client bridges suffer.
By outsourcing the computation off chain and only veryfing proofs on chain the design space of light client protocols improve drastically. This would also be the case for validator based bridges.
However, light client protocols could ultimately guarantee for even more security once the zk proof becomes efficient enough with the far goal in mind to proof the whole execution trace.
The argument for light client protocols in terms of security becomes even stronger in this case.

## Considerations for rollups
In general, rollups do not have their own consensus but rather rely on the consensus of the base chain. Therefor, light client protocols as described above are rather not applicable to rollup protocols.
Evenmore so, since block building and base layer are distinct protocols, proving the correct execution trace and state is especially of interest. This is exactly the part where most light client protocols introduce trust assumptions.
An additional thought is that rollups typically have access to a relatively recent block hash, which already serves as a native (trust minimized) data passing between L1 -> L2. This behaviour can be aggregated and thus facilitates l2 -> l2 bridging by storage proofs assuming that rollups have access to the L1 state root natively.
However, this does not solve the problem of proving the correctness of execution. Unless the target rollup waits for finality of the source rollup (which is not desirable) an implicit assumption is made that the L1 submitted state root of another rollup is correct.
