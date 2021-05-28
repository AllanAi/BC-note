https://wiki.polkadot.network/docs/zh-CN/getting-started

## 如何运作？

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

## 架构

Polkadot 采用共享安全性和可互操作的异构多链结构。

**中继链 (Relay Chain)**

中继链是 Polkadot 的中心链。所有的验证者都在中继链上抵押DOT并验证中继链。中继链仅由相对少量的交易类型构成，包括和治理机制互交，平行链拍卖，参与NPo'S。中继链是可以被设计成具有最精简的功能的 - 例如，中继链并不支持智能合约。中继链的主要只能是负责协调整个系统，包括各个平行链。其他的具体工作应该交予实现不同功能的平行链来执行。

**角色**

验证人

Validators, if elected to the validator set, produce blocks on the Relay Chain. They also accept proofs of valid state transition from collators. In return, they will receive staking rewards.

收集人

Collators are full nodes on both a parachain and the Relay Chain. They collect parachain transactions and produce state transition proofs for the validators on the Relay Chain. They can also send and receive messages from other parachains using XCMP.

提名人

Nominators bond their stake to particular validators in order to help them get into the active validator set and thus produce blocks for the chain. In return, nominators are generally rewarded with the portion of the staking rewards from that validator.

## 账户

**地址格式**

基于 Substrate 的链的地址格式是SS58。SS58 是一种对于比特币的 Base-58-Check 进行小幅修改的格式。需要注意，此格式包含*地址类型*前缀，可以识别地址具体属于哪一个网络。

例如:

- Polkadot 地址总是以数字1开头。
- Kusama 地址总是以大写字母开头，如C、D、F、G、H、J...
- 通用Substrate 地址以5开头。

**现有存款和回收**

When you generate an account (address), you only generate a *key* that lets you access it. The account does not exist yet on-chain. For that, it needs the existential deposit: 0.001666666667 KSM (on Kusama) or 1 DOT (on Polkadot mainnet).

如果帐户低于现有存款，则会导致该帐户*被回收*。该帐户以及该地址中的所有资金将从区块链状态中删除，以节省空间。您不会失去对回收地址的访问权限 - 只要您拥有私钥或恢复短语，您仍然可以使用该地址 - 但它需要充值额外的存续金额才能与链进行交互。

**多签名账户**

It is possible to create a multi-signature account in Substrate-based chains. A multi-signature account is composed of one or more addresses and a threshold. The threshold defines how many signatories (participating addresses) need to agree on the submission of an extrinsic in order for the call to be successful.

Multi-signature accounts **cannot be modified after being created**. Changing the set of members or altering the threshold is not possible and instead requires the dissolution of the current multi-sig and creation of a new one. As such, multi-sig account addresses are **deterministic**, i.e. you can always calculate the address of a multi-sig just by knowing the members and the threshold, without the account existing yet. This means one can send tokens to an address that does not exist yet, and if the entities designated as the recipients come together in a new multi-sig under a matching threshold, they will immediately have access to these tokens.

使用多重签名帐户进行交易

When you create a new call or approve a call as a multi-sig, you will need to place a small deposit. The deposit stays locked in the pallet until the call is executed. The reason for the deposit is to place an economic cost on the storage space that the multi-sig call takes up on the chain and discourage users from creating dangling multi-sig operations that never get executed. The deposit will be reserved in the caller's accounts so participants in multi-signature wallets should have spare funds available.

The deposit is dependent on the `threshold` parameter and is calculated as follows:

```
存款=存款基数+阈值*存款因子
```

|          | Polkadot (DOT) | Kusama (KSM)   | Polkadot (planck) | Kusama (planck) |
| -------- | -------------- | -------------- | ----------------- | --------------- |
| 存款基数 | 20.088         | 3.347999999942 | 200880000000      | 3347999999942   |
| 存款因子 | .032           | 0.005333333312 | 320000000         | 5333333312      |

Let's consider an example of a multi-sig on Polkadot with a threshold of 2 and 3 signers: Alice, Bob, and Charlie. First, Alice will create the call on chain by calling `as_multi` with the raw call. When doing this Alice will have to deposit `DepositBase + (2 * DepositFactor) = 20.152 DOT` while she waits for either Bob or Charlie to also approve the call. When Bob comes to approve the call and execute the transaction, he will not need to place the deposit and Alice will receive her deposit back.

## DOT

Polkadot

| 单位            | 小数位 | Example      |
| --------------- | ------ | ------------ |
| Planck          | 0      | 0.0000000001 |
| Microdot (uDOT) | 4      | 0.0000010000 |
| Millidot (mDOT) | 7      | 0.0010000000 |
| Dot (DOT)       | 10     | 1.0000000000 |

Kusama

| 单位            | 小数位 | Example        |
| --------------- | ------ | -------------- |
| Planck          | 0      | 0.000000000001 |
| Point           | 3      | 0.000000001000 |
| MicroKSM (uKSM) | 6      | 0.000001000000 |
| MilliKSM (mKSM) | 9      | 0.001000000000 |
| KSM             | 12     | 1.000000000000 |

## 