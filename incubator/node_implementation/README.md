# Changes to the node software implementing the EOSIO protocol

This section organizes various ideas regarding enhancements and other significant changes to the node software that implements the EOSIO protocol. This does not include changes that are necessary to the node software to conform to the EOSIO protocol (those are addressed [here](../protocol/README.md)) but rather further changes to the node software within the degrees of freedom permitted by the protocol. However, changes to the client-facing APIs exposed by the node software are accepted here. 

In addition, other related software that may not strictly be included as part of the node software, such as history solutions and client libraries or reference implementation, are also covered in this section. However, software implemented as code to be deployed onto the blockchain and executed as part of its supported VMs are not covered in this section. For changes proposed regarding such code, the section capturing [changes to code at the Frameworks and System levels](../frameworks_and_system/README.md) is likely where to look.

The changes are organized into the one of the following categories based on the area for which the protocol change primarily adds value:

* [Development utility](#development-utility-du)
* [Operational efficiency](#operational-efficiency-eo)
* [User convenience](#user-convenience-uc)
* [Complexity reduction](#complexity-reduction-cr)
* [Economics and governance](#economics-and-governance-eg)

## Development utility (DU)

This sub-section includes node implementation changes that primarily serve to enable new features for application developers which allow them to achieve more within their applications with less of an implementation and/or maintenance burden imposed on them.

## Operational efficiency (OE)

This sub-section includes node implementation changes that primarily attempt to reduce the costs of operators running EOSIO nodes regardless of whether these operators are running the nodes for the purpose of block production, providing APIs, enabling a specific application, or simply to contribute to the security and availability of the network. This includes changes to enable more efficient use of computation and memory as well as sharding / scaling solutions which allow for scaling the network to higher levels of throughput with putting excessive hardware demands on any one node.

## User convenience (UC)

This sub-section includes node implementation changes that primarily simplify simplify how users can interact with the blockchain through their client software.

## Complexity reduction (CR)

This sub-section includes deprecations and removal of functionality (possibly with replacement by new functionality) or even high-impact tech debt cleanup in the node implementation that enables the node implementation developers to continue improving the software quickly with low risk of making mistakes.

## Economics and governance (EG)

This sub-section includes node implementation changes that are primarily impacting the economics or governance of how the blockchain network can operate,