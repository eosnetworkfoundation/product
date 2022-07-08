# Levels of the EOSIO software stack

The EOSIO software stack can be divided across multiple levels (I am preferring to use "levels" here instead of "layers" to avoid confusion with the concept of "layer 1" and "layer 2" technologies that already have other specific meanings in the blockchain space).

The way I prefer to separate the levels of the EOSIO software is the following (in order of lower to higher levels):

1. Host
2. Consensus
3. Frameworks
4. System
5. Application


## Host level

Level 1 (the "Host level") are rules that are captured only in native code. Changing these rules requires changing the executables running on the host machine.

When discussing code existing at this level, the most important thing to first talk about is the actual EOSIO protocol. The section on [changes to the EOSIO protocol](protocol/README.md) captures a set of proposed changes to the protocol.

However, beyond the protocol it is also important to discuss particular implementations (there could be more than one) of the node software which conforms to the EOSIO protocol and what features and properties it has. There can be a lot of additional functionality that application developers rely on which are provided by a node implementation and go well beyond the EOSIO protocol. The section on [changes to the node software implementing the EOSIO protocol](node_implementation/README.md) captures a set of proposed changes to the node software implementing the EOSIO protocol.

## Consensus level

Level 2 (the "Consensus level") captures behavior and code that can have an impact on the ability for the network to reach consensus. However, the code itself is still running in a deterministic sandboxed VM supported by the blockchain protocol (in fact all other code in levels higher than this one do so as well). This code can be upgraded without requiring the operators of the node to download new host binaries. And it may or may not even be upgraded without requiring action to be taken by the operators to opt-in.

While the code at this level should be able to run in the deterministic sandboxed VM and not integrated as part of the compiled host binaries, the proposed items capturing the development work to be done for changes at this level can still be found in the section on [changes to the EOSIO protocol](protocol/README.md). In fact, prior to have the flexibility to put more consensus impacting code into a deterministic sandboxed VM, code at this level is expected to not exist and instead all of what would ideally be at this level would instead default to the Host level. 

## Frameworks level

Level 3 (the "Frameworks level") is the first level in which the code cannot break consensus (which also implies it is running in a deterministic sandboxed VM). If this code is updated or it misbehaves, everyone sees the same changes in the same order as everyone else no matter how much it deviated from what was intended. With some care taken, it should almost always be possible to recover the blockchain from a mistake in code made at this level (i.e. not bricking the blockchain), though in some cases it may require operators of nodes in the network to opt-in to the recovery change.

The line separating this level and the next one (the "System level") is fuzzy. But generally, the idea is to capture powerful tools and frameworks that enable a wide range of possibilities at this level and to not get too opinionated about anything. In particular, the code in the frameworks level should avoid constraining what governance solutions can be built on top to a narrow set that make specific assumptions about human behavior. That kind of opinionated constraining of the software belongs in the System level. The code at this level adopted in any particular blockchain is likely to mostly be the decision of the operators of that blockchain (whatever governance mechanism they use). However, some of the frameworks code is such that each application can choose to adopt it for themselves as needed without needing permission from others.

Examples of the kind of things I would expect to find at the frameworks level include:
* Fungible token standards. (Currently the only unofficial fungible token standard is the implemented by the `eosio.token` contract which, as a contract that full implements concrete fungible tokens as opposed to being a library that implements a more abstract standard, really is more appropriate to consider as belonging to t he System level.)
* Non-fungible token (NFT) standards. (There are a few standards for NFTs that exists in the EOSIO space, however they might be a little too bloated and opinionated to really be appropriate as a basic standard at the Frameworks level.)
* Account and permission system. (The most common account and permission system of EOSIO is actually part of the EOSIO protocol and so it actually belongs in the Host level. Ideally, the account and permission system widely adopted in the EOSIO space would instead be moved higher up to start at the Frameworks level and then further supported at the System level. There are some virtualized account systems in EOSIO blockchains, but none seem to have a widespread and ubiquitous adoption that one would really like to see for something at the Frameworks level.)
* Resource management abstractions. (I believe the Host level should have a very primitive resource management system, even more primitive than what the EOSIO protocol currently support. Then resource models are more convenient for the users of the blockchain, whether a fee-based one, staking-based one, etc., can be implemented on top starting at the Frameworks level.)

The section on [changes to code at the Frameworks and System levels](../frameworks_and_system/README.md) captures a set of proposed changes relevant to code existing at this level of the software stack.

## System level

Level 4 (the "System level") is best understood as the level between the Frameworks level described earlier and the next level which is the "Application level". The boundary between the System level and the Application level comes down to which code is sanctioned by the operators of the blockchain. The Frameworks and System level are where most of the differentiation in the flavor of particular blockchains sharing the same Host level protocol are defined; and of those two, the Frameworks level is mostly supposed to be common code and the System level is where the particular blockchains make opinionated commitments to particular ways of doing things.

The section on [changes to code at the Frameworks and System levels](../frameworks_and_system/README.md) captures a set of proposed changes relevant to code existing at this level of the software stack.

## Application level

Level 5+ (the "Application level") is where third-party (not sanctioned by the operators of the blockchain) code exists. In a public, decentralized blockchain, this level is open-access meaning anonymous parties can freely add and modify their portion of code without asking anyone for permission.

Level 5 can be further refined based on the particular applications. For example, I can see the following breakdown being very common:
5. Protected DeFi platform
6. DeFi application

The "Protected DeFi platform" level in teh above example is one in which the applications that opt into building upon that platform have some restrictions in terms of movement of fungible and non-fungible tokens (delays in withdrawing out of that platform, and a settlement time in which transfer can be reverted for intra-platform transfers between DeFi apps) in exchange for greater security and a way to deal with many types of hacks (based on "intent of code is law" and governance). For additional thoughts on how this may work, see the "Token settlement contract" section of this [forum post](https://forums.eoscommunity.org/t/a-possible-path-for-the-future-of-how-code-on-the-eos-blockchain-is-structured-and-managed/3344).

Those who prefer the philosophy of "code is law" and care a lot about immutability could build their DeFi application directly at level 5 (i.e. on top of the System level directly), assuming they accept the risk of no well-established protection in case of hacks.

If the Protected DeFi platform becomes sanctioned by the operators of the blockchain, then one may think of that code as actually existing at the System level. However, even in that case it would not be mandatory for DeFi applications to integrate into the Protected DeFi platform. Applications that want protections from some hacks and are willing to exchange liquidity for that benefit would directly build on top of the System level but opt into the Protected DeFi platform. Applications that do not want to make that tradeoff due to strong adherence to "code is law" would also build directly on top of the System level but choose not to opt into the Protected DeFi platform.

So the way the levels are organized can be slightly tweaked for an application depending on whether some platforms the application may choose to build upon are adopted by the operators of the blockchain as part of the System managed code or not. In addition, even if a particular platform is adopted by the operators of the blockchain as part of the System level, some third-party may choose to deploy the same platform at a higher level with a different configuration. This could make sense if this alternative deployment is more trusted in terms of its governance execution than the one at the System level. 

In most cases, people tend to trust code more if it is at the System level (with a commitment from the operators of the blockchain to manage it) than if it was at the Application level (managed by some third-party team or DAO-like governance model). But that does not necessarily need to be the case, particularly if the code in question requires a lot of human involvement (e.g. for regular governance function, dispute resolution, or judgement / arbitration) to operate successfully. In those cases, the humans involved in a higher level instance of the same platform code may be more successful and more trusted than the instance at the System level.

For the above reasons, I think it makes sense to restrict System level applications only to those in which it either really only make sense to have one instance of it per blockchain (they naturally benefit from monopolization over that blockchain) or that implement critical functionality that does not involve a lot of human judgement to operate regularly. For applications that do not fit those criteria, it is probably better to leave them as platforms existing at levels higher than the System level and give the market freedom to neutrally decide which ones are best. 

However, it may make sense to provide and perhaps even maintain reference implementations of some critical applications (or libraries), even if they do not meet the lack of human judgement criterion in operation, through similar processes as managing code that will actually deployed at the System level. This can allow users to benefit from the already trusted code development and auditing processes of System level code but still allow competition in the marketplace for the particular instantiations of this code (along with their particular configurations and specific people involved in the governance processes), which may lead to the best of both worlds. So for such code, it may still make sense to include proposals in the section on [changes to code at the Frameworks and System levels](frameworks_and_system/README.md) even if there is no intention of having this code deployed as if it is System level code.
