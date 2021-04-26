https://wiki.polkadot.network/docs/zh-CN/getting-started

## Polkadot 如何运作？

The Polkadot network uses a [sharded model](https://en.wikipedia.org/wiki/Shard_(database_architecture)) where shards - called "[parachains](https://wiki.polkadot.network/docs/zh-CN/learn-parachains)", allow transactions to be processed in parallel instead of sequentially. Each parachain in the network has a unique state transition function (STF). Based on Polkadot's design, as long as a chain's logic can compile to Wasm and adheres to the Relay Chain API, then it can connect to the Polkadot network as a parachain.

Polkadot has a Relay Chain acting as the main chain of the system. Parachains construct and propose blocks to validators on the Relay Chain, where the blocks undergo rigorous [availability and validity](https://wiki.polkadot.network/docs/zh-CN/learn-availability) checks before being added to the finalized chain. As the Relay Chain provides the security guarantees, [collators](https://wiki.polkadot.network/docs/zh-CN/learn-collator) - full nodes of these parachains - don't have any security responsibilities, and thus do not require a robust incentive system. This is how the entire network stays up to date with the many transactions that take place.

![polkadot-relay-chain](https://wiki.polkadot.network/docs/assets/polkadot_relay_chain.png)

为了与其它的区块确认性程序的链条进行交互 (例如: 比特币)，Polkadot 有[桥接链（bridge parachains）](https://wiki.polkadot.network/docs/zh-CN/learn-bridges) 提供双向兼容性。

The [Cross-Chain Messaging Protocol (XCMP)](https://wiki.polkadot.network/docs/zh-CN/learn-crosschain) allows parachains to send messages of any type to each other. The shared security and validation logic of the Relay Chain provide the environment for trust-free message passing that opens up true interoperability.

## 专业术语

https://wiki.polkadot.network/docs/zh-CN/glossary

## 常见问题(FAQ)

**Polkadot Launch**

The Genesis block of the Polkadot network was launched on May 26, 2020, as a Proof of Authority (PoA) network, with governance controlled by the single Sudo (super-user) account. During this time, validators started joining the network and signaling their intention to participate in consensus.

The network evolved to become a Proof of Stake (PoS) network on June 18, 2020. With the chain secured by the decentralized community of validators, the Sudo module was removed on July 20, 2020, transitioning the governance of the chain into the hands of the token (DOT) holders. This is the point where Polkadot became decentralized.

The final step of the transition to full-functioning Polkadot was the enabling of transfer functionality, which occurred on Polkadot at block number 1,205,128 on August 18, 2020, at 16:39 UTC.

On August 21, 2020, Redenomination of DOT occurred. From this date, one DOT (old) equals 100 new DOT.

## Polkadot 架构

Polkadot 采用共享安全性和可互操作的异构多链结构。

**中继链 (Relay Chain)**

中继链是 Polkadot 的中心链。所有的验证者都在中继链上抵押DOT并验证中继链。中继链仅由相对少量的交易类型构成，包括和治理机制互交，平行链拍卖，参与NPo'S。中继链是可以被设计成具有最精简的功能的 - 例如，中继链并不支持智能合约。中继链的主要只能是负责协调整个系统，包括各个平行链。其他的具体工作应该交予实现不同功能的平行链来执行。