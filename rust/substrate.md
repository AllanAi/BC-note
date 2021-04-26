https://substrate.dev/docs/zh-CN/

https://substrate.dev/recipes/

**Runtime**: the logic that defines how blocks are processed, including state transition logic. In Substrate, runtime code is compiled to [Wasm](https://substrate.dev/docs/en/knowledgebase/getting-started/glossary#webassembly-wasm) and becomes part of the blockchain's storage state. This enables one of the defining features of a Substrate-based blockchain: [forkless runtime upgrades](https://substrate.dev/docs/en/knowledgebase/runtime/upgrades#forkless-runtime-upgrades).

## Extrinsics

在Substrate的上下文里，extrinsic指的是来自链外的信息，并且会被纳入到区块中。 Extrinsic分为三类：inherents, 签名交易和无签名交易。

需要注意的是，执行函数时触发的 [事件](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/events) 不属于extrinsic。 发出的事件为链本身固有的信息。 举个例子，质押奖励是事件，而不是extrinsic，因为奖励是由链上逻辑触发而生成的。

**区块结构**

Substrate 中的每一个区块都是由一个区块头和一组extrinsic组合而成。 区块头包含区块高度、父区块哈希值、extrinsic根哈希值、链上状态根哈希值，以及摘要等信息。

Extrinsics 被打包到一起进入当前区块中并会被执行，每个extrinsic都在runtime进行了定义。 而extrinsic根哈希值则是由这组待执行交易通过加密算法计算得出的信息摘要。 Extrinsic根哈希值主要有两个用途。 首先，它可以在区块头构建和分发完成之后，防止任何人对该区块头中所含extrinsic内容进行篡改。 其次，它提供了一种方法，在只有区块头信息的条件下，可以帮助轻客户端快速地验证某一区块中存在某笔交易。

 **Inherents**

Inherent指的是那些仅能由区块创建者插入到区块当中的无签名信息。例如，区块创建者可以向区块插入了一条包含时间戳的inherent信息。 时间戳合法性的验证和转账合法性的验证不同，转账可通过签名来验证，而时间戳不行。 相反，验证人根据其他验证人认为该时间戳的合理性来接受或拒绝该块，也就意味着该时间戳在这些验证人自己系统时钟的一些可接受范围内。

 **签名交易**

签名交易里包含了签发该交易的账户私钥签名，这意味着此账户同意承担相应的区块打包费用。 由于签名交易打包上链的费用在交易执行前就可以被识别，所以在网络节点中传播此类交易造成信息泛滥的风险很小。

**不具签名交易**

Substrate 中一个不具签名交易的例子，就是由验证节点定时发送的 [I'm Online](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/frame#im-online)心跳交易。 这种交易虽然包含了一个会话密钥签名，但会话密钥并不能控制资金，因此也无法支付交易费用。 交易池通过检查在某个session内验证人是不是已经提交了一个心跳来防止垃圾信息，如果已经存在会拒绝新的心跳交易。

**Signed Extension**

`SignedExtension` 是一个trait ，通过它可以使用额外的数据或逻辑来扩展交易。

## 交易池

交易池包含所有在网络广播的，已被本地节点接收和验证的，所有具签名和不具未签名的交易。

**排序**

如果交易是有效的，交易队列会将交易分为两组：

- *就绪队列* - 包含所有可放到新的待处理区块中的交易。 对于随 FRAME 构建的 Runtime，所有交易必须严格遵循就绪队列中的顺序。
- *未来队列* - 包含所有可能在未来变成有效的交易。 例如，一个交易可能有一个对其账户来说过高的 `索引 (nonce)` 值。 此交易将在未来队列中等待，直到之前的交易上传至区块链上。

**交易依赖关系**

The `ValidTransaction` [结构体](https://substrate.dev/rustdocs/v3.0.0/sp_runtime/transaction_validity/struct.ValidTransaction.html) 定义了 `requires` 和 `provides` 参数来构建交易的依赖关系。 再加上 `priority` (將在下面讨论)，这个依赖图允许交易池生成有效的交易线性顺序。

**交易优先级**

Transaction `priority` in the `ValidTransaction` [结构体](https://substrate.dev/rustdocs/v3.0.0/sp_runtime/transaction_validity/struct.ValidTransaction.html) 决定了就绪队列中的交易顺序。 当某个节点成为下一个区块生成者时，它将在下一个区块把交易按优先级别从高到低排序，直到达到区块的重量或长度限制。

对于用 FRAME 构建的 Runtime，`priority` 定义为交易要支付的 `fee` (费用)。 例如：

- 如果我们从 *不同的* 发送者那里收到 2 个交易（而且 `nonce=0` 时），我们通过 `priority` 来确定哪个交易更为重要，并优先把它打包进区块中。
- 如果我们从 *同一个* 发送方收到 2 个相同 `nonce` 的交易，那么只会有一个交易会被打包到链上。 我们使用 `priority` 来选择 `fee` 较高的交易，并把它储存到交易池中。

**交易权重**

链可用的资源是有限的。 這裡资源包括内存、存储 I/O、计算、交易/区块大小和状态数据库的大小。 有几种机制可以管理资源被获取，并防止链上的组件过多消耗任何资源。 权重就是用于管理 *验证区块花费的时间* 的机制。 一般来说，这用来限制存储 I/O 和计算。

## 账户摘要

Substrate 使用多组公钥/私钥对来表示网络的参与者。

- **Stash Key**: a Stash account is meant to hold large amounts of funds. Its private key should be as secure as possible in a cold wallet.
- **Controller Key**: a Controller account signals choices on behalf of the Stash account, like payout preferences, but should only hold a minimal amount of funds to pay transaction fees. Its private key should be secure as it can affect validator settings, but will be used somewhat regularly for validator maintenance.
- **Session Keys**: these are "hot" keys kept in the validator client and used for signing certain validator operations. They should never hold funds.

**Stash 密钥**

Stash 密钥是定义 Stash 账户的公私钥对。 这个账户就像一个 “储蓄账户”，你不应该用它进行频繁的交易。 因此，应以最安全的方式来处理其私钥，例如在安全层或用安全硬件来加以保护。

由于 Stash 密钥以离线方式进行保存，它指定了一个 Controller 账户来根据 Stash 账户资金的权重来做出与支付无关的决定。 它还可以指定一个代理账户来代表它在治理中进行投票。

**Controller 密钥**

Controller 密钥是定义 Controller 帐户的公私钥对。 在 Substrate 的 NPOS 模型中，Controller 密钥会代表某个账户进行验证或提名。

Controller 密钥被用于支付偏好设置，例如提名奖励的目的地址，如果是验证人，可用它设置会话密钥。 Controller 账户只需要支付交易费用，所以它只需要最少量的资金。

Controller 密钥不能用来从 Stash 账户中支出资金。 然而，Controller 账户的行为可能会导致惩罚，因此仍应妥善保管。

**会话密钥**

会话密钥是验证者用来签署和共识相关消息的 "热密钥"。 不应该将它们作为控制资金的账户密钥，应只应用于其指定用途。 它们可以定期更改；更改的方式是，你的 Controller 账户只需通过对会话公钥进行签名创建一个证书，并通过外部交易的形式广播该证书就可以了。 会话密钥同样是以泛型进行定义的，且在 runtime 中进行具体化。

为创建会话密钥，验证人的操作者必须证明其拥有一个密钥能够代表其 Stash 帐户 (抵押) 和提名人。 为此，他们使用自己的 Controller 密钥对该秘钥签名，从而创建证书。 然后，他们在链上发布一笔包含该会话证书的交易，来告知区块链该会话密钥代表了他们的 Controller 密钥。

Session keys are used by validators to sign consensus-related messages. [`SessionKeys`](https://substrate.dev/rustdocs/v3.0.0/sp_session/trait.SessionKeys.html) is a generic, indexable type that is made concrete in the runtime.

As a node operator, you can generate keys using the RPC call [`author_rotateKeys`](https://substrate.dev/rustdocs/v3.0.0/sc_rpc/author/trait.AuthorApi.html#tymethod.rotate_keys). You will then need to register the new keys on chain using a [`session.setKeys`](https://substrate.dev/rustdocs/v3.0.0/pallet_session/struct.Module.html#method.set_keys) transaction.

## Runtime

Runtime 是用于定义区块链的业务逻辑。 在基于 Substrate 开发的区块链中，runtime 被称为”[状态转换函数](https://substrate.dev/docs/zh-CN/knowledgebase/getting-started/glossary#state-transition-function-stf)“；Substrate 开发人员在 runtime 中定义了用于表示区块链 [状态](https://substrate.dev/docs/zh-CN/knowledgebase/getting-started/glossary#state)的存储项，同时也定义了允许区块链用户对该状态进行更改的[函数](https://substrate.dev/docs/zh-CN/knowledgebase/learn-substrate/extrinsics)。

为了能够提供无须分叉的升级功能，Substrate采用了可编译成[ WebAssembly (Wasm) ](https://substrate.dev/docs/zh-CN/knowledgebase/getting-started/glossary#webassembly-wasm)字节码的 runtime 形式。 此外，Substrate 还对 runtime 必须实现的[核心基本类型](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/primitives#core-primitives) 进行定义。

核心 Substrate 代码库随附有 [FRAME](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/frame) 框架，FRAME 是Parity 的 Substrate runtime 开发系统，已经应用于 [Kusama](https://github.com/paritytech/polkadot/blob/master/runtime/kusama/src/lib.rs) 和 [Polkadot](https://github.com/paritytech/polkadot/blob/master/runtime/polkadot/src/lib.rs) 等链上。 FRAME 定义了额外的 [runtime 基础类型](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/primitives#frame-primitives)，并提供了一个框架，使得通过编写模块 (称为 "pallets") 来构建 runtime 变得十分容易。 每个 pallet 用于封装特定于该域的逻辑，这些逻辑可表示为一组[存储项](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/storage)、[事件](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/events)、[错误](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/errors)和[可调用函数](https://substrate.dev/docs/zh-CN/knowledgebase/getting-started/glossary#dispatch)的集合。 FRAME 开发人员可选择[创建自己的 pallet ](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/pallets)，也可选择重用包括[50多个 Substrate 随附 pallet ](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/frame#prebuilt-pallets)在内的现有资源。

### FRAME

**Framework for Runtime Aggregation of Modularized Entities (FRAME)** 是一组可简化 runtime 开发的模块（称为pallets）和支持库。 其中 pallets 指的是 FRAME 中那些单一功能模块，承载着特定业务逻辑。

当用 FRAME 来构建 Substrate runtime 时，runtime 是由多个称为 pallet 的组件组合而成的。 每一个 pallet 都包含一组类型变量、存储单元和函数，它们共同定义了 runtime 所包含的特性和功能。

**System Library**

The [System library](https://substrate.dev/rustdocs/v3.0.0/frame_system/index.html) provides low-level types, storage, and functions for your blockchain. 其它所有的 pallets 都依赖于System 库，所以 System 库也是 Substrate runtime 的基础。

System 库定义了 Substrate runtime 的所有核心类型，例如：

- 来源 (Origin)
- 区块数 (Block Number)
- 用户帐号 (Account ID)
- 哈希值 (hash)
- 区块头 (Header)
- 版本 (Version)
- 其它...

它还定义了大量的系统关键存储项，例如：

- Account Nonce
- Block Hash
- 区块数 (Block Number)
- Events
- 其它...

除此之外，System 模块还定义了许多底层函数，这些函数可以用来访问区块链存储单元、验证交易发送方等等。

### Pallets

一个完整的Substrate pallet由5部分组成：

```rust
// 1. 导入库和依赖项
//此pallet支持使用任何带有`no_std`标志编译的Rust库。
use support::{decl_module, decl_event, decl_storage, ...}

// 2. Runtime配置Trait 
//所有runtime类型和常量都放在这里。 
//如果此pallet依赖于其他特定的pallet，则应将依赖pallet的配置trait添加到继承的trait列表中。
pub trait Config: system::Config { ... }

// 3. Runtime事件
// 事件是一种用于报告特定条件和情况发生的简单手段，用户、Dapp和区块链浏览器都可能对事件的感兴趣。没有它就很难发现。
decl_event!{ ... }

 //4. Runtime存储
//Runtime存储允许在保证“类型安全“前提下使用Substrate存储数据库，因而可在块与块之间留存内容。
decl_storage!{ ... }
 //5. Pallet声明 
//此模块定义了最终从此pallet导出的"Module"结构体。
//它定义了该pallet公开的可调用函数，并在区块执行时协调该pallet行为。
decl_module! { ... }
```

### 存储

FRAME's [`Storage pallet`](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/index.html) gives runtime developers access to Substrate's flexible storage APIs, which can support any value that is encodable by [Parity's SCALE codec](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/codec). These include:

- [Storage Value](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.StorageValue.html) - used to store any single value, such as a `u64`.
- [Storage Map](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.StorageMap.html) - used to store a key-value hash map, such as a balance-to-account mapping.
  - `translate()` - use the provided function to translate all elements of the map, in no particular order. To remove an element from the map, return `None` from the translation function. See the docs: [`IterableStorageMap`](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.IterableStorageMap.html#tymethod.translate) and [`IterableStorageDoubleMap`](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.translate).
- [Storage Double Map](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.StorageDoubleMap.html) - used as an implementation of a storage map with two keys to provide the ability to efficiently removing all entries that have a common first key.

```rust
decl_storage! {
    trait Store for Module<T: Config> as Example {
        // A StorageValue item.
        SomePrivateValue: u32;
        pub SomePrimitiveValue get(fn some_primitive_value): u32;
        // A StorageMap item.
        pub SomeMap get(fn some_map): map hasher(blake2_128_concat) T::AccountId => u32;
        // A StorageDoubleMap item.
        pub SomeDoubleMap: double_map hasher(blake2_128_concat) u32, hasher(blake2_128_concat) T::AccountId => u32;
    }
}

Copy
```

Notice that the map's storage items specify [the hashing algorithm](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/storage#hashing-algorithms) that will be used.

In the example above, all the storage items except `SomePrivateValue` are made public by way of the `pub` keyword. Blockchain storage is always publicly [visible from *outside* of the runtime](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/storage#accessing-storage-items). However, runtime engineers can make storage items private, to control the access of storage items from pallets *within* the runtime.

Here is an example that implements a getter method named `some_value` for a Storage Value named `SomeValue`. This pallet would now have access to a `Self::some_value()` method in addition to the `SomeValue::get()` method:

```rust
decl_storage! {
    trait Store for Module<T: Config> as Example {
        pub SomeValue get(fn some_value): u64;
    }
}
```

Here is an example of specifying the default value for all items in a map:

```rust
decl_storage! {
    trait Store for Module<T: Config> as Example {
        pub SomeMap: map u64 => u64 = 1337;
    }
}
```

**Verify First, Write Last**

Substrate does not cache state prior to extrinsic dispatch. Instead, it applies changes directly as they are invoked. If an extrinsic fails, any state changes will persist. Because of this, it is important not to make any storage mutations until it is certain that all preconditions have been met. In general, code blocks that may result in mutating storage should be structured as follows:

```rust
{
  // all checks and throwing code go here

  // ** no throwing code below this line **

  // all event emissions & storage writes go here
}
```

### 执行流程

Substrate runtime的执行由Executive模块来协调。

与FRAME中的其他模块不同，Executive不是*runtime*模块， 而是一个普通的Rust模块，它负责调用区块链中包含的各种runtime模块。

Executive模块对外暴露了 `execute_block` 函数，以实现如下功能：

- [初始化区块](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/execution#initializing-a-block)
- [执行extrinsics](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/execution#executing-extrinsics)
- [完结区块](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/execution#finalizing-a-block)

**验证交易**

在区块开始执行前，将检查签名交易的有效性。 这个行为不会产生任何副作用；它只是确保交易无论能否打包到区块，也不会引起程序崩溃。 因此， 对存储所作的更改将被丢弃。

**执行区块**

只要有效交易的队列不为空，Executive模块就开始执行区块。

**初始化区块**

区块初始化时，System模块和其他runtime模块都会首先调用其`on_initialize` 函数，把由模块定义的、需要前置的业务逻辑在交易执行前全部处理掉。 除System模块总是优先处理外，其余模块均按照在`construct_runtime!`宏里定义的顺序来执行。

接下来是初始检查，该步骤将验证区块头中的父哈希是否正确，以及extrinsics trie的根是否囊括了所有的extrinsics。

**执行Extrinsics**

区块初始化完成以后，将按照交易优先级顺序执行每一个有效的extrinsic。 Extrinsics一定不能在rutnime逻辑中引起程序崩溃，否则系统将很容易受到用户攻击，而通过这种攻击，用户可不受任何惩罚地消耗计算资源。

当extrinsic执行时，原有存储状态不会提前被缓存下来，修改将直接应用到存储上。 因此，在更改存储状态之前，runtime开发人员应进行所有必要检查，以确保extrinsic能执行成功。 一旦extrinsic在执行过程中失败了，存储更改将不能回滚。

extrinsic执行时触发的[事件](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/events)也会写入存储。 因此，在完成所有待执行动作之前，不应该触发相关事件。 否则，倘若extrinsic在事件触发后才执行失败的话，该事件将不能回滚。

**完结区块**

After all queued extrinsics have been executed, the Executive module calls into each module's `on_idle` and `on_finalize` function to perform any final business logic which should take place at the end of the block. 所有模块也将再次按照在 `construct_runtime!` 宏中定义的顺序来执行，但 System 模块须等到最后完成。

接下来是终极检查，将验证区块头中的摘要和存储根是否与计算结果相匹配。

`on_idle` will also pass through the remaining weight of the block to allow for execution based on the usage of the blockchain.

### 事件

Runtime事件是由 `decl_event!` 宏创建的。

```rust
decl_event!(
    pub enum Event<T> where AccountId = <T as Config>::AccountId {
        /// Set a value.
        ValueSet(u32, AccountId),
    }
);
```

`Event`枚举类型需要在runtime的配置trait中进行声明。

```rust
pub trait Config: system::Config {
    type Event: From<Event<Self>> + Into<<Self as system::Config>::Event>;
}
```

**向runtime暴露事件**

模块中的事件需要暴露给Substrate的runtime(`/runtime/src/lib.rs`)）。

首先，要在模块的配置trait中实现事件类型：

```rust
// runtime/src/lib.rs
impl template::Config for Runtime {
    type Event = Event;
}
```

然后再将该 `Event` 类型添加到 `construct_runtime!` 宏里:

```rust
// runtime/src/lib.rs
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        // --snip--
        TemplateModule: template::{Module, Call, Storage, Event<T>},
        //--add-this------------------------------------->^^^^^^^^
    }
);
```

**保存事件**

Substrate提供了一个默认实现，来存储定义在`decl_module!` 宏内的事件。

```rust
decl_module! {
    pub struct Module<T: Config> for enum Call where origin: T::Origin {
        // Default implementation of `deposit_event`
        fn deposit_event() = default;

        fn set_value(origin, value: u64) {
            let sender = ensure_signed(origin)?;
            // --snip--
            Self::deposit_event(RawEvent::ValueSet(value, sender));
        }
    }
}
```

The default behavior of this function is to call [`deposit_event`](https://substrate.dev/rustdocs/v3.0.0/frame_system/pallet/struct.Pallet.html#method.deposit_event) from the FRAME system, which writes the event to storage.

函数将会把事件存到区块的System模块的runtime存储中。 在每一个新区块的开头，System模块都会自动删除上一个区块中存储的所有事件。

下游的支持库如[Polkadot-JS api](https://substrate.dev/docs/zh-CN/knowledgebase/integrate/polkadot-js)，可直接支持通过默认实现保存的事件，但如果想以不同方式处理事件，也可实现自定义的`deposit_event`函数。

**监听事件**

Substrate RPC不会直接暴露用于查询事件的端口。 如果采用了事件保存的默认实现，则可通过查询System模块的存储单元来查看当前区块的事件列表。 否则，则可考虑通过[Polkadot-JS api](https://substrate.dev/docs/zh-CN/knowledgebase/integrate/polkadot-js) 的WebSocket来订阅runtime事件。

### 错误

Runtime代码应该显式且优雅地处理所有错误情况，也就是说runtime代码 **必须是**“不抛出”的，用Rust术语来讲，就是决不能"[panic](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html)" 。 A common idiom for writing non-throwing Rust code is to write functions that return [`Result` types](https://substrate.dev/rustdocs/v3.0.0/frame_support/dispatch/result/enum.Result.html). `Result`枚举类型具有一个名为 `Err`的变量，该变量可让函数传达执行失败的信息，以避免程序“panic”。 Dispatchable calls in the FRAME system for runtime development *must* return a [`DispatchResult` type](https://substrate.dev/rustdocs/v3.0.0/frame_support/dispatch/type.DispatchResult.html) that *should* be a [`DispatchError`](https://substrate.dev/rustdocs/v3.0.0/frame_support/dispatch/enum.DispatchError.html) if the dispatchable function encountered an error.

每个FRAME pallet都可以使用[`decl_error!` 宏](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/macros#decl_error)来自定义 **DispatchError**的内容。

```rust
// Errors inform users that something went wrong.
decl_error! {
    pub enum Error for Module<T: Config> {
        /// Error names should be descriptive.
        InvalidParameter,
        /// Errors should have helpful documentation associated with them.
        OutOfSpace,
    }
}
```

为了能在pallet中触发自定义错误，pallet的module部分必须要配置`Error` 类型。

```rust
decl_module! {
    pub struct Module<T: Config> for enum Call where origin: T::Origin {
        // Errors must be initialized if they are used by the pallet.
        type Error = Error<T>;

        /* --snip-- */
  }
}
```

The FRAME Support module also includes a helpful [`ensure!` macro](https://substrate.dev/rustdocs/v3.0.0/frame_support/macro.ensure.html) that can be used to check pre-conditions and emit an errors if they are not met.

```rust
frame_support::ensure!(param < T::MaxVal::get(), Error::<T>::InvalidParameter);
```

### 交易费用

当出块人构造一个块时，必须要限制该区块的执行时间。 区块主体由一系列[extrinsics](https://substrate.dev/docs/zh-CN/knowledgebase/learn-substrate/extrinsics)组成。 由于执行不同extrinsic所需的资源可能有所不同，Substrate提供了一种称为“权重”的灵活机制，来代表执行某extrinsic所需的*时间*。 为了经济上的可持续发展并限制信息泛滥，某些交易（主要是用户发送的交易）在执行之前需要付费。

**打包费用**

交易费用由两部分组成：

- `length_fee`: 每字节的费用乘以编码后的extrinsic长度（以字节为单位）。 See `TransactionByteFee`。
- `weight_fee`: 基于extrinsic权重的费用，是一个接收两个入参的函数。 第一个参数为 `ExtrinsicBaseWeight`，它是在runtime中声明的，且应用于所有的extrinsics。 基本权重涵盖了如签名验证之类的不可变开销。 第二个参数为可供灵活使用的`#[weight]`注释，它用于标记extrinsic的复杂性。 In order to convert the weight to `Currency`, the runtime must define a [`WeightToFee`](https://substrate.dev/rustdocs/v3.0.0/pallet_transaction_payment/trait.Config.html#associatedtype.WeightToFee) struct that implements a conversion function: [`Convert`](https://substrate.dev/rustdocs/v2.0.0/sp_runtime/traits/trait.Convert.html)。

基于上述情况，一个可调用函数的最终费用为：

```
fee =
  len(tx) * length_fee +
  WeightToFee(weight)
```

这个 `费用`称之为“打包费”。 请注意，打包费是在实际调用 extrinsic *之前* 向发送方收取的，这样即使 extrinsic 执行失败，费用仍会被收取。 如果帐户中的余额不足以支付费用并保持激活 (即帐户最低余额要求及打包费总和)，则不会扣除任何费用而交易也不会被执行。 后一种情况应该很少见，因为交易队列和区块构造逻辑会在向块中添加 extrinsic 之前就执行相关检查。

**费用加乘因素**

最终费用可以包含一定程度的可变性。 为了满足这个要求，Substrate 提供了：

- [`NextFeeMultiplier`](https://substrate.dev/rustdocs/v3.0.0/pallet_transaction_payment/struct.Module.html#method.next_fee_multiplier)： 存储在 Transaction Payment 模块中的可配置乘数。
- [`FeeMultiplierUpdate`](https://substrate.dev/rustdocs/v3.0.0/pallet_transaction_payment/trait.Config.html#associatedtype.FeeMultiplierUpdate)： Runtime 的可配置参数，用于描述该乘数的变化方式。

`NextFeeMultiplier` 的类型为 `Fixed64`，它可以表示一个固定的浮点数。 因此根据上述的打包费公式，最终版本将为：

```
fee =
  len(tx) * length_fee +
  WeightToFee(weight)

final_fee = fee * NextFeeMultiplier
```

**附加费用**

打包费必须在extrinsic执行之前是可计算的，因此只能代表固定的逻辑。 有些交易则需要使用其他策略来限制资源的使用。 例如：

- 保证金：保证金是用于在某些链上事件触发后，要么退还要么罚没的一种费用。 例如，runtime开发人员可能会希望实现一种用于投票的保证金机制。在这个情景下，保证金会在投票结束之后正常退还。但如果投票人尝试作恶，则保证金会在投票结束后被罚没。
- 押金：押金是可被返还的费用。 例如，用户可能会被要求支付押金以执行占用存储的操作； 如果后续操作释放了该存储空间，则用户的押金将被返还。
- 销毁：交易可能会基于其逻辑在执行中销毁资金。 例如，如果交易的结果是创建了新的存储条目，从而增加了链上状态的占用空间，则交易可能会因此销毁请求方发送的资金。
- 限制：runtime开发人员可以自由地对某些操作强制执行恒定的或可配置的限制。 例如，默认的Staking pallet仅允许1个提名人提名16个验证人，以限制验证人选举过程的复杂性。

重点要注意，如果在链中查询交易费用，它只会返回打包费。

**默认权重注释**

Substrate中的所有可调用函数必须要指定权重。 可通过使用标注系统，将数据库读/写权重的固定值和基于基准测试的固定值组合在一起。 最基本的示例如下所示：

```rust
#[weight = 100_000]
fn my_dispatchable() {
    // ...
}
```

请注意，`ExtrinsicBaseWeight` 会被自动加到权重中，来纳入一个空的 extrinsic 包含到区块内的成本。

**把数据库访问参数化**

为了使权重标注独立于已部署的数据库后端，我们把它们定义为常量，在标注中用于表示可调用函数的数据库使用情况：

```rust
#[weight = T::DbWeight::get().reads_writes(1, 2) + 20_000]
fn my_dispatchable() {
    // ...
}
```

可调用函数增加了20,000权重，包含一个数据库读取和两个数据库写入操作，以及其他一些操作。 通常我们所说的一次数据库访问，就是指对在 `decl_storage!` 代码块中声明的值的一次访问。 然而，由于某个值被访问了一次后就会被缓存，而再次访问则不会导致重复的数据库操作。因此，只有第一次访问会被计入权重。 这意味着：

- 多次读取相同的值只计算一次读取权重。
- 多次写入相同的值只计算一次写入权重。
- 多次读取相同值并随后对该值进行写入，只计算一次读取和一次写入的权重。
- 一次写入并随后跟着一次读取，只计算一次写入的权重。

**可调用函数的类**

可调用函数可分成三个类：`Normal`, `Operational`, 和`Mandatory`。 如果未在权重标注中额外定义，则可调用函数默认为`Normal`。 开发人员可以指定可调用函数使用另一个类，例如：

```rust
#[weight = (100_000, DispatchClass::Operational)]
fn my_dispatchable() {
    // ...
}
```

此元组形式的标注还可以指定最后一个参数，该参数会确定是否向用户收取基于所标注的权重的费用。 除非另有定义，否则默认为`Pays::Yes` ：

```rust
#[weight = (100_000, DispatchClass::Normal, Pays::No)]
fn my_dispatchable() {
    // ...
}
```

**Normal 类的可调用函数**

此类的可调用函数表示常规的用户触发交易。 These types of dispatches may only consume a portion of a block's total weight limit; this portion can be found by examining the [`AvailableBlockRatio`](https://substrate.dev/rustdocs/v3.0.0/frame_system/limits/struct.BlockLength.html#method.max_with_normal_ratio). Normal dispatches are sent to the [transaction pool](https://substrate.dev/docs/zh-CN/knowledgebase/learn-substrate/tx-pool).

**Operational 类的可调用函数**

与 normal 类的可调用函数 *使用了* 网络功能相反，operational 类的可调用函数 *提供了* 联网功能。 These types of dispatches may consume the entire weight limit of a block, which is to say that they are not bound by the [`AvailableBlockRatio`](https://substrate.dev/rustdocs/v3.0.0/frame_system/limits/struct.BlockLength.html#method.max_with_normal_ratio). Dispatches in this class are given maximum priority and are exempt from paying the `length_fee`.

**Mandatory类的可调用函数**

即使有可能会导致区块超出其权重上限，Mandatory 类的可调用函数也必然会被包含到区块里。 这种可调用函数类只能应用于[inherents](https://substrate.dev/docs/zh-CN/knowledgebase/learn-substrate/extrinsics#Inherents)，且只能用在作为区块验证过程一部分的函数上。 由于这种类型的可调用函数始终会被包含到区块中，而与函数的权重无关，因此在函数的验证过程中，如何防止恶意验证程序滥用函数，以及保证构建出有效且权重不会太高的区块，显得至关重要。 通常可以通过确保操作始终非常轻，并且只能在一个块中被包含一次，来实现此目的。 为了使恶意的验证者更难以滥用这种类型的可调用函数，在返回错误的情况下，该可调用函数将不能被包含到区块当中去。 这种类型的可调用函数是为了这种假设而存在：允许创建一个超权重区块比禁止创建任何区块更好。

**动态权重**

除固定权重和常量之外，在计算权重时还可把可调用函数的输入参数这个因素考虑进来。 这种权重应该是可以通过对输入参数进行简单运算而轻松得出的：

```rust
#[weight = FunctionOf(
  |args: (&Vec<User>,)| args.0.len().saturating_mul(10_000),
  DispatchClass::Normal,
  Pays::Yes,
)]
fn handle_users(origin, calls: Vec<User>) {
    // Do something per user
}
```

**调用后权重校正**

取决于最终执行逻辑，可调用函数实际消耗的权重有可能比调用前预估的要少。 权重校正的重要性在[权重篇章](https://substrate.dev/docs/zh-CN/knowledgebase/learn-substrate/weight#post-dispatch-weight-correction)里已经解释得非常清楚。 为了校正权重，需在可调用函数中特殊声明一个返回类型，以返回其实际执行权重：

```rust
#[weight = 10_000 + 500_000_000]
fn expensive_or_cheap(input: u64) -> DispatchResultWithPostInfo {
    let was_heavy = do_calculation(input);

    if (was_heavy) {
        // None means "no correction" from the weight annotation.
        Ok(None.into())
    } else {
        // Return the actual weight consumed.
        Ok(Some(10_000).into())
    }
}
```

### 调试

在所有软件开发中，调试都是必不可少的，区块链也不例外。 大部分用于通用目的的Rust调试工具，也同样适用于Substrate。 但是，由于Substrate runtime是在`no_std`环境中运行的，进行调试时会有一些限制。

**从Native Runtime打印日志**

For performance-preserving, native-only debugging, use the macros in the [`frame_support::debug::native` module](https://substrate.dev/rustdocs/v3.0.0/frame_support/debug/native/index.html).

```rust
pub fn do_something(origin) -> DispatchResult {
    print("Execute do_something");

    let who = ensure_signed(origin)?;
    let my_val: u32 = 777;

    Something::put(my_val);

    frame_support::debug::native::debug!("called by {:?}", who);

    Self::deposit_event(RawEvent::SomethingStored(my_val, who));
    Ok(())
}
```

The [`frame_support::debug::native::debug!`](https://substrate.dev/rustdocs/v3.0.0/frame_support/debug/native/macro.debug.html) macro avoids extra overhead in Wasm runtimes, but is only effective when the runtime is executing in native mode. 为了查看这些日志消息，必须给节点配置正确的日志目标。 默认情况下，日志目标的名称与包含了日志消息的crate的名称保持一致。

```sh
./target/release/node-template --dev -lpallet_template=debug
```

`debug!`宏接收了一个可选参数`target`，于指定日志目标名称。

```rust
frame_support::debug::native::debug!(target: "customTarget", "called by {:?}", who);
```

**从Wasm Runtime打印日志**

可以通过牺牲性能以换取Wasm runtime输出日志记录。

```rust
pub fn do_something(origin) -> DispatchResult {
    print("Execute do_something");

    let who = ensure_signed(origin)?;
    let my_val: u32 = 777;

    Something::put(my_val);

    frame_support::debug::RuntimeLogger::init();
    frame_support::debug::debug!("called by {:?}", who);

    Self::deposit_event(RawEvent::SomethingStored(my_val, who));
    Ok(())
}
```

**Printable Trait**

Printable trait提供了runtime在`no_std` 和`std`两种环境中进行打印的一种方式。

Substrate默认对某些类型 (`u8`, `u32`, `u64`, `usize`, `&[u8]`, `&str`) 实现了此特征。 当然也可把它应用到自定义类型上。 以下示例用node-template作为代码样版，在一个pallet的`Error`类型上实现了printable trait。

```rust
use sp_runtime::traits::Printable;
use sp_runtime::print;

// The pallet's errors
decl_error! {
    pub enum Error for Module<T: Config> {
        /// Value was None
        NoneValue,
        /// Value reached maximum and cannot be incremented further
        StorageOverflow,
    }
}

impl<T: Config> Printable for Error<T> {
    fn print(&self) {
        match self {
            Error::NoneValue => "Invalid Value".print(),
            Error::StorageOverflow => "Value Exceeded and Overflowed".print(),
            _ => "Invalid Error Case".print(),
        }
    }
}

/// takes no parameters, attempts to increment storage value, and possibly throws an error
pub fn cause_error(origin) -> dispatch::DispatchResult {
    // Check it was signed and get the signer. See also: ensure_root and ensure_none
    let _who = ensure_signed(origin)?;

    print("My Test Message");

    match Something::get() {
        None => {
            print(Error::<T>::NoneValue);
            Err(Error::<T>::NoneValue)?
        }
        Some(old) => {
            let new = old.checked_add(1).ok_or(
                {
                    print(Error::<T>::StorageOverflow);
                    Error::<T>::StorageOverflow
                })?;
            Something::put(new);
            Ok(())
        },
    }
}
```

使用RUST_LOG环境变量来运行节点二进制文件以打印值。

```sh
RUST_LOG=runtime=debug ./target/release/node-template --dev
```

每次调用runtime函数时，值都会打印在终端或标准输出中。

```rust
2020-01-01 tokio-blocking-driver DEBUG runtime  My Test Message  <-- str implements Printable by default
2020-01-01 tokio-blocking-driver DEBUG runtime  Invalid Value    <-- the custom string from NoneValue
2020-01-01 tokio-blocking-driver DEBUG runtime  DispatchError
2020-01-01 tokio-blocking-driver DEBUG runtime  8
2020-01-01 tokio-blocking-driver DEBUG runtime  0                <-- index value from the Error enum definition
2020-01-01 tokio-blocking-driver DEBUG runtime  NoneValue        <-- str which holds the name of the ident of the error
```

**Substrate自带的`print`函数**

```rust
use sp_runtime::print;

// --snip--
pub fn do_something(origin) -> DispatchResult {
    print("Execute do_something");

    let who = ensure_signed(origin)?;
    let my_val: u32 = 777;

    Something::put(my_val);

    print("After storing my_val");

    Self::deposit_event(RawEvent::SomethingStored(my_val, who));
    Ok(())
}
// --snip--
```

可使用`RUST_LOG` 环境变量启动区块链，以查看打印日志。

```sh
RUST_LOG=runtime=debug ./target/release/node-template --dev
```

如果触发了错误，这些值将打印在终端或标准输出中。

```sh
2020-01-01 00:00:00 tokio-blocking-driver DEBUG runtime  Execute do_something
2020-01-01 00:00:00 tokio-blocking-driver DEBUG runtime  After storing my_val
```

**If Std**

传统的`print` 函数允许您打印并实现`Printable` trait。 但是，在某些传统用例下，您可能想要做的不仅仅是打印，或者不想仅仅出于调试目的而去麻烦地使用Substrate的一些traits。 The [`if_std!` macro](https://substrate.dev/rustdocs/v3.0.0/sp_std/macro.if_std.html) is useful for this situation.

使用此宏时要注意，仅当真正运行的是runtime的native版本时，宏内部代码才会执行。

```rust
use sp_std::if_std; // Import into scope the if_std! macro.
```

`println!` 语句应放置在`if_std`宏内。

```rust
decl_module! {

        // --snip--
        pub fn do_something(origin) -> DispatchResult {

            let who = ensure_signed(origin)?;
            let my_val: u32 = 777;

            Something::put(my_val);

            if_std! {
                // This code is only being compiled and executed when the `std` feature is enabled.
                println!("Hello native world!");
                println!("My value is: {:#?}", my_val);
                println!("The caller account is: {:#?}", who);
            }

            Self::deposit_event(RawEvent::SomethingStored(my_val, who));
            Ok(())
        }
        // --snip--
}
```

每次调用runtime函数时，值都会打印在终端或标准输出中。

```sh
$       2020-01-01 00:00:00 Substrate Node
        2020-01-01 00:00:00   version x.y.z-x86_64-linux-gnu
        2020-01-01 00:00:00   by Anonymous, 2017, 2020
        2020-01-01 00:00:00 Chain specification: Development
        2020-01-01 00:00:00 Node name: my-node-007
        2020-01-01 00:00:00 Roles: AUTHORITY
        2020-01-01 00:00:00 Imported 999 (0x3d7a…ab6e)
        # --snip--
->      Hello native world!
->      My value is: 777
->      The caller account is: d43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d (5GrwvaEF...)
        # --snip--
        2020-01-01 00:00:00 Imported 1000 (0x3d7a…ab6e)
```

### 测试

### Benchmarking

### 链上随机生成

### 升级

无分叉来升级 runtime 是 Substrate 框架用于区块链开发的特别亮点之一。 这种能力是通过将状态转换函数的定义（即 runtime 自身）作为区块链持续变化的 runtime 状态来实现的。 这使得网络维护人员能够利用区块链无需信任、去中心化共识的特性，对 runtime 进行安全升级。

In the FRAME system for runtime development, the System library defines [the `set_code` call](https://substrate.dev/rustdocs/v3.0.0/frame_system/pallet/enum.Call.html#variant.set_code) that is used to update the definition of the runtime. The [Forkless Upgrade a Chain tutorial](https://substrate.dev/docs/zh-CN/tutorials/forkless-upgrade/scheduled-upgrade) describes the details of FRAME runtime upgrades and demonstrates two mechanisms for performing them. 该教程演示的两种升级方法严格意义上都是 *累加型* 的，这意味着它们通过 *扩展*，而不是 *更新* 现有的 runtime 状态来修改 runtime 逻辑。 如果 runtime 升级时对现时的状态有所更改，则可能有必要执行 “存储迁移”。

**Runtime 版本控制**

为了使 [执行器](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/executor) 能够选择合适的 runtime 执行环境，它需要知道原生 runtime 及 Wasm runtime 中的 `spec_name`、`spec_version` 和 `authoring_version` 信息。

The runtime provides a [runtime version struct](https://substrate.dev/rustdocs/v3.0.0/sp_version/struct.RuntimeVersion.html). 下面是 runtime version 结构体的简单示例：

```rust
pub const VERSION: RuntimeVersion = RuntimeVersion {
  spec_name: create_runtime_str!("node-template"),
  impl_name: create_runtime_str!("node-template"),
  authoring_version: 1,
  spec_version: 1,
  impl_version: 1,
  apis: RUNTIME_API_VERSIONS,
  transaction_version: 1,
};
```

- `spec_name`：用于区分不同 Substrate runtime 的标识符。
- `impl_name`：规范的实现名称。 这对节点的影响不大，仅用于区分不同团队所实现的代码。
- `authoring_version`：出块接口版本号。 除非该值等于原生 runtime 的版本号，否则出块节点将不会尝试生成区块。
- `spec_version`：runtime 规范版本号。 全节点不会尝试使用原生 runtime 来替代链上的 Wasm runtime，除非原生和 Wasm runtime 中的 `spec_name`、`spec_version`、和 `authoring_version` 都相等。
- `impl_version`：规范实现版本号。 节点可以完全忽略此值；它仅用来说明代码是不同的；只要其他两个版本相同，那么即使实现代码有所差异，它们所作的操作本质都是一样的。 不影响共识的优化是唯一能够导致 `impl_version` 改变的变化。
- `transaction_version`：外部交易接口版本号。 该值必须在以下情况下更新：已更新外部交易参数(number、order 或 types)、外部交易或模块被移除、`construct_runtime!` 宏中的模块顺序 *或* 外部交易模块的顺序发生变化。 如果该值发生变化，那么 `spec_version` 也必须更新。
- `apis` is a list of supported [runtime APIs](https://substrate.dev/rustdocs/v3.0.0/sp_api/macro.impl_runtime_apis.html) along with their versions.

如上所述，执行器在选择执行原生 runtime 之前都会验证是否具有相同的共识算法逻辑，姑勿论它版本高低如何。

> **注：** runtime 的版本号是手动设置的。 因此如果为 runtime 指定错误的版本号，那么执行器仍可能作出不恰当的决定。

**存储迁移**

存储迁移是一系列自定义的一次性函数，它允许开发人员能够重新调整现有存储以满足新的需求。 例如，假设 runtime 升级的目的是将代表用户余额的数据类型从 *无符号 (unsigned)* 整数转换为 *有符号 (signed)* 整数 — 在这种情况下，存储迁移将以无符号的形式读取现有值，然后转换为有符号整数的值回写到存储中。 如果没有及时进行存储迁移，将导致 runtime 执行引擎错误地解释代表 runtime 状态的存储值，这可能会发生不可预料的结果。 Substrate runtime 存储迁移属于被广泛称为 "[数据迁移](https://en.wikipedia.org/wiki/Data_migration)" 的一种存储管理类别。

FRAME storage migrations are implemented by way of [the `OnRuntimeUpgrade` trait](https://substrate.dev/rustdocs/v3.0.0/frame_support/traits/trait.OnRuntimeUpgrade.html), which specifies a single function, `on_runtime_upgrade`. 该函数提供了一个回调 (hook)，它允许 runtime 开发人员指定在 runtime 升级 *之后* 立即执行的逻辑，这个逻辑会在任何 [外部调用或甚至于 `on_initialize` 函数](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/execution#executing-a-block) 执行 *之前* 运行。

**迁移测试**

测试存储迁移非常重要，因此 Substrate 提供了许多实用工具来协助完成此过程。 [Substrate Debug Kit](https://github.com/paritytech/substrate-debug-kit) 包含一个 [Remote Externalities](https://github.com/paritytech/substrate-debug-kit/tree/master/remote-externalities) 工具，该工具允许在真实的链数据上运行存储迁移单元测试。 使用 [Fork Off Substrate](https://github.com/maxsam4/fork-off-substrate) 脚本可以轻松创建用于启动本地测试链，测试 runtime 升级和存储迁移的链规范文档。

## 链规范

https://substrate.dev/docs/zh-CN/knowledgebase/integrate/chain-spec

## Subkey 工具

https://substrate.dev/docs/zh-CN/knowledgebase/integrate/subkey

## 内存分析

https://substrate.dev/docs/zh-CN/knowledgebase/integrate/memory-profiling

## SCALE 编解码器

SCALE（Simple Concatenated Aggregate Little-Endian）编解码器是一个轻量级、高效的二进制序列化和反序列化编解码器。

它是专为资源受限的执行环境所设计的，比如 [Substrate runtime](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/)，可对数据进行高性能、无需拷贝的编码和解码。 它无法进行自我描述，并假定解码环境拥有编码数据的所有类型信息。

Substrate 使用 [`parity-scale-codec`](https://github.com/paritytech/parity-scale-codec)，它是 SCALE 编解码器的 Rust 实现。 采用这个库和 SCALE 编解码器会使 Substrate 和区块链系统会更有优势，因为：

- 相对于 [serde](https://serde.rs/) 这样的通用序列化框架来说，它是轻量级的，因为 serde 添加了大量的模板，会使二进制文件过大。

- 它不使用 Rust STD，因此可以把文件编译为 Wasm格式，用在 Substrate runtime 之中。

- 它对在Rust中为新数据类型派生编解码逻辑的支持极为友好：

  ```
  #[derive(Encode, Decode)]
  ```

## 共识机制

Substrate 提供了几个区块生成算法，也允许您创建自己的算法：

- Aura (轮询调度)
- BABE (基于时隙)
- 工作量证明

**分叉选择规则**

Substrate 允许你编写自定义分叉选择规则，或者从给定的规则中选择一个。 例如：

**最长链规则**

最长链规则简单来说，就是把最长的链作为最佳链。 Substrate provides this chain selection rule with the [`LongestChain` 结构体](https://substrate.dev/rustdocs/v3.0.0/sc_consensus/struct.LongestChain.html)。 GRANDPA 使用最长链规则进行投票。

**GHOST 规则**

GHOST（The Greedy Heaviest Observed SubTree，贪婪最重可观察子树）规则是指从创世区块开始，每一个分叉都是通过递归地选择包含最多区块的分支来解决的。

**区块生成**

区块链网络中的一些节点可以生成新区块，该过程称为“创建区块”。 究竟哪个节点可以创建区块，取决于你使用什么样的共识引擎。 在中心化网络中，可以是由单个节点来创建所有的区块。但在完全无许可网络中，必须要使用一种算法来选择每个区块的创建者。

工作量证明

在类似比特币这样的工作量证明系统中，任何节点在任意时刻都可以生成区块，只要它成功解决了一个计算密集型难题。 解决难题需要花费CPU时间，因此矿工只能使用他们的计算资源来生成区块。 Substrate 提供一个工作量证明区块生成引擎。

时隙

基于时隙的共识算法必须包含一组已知的、允许生成区块的验证者的集合。 时间被分成离散的时隙，在每个时隙中只有一部分验证人节点可生成一个区块。 指定哪些验证者可以创作区块的方法因共识引擎的不同而不同。 Substrate 提供 Aura 和 Babe，这两者都是基于时隙的区块生成引擎。

**BABE**

[BABE](https://substrate.dev/rustdocs/v3.0.0/sc_consensus_babe/index.html) also provides slot-based block authoring with a known set of validators. 在这些方面，它类似于Aura。 不同于AURA, 时隙分配是基于对可验证随机函数(VRF) 的计算。 在每一个*时段 (epoch)*中，每个验证节点得到一个权重。这个时段被分解为若干时隙，而验证节点在每个时隙里计算自己的VRF。 当一个验证者在该时隙的VRF输出值小于它的权重时，就可以创建一个区块。

由于同一个时隙中可能有多个验证者能够生成区块，分叉在BABE中要比Aura更常见，即使在良好的网络条件下也是常见的。

Substrate的BABE实现也提供了一个备用机制，用于在某个时隙中没有验证者被选中的情形。 分配这些“次要”时隙使得 BABE 实现了恒定的区块时间。

**最终确定性**

不管在什么系统中，用户都想知道他们的交易是什么时候确定完成的，区块链也一样。 在一些传统系统中，当收据交出或文件签名时，就是最终确定的了。

如果使用前述的区块创建模式和分叉选择规则，交易是永远不能最终确定的。 总是有机会出现一个更长的(或更重的) 链，从而回滚你的交易。 当然，在某个特定区块之上构建的区块越多，就越不可能回滚。 这样的话，采用适当的分叉选择规则的区块生成算法只提供了概率上的最终确定性。

当需要明确无误的最终确定性时，可以在区块链逻辑中添加一个最终确定性小工具。 固定验证人集合的成员为区块的最终性进行投票，当某个区块拥有足够数量的投票时，该区块将被视为是最终确定的。 在大多数系统中，这一阈值为2/3。 以这种方式确定的区块无法被回滚，除非采用像硬分叉这样的外部协调手段。

> 一些共识机制将区块生成和最终确定性组合在一起，就像这样：区块最终确定是区块生成流程的一部分，除非块 `N` 已最终确定，否则新的 `N+1` 块无法创建。 但是 Substrate 分离了这两个过程，允许你使用任何具备概率上保证最终性的区块生成引擎，或者将其与一个最终确定性配件结合起来以具备完全的的最终确定性。

在使用最终确定性小工具的系统中，必须修改分叉选择规则以考虑最终确定性博弈的结果。 例如，节点应该选取包含有最近的最终确定区块的最长链，而不是简单地选择最长链。

**GRANDPA**

[GRANDPA](https://substrate.dev/rustdocs/v3.0.0/sc_finality_grandpa/index.html) provides block finalization. 它有一个类似BABE的权威验证人集合。 然而 GRANDPA 并不生产区块； 它只是监听其它共识引擎（如上面讨论的三个引擎）所产生的区块。 GRANDPA 验证人对 *链*，而不是*区块*进行投票。换言之，当验证人对他们认为 "最优" 的区块投票时，他们同时也对该链前面的所有区块作了投票。 一旦超过 2/3 的 GRANDPA 验证人给某一个特定的区块投了票，这个区块就被视为是最终确定的。

由于BABE和GRANDPA都将被用于Polkadot网络，Web3基金会提供了研究级别的算法介绍。

- [BABE研究](https://research.web3.foundation/en/latest/polkadot/block-production/Babe.html)
- [GRANDPA 研究](https://research.web3.foundation/en/latest/polkadot/finality.html)

包括 GRANDPA 在内的完全最终性算法至少需要 `2f + 1` 个可靠节点， 其中 `f` 是错误或恶意节点的数量。 查看这些开创性的论文 [Reaching Agreement in the Presence of Faults](https://lamport.azurewebsites.net/pubs/reaching.pdf) 或[Wikipedia: Byzantine Fault](https://en.wikipedia.org/wiki/Byzantine_fault)，以了解更多的这些概念以及阈值的来源.

## 区块导入过程

导入队列是存在于每个 Substrate 节点中的抽象工作队列。 它并不是 Rumtime 的一部分。 导入队列负责处理并且验证外来信息，如果信息是有效的，就将它们导入到节点的状态中。 导入队列处理的最基本信息就是区块本身，同时它也负责导入与共识相关的消息，如理由声明等，以及在轻客户端中确定性证明。

导入队列从网络中搜集传入元素，并将它们存储到一个池子中。 这些元素随后会被检查是否有效，如果无效则会被丢弃。 有效的元素将被导入到节点的本地状态里。

**区块导入管道**

在最简单的情况下，区块会直接导入到客户端。 但是大多数共识引擎需要对导入的区块进行额外的验证，或者需要更新自己的本地备用数据库，或两者兼而有之。 为了让共识引擎能够达成其需求，我们通常会将客户端包在另一个同样实现了 `BlockImport` trait的结构体当中。 该嵌套方式就使 "区块导入管道 "这个术语应运而生。

`BlockImport` 的嵌套不止限于一层。 事实上，对于同时使用出块引擎和终块确认引擎的节点来说，多层次嵌套十分常见。 例如，Polkadot 的区块导入管道包含了 `BabeBlockImport` 模块，该模块封装了一个`GrandpaBlockImport`，而这个 GrandpaBlockImport 又封装了一层 `Client`实现。

## Runtime Executor

执行器负责向 Substrate Runtime 调度和执行调用。

Substrate Runtime 被编译成一个本地可执行文件和一个 WebAssembly（Wasm）二进制文件。

本机 Runtime 作为节点可执行文件的一部分，而 Wasm 二进制文件则存储在区块链上的一个已知的存储键值下。

**执行策略**

在 Runtime 开始执行之前，Substrate 客户端会建议使用哪个 Runtime 执行环境。 这是由执行策略控制的，可以针对区块链执行过程的不同部分进行配置。 The strategies are listed in the [`ExecutionStrategy` enum](https://substrate.dev/rustdocs/v3.0.0/sp_state_machine/enum.ExecutionStrategy.html):

- `NativeWhenPossible`: Execute with native build (if available, WebAssembly otherwise).
- `AlwaysWasm`: Only execute with the WebAssembly build.
- `Both`: Execute with both native (where available) and WebAssembly builds.
- `NativeElseWasm`: Execute with the native build if possible; if it fails, then execute with WebAssembly.

All strategies respect the runtime version, meaning if the native and wasm runtime versions differ (which the wasm runtime is more updated then the native one), the wasm runtime is chosen to run.

The default execution strategies for the different parts of the blockchain execution process are:

- 同步： `NativeElseWasm`
- Block Import (for non-validator): `NativeElseWasm`
- Block Import (for validator): `AlwaysWasm`
- Block Construction: `AlwaysWasm`
- Off-Chain Worker: `NativeWhenPossible`
- Other: `NativeWhenPossible`

Source: [[1\]](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/executor#footnote-execution-strategies-src01), [[2\]](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/executor#footnote-execution-strategies-src02)

They can be overridden via the command line argument `--execution-{block-construction, import-block, offchain-worker, other, syncing} <strategy>`, or `--execution <strategy>` to apply the specified strategy to all five aspects. Details can be seen at `substrate --help`. When specifying on cli, the following shorthand strategy names are used:

- `Native` mapping to the `NativeWhenPossible` strategy
- `Wasm` mapping to the `AlwaysWasm` strategy
- `Both` mapping to the `Both` strategy
- `NativeElseWasm` mapping to `NativeElsmWasm` strategy

**Wasm 执行**

The Wasm representation of the Substrate runtime is considered the canonical runtime. Because this Wasm runtime is placed in the blockchain storage, the network must come to consensus about this binary. Thus it can be verified to be consistent across all syncing nodes.

The Wasm execution environment can be more restrictive than the native execution environment. For example, the Wasm runtime always executes in a 32-bit environment with a configurable memory limit (up to 4 GB).

For these reasons, the blockchain prefers to do block construction with the Wasm runtime. Some logic executed in Wasm will always work in the native execution environment, but the same cannot be said the other way around. Wasm execution can help to ensure that block producers create valid blocks.

**本地执行**

The native runtime will only be used by the executor when it is chosen as the execution strategy and it is compatible with the requested runtime version (see [Runtime Versioning](https://substrate.dev/docs/en/knowledgebase/runtime/upgrades#runtime-versioning)). For all other execution processes other than block construction, the native runtime is preferred since it is more performant. In any situation where the native executable should not be run, the canonical Wasm runtime is executed instead.

## 密码学

**ECDSA**

Substrate 提供了一个使用 [secp256k1](https://en.bitcoin.it/wiki/Secp256k1) 曲线的[ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)签名方案。 这与用于保护 [Bitcoin](https://en.wikipedia.org/wiki/Bitcoin) 和 [Ethereum](https://en.wikipedia.org/wiki/Ethereum) 安全的加密算法相同。

**Ed25519**

[Ed25519](https://en.wikipedia.org/wiki/EdDSA#Ed25519) 是一个基于 [Curve25519曲线](https://en.wikipedia.org/wiki/Curve25519) 的EdDSA签名方案， 它在设计和实现的多个层面上做了精心的处理，在不影响安全性的前提下实现了非常高的速度。

**SR25519**

[SR25519](https://research.web3.foundation/en/latest/polkadot/keys/1-accounts-more.html) 是基于与 [Ed25519](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/cryptography#ed25519) 相同的基础曲线算法。 但是，它使用 Schnorr 签名替换了 EdDSA 的原有方案。

相比于 [ECDSA](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/cryptography#ecdsa)/EdDSA 方案，Schnorr 签名具有一些显著的特性：

- 它更适用于分层确定性的密钥推导。
- 它通过使用 [signature aggregation](https://bitcoincore.org/en/2017/03/23/schnorr-signature-aggregation/) 实现本机多签。
- 它总体上更能防止滥用。

当使用 Schnorr 签名替代ECDSA 方案时，一个不好的地方是，虽然两者都要占用64个字节，但只有 ECDSA 签名才会传递其公钥。

## 存储

Substrate 区块链对外开放了远端过程调用 (RPC) 服务，该服务可用于查询 runtime 的存储。 当您使用 Substrate RPC 访问一个存储项时，您只需要提供与该存储项关联的 [键](https://substrate.dev/docs/zh-CN/knowledgebase/advanced/storage#Key-Value-Database)。 [Substrate 的 Runtime 存储 APIs](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/storage) 开放了数种存储项类型；请继续阅读来了解如何为不同类型的存储项计算其存储键。

To calculate the key for a simple [Storage Value](https://substrate.dev/docs/zh-CN/knowledgebase/runtime/storage#storage-value), take the [TwoX 128 hash](https://github.com/Cyan4973/xxHash) of the name of the module that contains the Storage Value and append to it the TwoX 128 hash of the name of the Storage Value itself. For example, the [Sudo](https://substrate.dev/rustdocs/v3.0.0/pallet_sudo/index.html) pallet exposes a Storage Value item named [`Key`](https://substrate.dev/rustdocs/v3.0.0/pallet_sudo/struct.Module.html#method.key)：

```
twox_128("Sudo")                   = "0x5c0d1176a568c1f92944340dbfed9e9c"
twox_128("Key")                    = "0x530ebca703c85910e7164cb7d1c9e47b"
twox_128("Sudo") + twox_128("Key") = "0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b"
```

如果 `Alice` 账号是 sudo 用户，读取 Sudo 模块的 `Key` 存储值的 RPC 请求和响应可表示为：

```
state_getStorage("0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b") = "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"
```

## SS58 地址格式

SS58 是一个简单的地址格式，设计用于基于 Substrate 开发的链。 使用其他地址格式也是没有问题的，但 SS58 是一个可靠的默认项。 它在很大程度上基于比特币的 Base-58-check 格式，并作了一些修改。基本思路是用一个 base-58 编码值代表一个在 Substrate 链上的特定帐户。 不同的链有不同的识别帐户的方法。 因此，SS58 被设计为可扩展的地址格式。您可以在 Substrate GitHub Wiki 上找到当前 SS58 地址格式的规范：https://github.com/paritytech/substrate/wiki/External-Address-Format-(SS58)

## Recipes

![Substrate Architecture Diagram](https://substrate.dev/recipes/img/substrate-architecture.png)

### Pallets

Pallets are individual pieces of runtime logic for use in FRAME runtimes.

#### Events

Having a [transaction](https://substrate.dev/docs/en/knowledgebase/getting-started/glossary#transaction) included in a block does not guarantee that the function executed successfully. To verify that functions have executed successfully, emit an [event](https://substrate.dev/docs/en/knowledgebase/getting-started/glossary#events) at the bottom of the function body.

Events notify the off-chain world of successful state transitions.

```rust
// Simple Events
pub trait Config: frame_system::Config {
    type Event: From<Event> + Into<<Self as frame_system::Config>::Event>;
}
decl_event!(
    pub enum Event {
        EmitInput(u32),
    }
);
// Events with Generic Types
pub trait Config: frame_system::Config {
	type Event: From<Event<Self>> + Into<<Self as frame_system::Config>::Event>;
}
decl_event!(
    pub enum Event<T> where AccountId = <T as frame_system::Config>::AccountId {
        EmitInput(AccountId, u32),
    }
);
decl_module! {
    pub struct Module<T: Config> for enum Call where origin: T::Origin {

        // This line is new
        fn deposit_event() = default;

        /// A simple call that does little more than emit an Simple event
        #[weight = 10_000]
        fn do_something(origin, input: u32) -> DispatchResult {
          let _ = ensure_signed(origin)?;

          // In practice, you could do some processing with the input here.
          let new_number = input;

          // emit event
          Self::deposit_event(Event::EmitInput(new_number));
          Ok(())
        }
      
        /// A simple call that does little more than emit an Generic event
        #[weight = 10_000]
        fn do_something(origin, input: u32) -> DispatchResult {
          let user = ensure_signed(origin)?;

          // could do something with the input here instead
          let new_number = input;

          Self::deposit_event(RawEvent::EmitInput(user, new_number));
          Ok(())
        }
    }
}
```

Constructing the Runtime

```rust
impl simple_event::Config for Runtime {
    type Event = Event;
}
impl generic_event::Config for Runtime {
	type Event = Event;
}
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        // --snip--
        GenericEvent: generic_event::{Module, Call, Event<T>},
        SimpleEvent: simple_event::{Module, Call, Event},
    }
);
```

#### Storage

**Maps**

Declaring a StorageMap

```rust
decl_storage! {
    trait Store for Module<T: Config> as SimpleMap {
        SimpleMap get(fn simple_map): map hasher(blake2_128_concat) T::AccountId => u32;
    }
}
```

- `SimpleMap` - the name of the storage map
- `get(fn simple_map)` - the name of a getter function that will return values from the map.
- `: map hasher(blake2_128_concat)` - beginning of the type declaration. This is a map and it will use the [`blake2_128_concat`](https://substrate.dev/rustdocs/v3.0.0/frame_support/trait.Hashable.html#tymethod.blake2_128_concat) hasher. More on this below.
- `T::AccountId => u32` - The specific key and value type of the map. This is a map from `AccountId`s to `u32`s.

**Choosing a Hasher**

https://substrate.dev/recipes/storage-maps.html#choosing-a-hasher

**The Storage Map API**

```rust
// Insert
<SimpleMap<T>>::insert(&user, entry);

// Get
let entry = <SimpleMap<T>>::get(account);

// Take
let entry = <SimpleMap<T>>::take(&user);

// Contains Key
<SimpleMap<T>>::contains_key(&user)
```

The rest of the API is documented in the rustdocs on the [`StorageMap` trait](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.StorageMap.html).

**Cache Multiple Calls**

```rust
decl_storage! {
    trait Store for Module<T: Config> as StorageCache {
        // copy type
        SomeCopyValue get(fn some_copy_value): u32;

        // clone type
        KingMember get(fn king_member): T::AccountId;
        GroupMembers get(fn group_members): Vec<T::AccountId>;
    }
}
```

[trait.StorageValue](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.StorageValue.html)

Copy Types

For [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) types, it is easy to reuse previous storage calls by simply reusing the value, which is automatically cloned upon reuse.

Clone Types

If the type was not `Copy`, but was [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html), then it is still better to clone the value in the method than to make another call to runtime storage.

**Using Vectors as Sets**

**Using Maps as Sets**

Iterating over all items in a `map-set` is achieved by using the [`IterableStorageMap` trait](https://substrate.dev/rustdocs/v3.0.0/frame_support/storage/trait.IterableStorageMap.html), which iterates `(key, value)` pairs

**Double Maps**

```rust
pub type GroupIndex = u32; // this is Encode (which is necessary for double_map)

decl_storage! {
    trait Store for Module<T: Config> as Dmap {
        /// Member score (double map)
        MemberScore get(fn member_score):
            double_map hasher(blake2_128_concat) GroupIndex, hasher(blake2_128_concat) T::AccountId => u32;
        /// Get group ID for member
        GroupMembership get(fn group_membership): map hasher(blake2_128_concat) T::AccountId => GroupIndex;
        /// For fast membership checks, see check-membership recipe for more details
        AllMembers get(fn all_members): Vec<T::AccountId>;
    }
}

fn remove_group_score(origin, group: GroupIndex) -> DispatchResult {
    let member = ensure_signed(origin)?;

    let group_id = <GroupMembership<T>>::get(member);
    // check that the member is in the group
    ensure!(group_id == group, "member isn't in the group, can't remove it");

    // remove all group members from MemberScore at once
    <MemberScore<T>>::remove_prefix(&group_id);

    Self::deposit_event(RawEvent::RemoveGroup(group_id));
    Ok(())
}
```

**Using and Storing Structs**

```rust
use frame_support::codec::{Encode, Decode};

#[derive(Encode, Decode, Default, Clone, PartialEq)]
pub struct MyStruct {
    some_number: u32,
    optional_number: Option<u32>,
}
```

Structs with Generic Fields

```rust
#[derive(Encode, Decode, Clone, Default, RuntimeDebug)]
pub struct InnerThing<Hash, Balance> {
    number: u32,
    hash: Hash,
    balance: Balance,
}
type InnerThingOf<T> =
	InnerThing<<T as frame_system::Config>::Hash, <T as pallet_balances::Config>::Balance>;

#[derive(Encode, Decode, Default, RuntimeDebug)]
pub struct SuperThing<Hash, Balance> {
	super_number: u32,
	inner_thing: InnerThing<Hash, Balance>,
}

decl_storage! {
	trait Store for Module<T: Config> as NestedStructs {
		InnerThingsByNumbers get(fn inner_things_by_numbers):
			map hasher(blake2_128_concat) u32 => InnerThingOf<T>;
		SuperThingsBySuperNumbers get(fn super_things_by_super_numbers):
			map hasher(blake2_128_concat) u32 => SuperThing<T::Hash, T::Balance>;
	}
}

fn insert_inner_thing(origin, number: u32, hash: T::Hash, balance: T::Balance) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    let thing = InnerThing {
                    number,
                    hash,
                    balance,
                };
    <InnerThingsByNumbers<T>>::insert(number, thing);
    Self::deposit_event(RawEvent::NewInnerThing(number, hash, balance));
    Ok(())
}
```

**Ringbuffer Queue**

**Basic Token**

Don't Panic! In Substrate, **your runtime must never panic**.

**Configurable Pallet Constants**

To declare constant values within a runtime, it is necessary to import the [`Get`](https://substrate.dev/rustdocs/v3.0.0/frame_support/traits/trait.Get.html) trait from `frame_support`

```rust
use frame_support::traits::Get;
```

Configurable constants are declared as associated types in the pallet's configuration trait using the `Get<T>` syntax for any type `T`.

```rust
pub trait Config: frame_system::Config {
    type Event: From<Event> + Into<<Self as frame_system::Config>::Event>;

    /// Maximum amount added per invocation
    type MaxAddend: Get<u32>;

    /// Frequency with which the stored value is deleted
    type ClearFrequency: Get<Self::BlockNumber>;
}
```

In order to make these constants and their values appear in the runtime metadata, it is necessary to declare them with the `const` syntax in the `decl_module!` block. Usually constants are declared at the top of this block, right after `fn deposit_event`.

```rust
decl_module! {
    pub struct Module<T: Config> for enum Call where origin: T::Origin {
        fn deposit_event() = default;

        const MaxAddend: u32 = T::MaxAddend::get();

        const ClearFrequency: T::BlockNumber = T::ClearFrequency::get();

        // --snip--
    }
}
```

This example manipulates a single value in storage declared as `SingleValue`.

```rust
decl_storage! {
    trait Store for Module<T: Config> as Example {
        SingleValue get(fn single_value): u32;
    }
}
```

`SingleValue` is set to `0` every `ClearFrequency` number of blocks in the `on_finalize` function that runs at the end of blocks execution.

```rust
fn on_finalize(n: T::BlockNumber) {
    if (n % T::ClearFrequency::get()).is_zero() {
        let c_val = <SingleValue>::get();
        <SingleValue>::put(0u32);
        Self::deposit_event(Event::Cleared(c_val));
    }
}
```

Signed transactions may invoke the `add_value` runtime method to increase `SingleValue` as long as each call adds less than `MaxAddend`. *There is no anti-sybil mechanism so a user could just split a larger request into multiple smaller requests to overcome the `MaxAddend`*, but overflow is still handled appropriately.

```rust
fn add_value(origin, val_to_add: u32) -> DispatchResult {
    let _ = ensure_signed(origin)?;
    ensure!(val_to_add <= T::MaxAddend::get(), "value must be <= maximum add amount constant");

    // previous value got
    let c_val = <SingleValue>::get();

    // checks for overflow when new value added
    let result = match c_val.checked_add(val_to_add) {
        Some(r) => r,
        None => return Err(DispatchError::Other("Addition overflowed")),
    };
    <SingleValue>::put(result);
    Self::deposit_event(Event::Added(c_val, val_to_add, result));
    Ok(())
}
```

In more complex patterns, the constant value may be used as a static, base value that is scaled by a multiplier to incorporate stateful context for calculating some dynamic fee (i.e. floating transaction fees).

Supplying the Constant Value

When the pallet is included in a runtime, the runtime developer supplies the value of the constant using the [`parameter_types!` macro](https://substrate.dev/rustdocs/v3.0.0/frame_support/macro.parameter_types.html). This pallet is included in the `super-runtime` where we see the following macro invocation and trait implementation.

```rust
parameter_types! {
    pub const MaxAddend: u32 = 1738;
    pub const ClearFrequency: u32 = 10;
}

impl constant_config::Config for Runtime {
    type Event = Event;
    type MaxAddend = MaxAddend;
    type ClearFrequency = ClearFrequency;
}
```

#### Simple Crowdfund

#### Instantiable Pallets

Instantiable pallets enable multiple instances of the same pallet logic within a single runtime. Each instance of the pallet has its own independent storage, and extrinsics must specify which instance of the pallet they are intended for.

#### Weights

#### Charity & Imbalances

Our charity needs an account to hold its funds. Unlike other accounts, it will not be controlled by a user's cryptographic key pair, but directly by the pallet. To instantiate such a pool of funds, import [`ModuleId`](https://substrate.dev/rustdocs/v3.0.0/sp_runtime/struct.ModuleId.html) and [`AccountIdConversion`](https://substrate.dev/rustdocs/v3.0.0/sp_runtime/traits/trait.AccountIdConversion.html) from [`sp-runtime`](https://substrate.dev/rustdocs/v3.0.0/sp_runtime/index.html).

The `PALLET_ID` must be exactly eight characters long.

#### Fixed Point Arithmetic

#### Off-chain Workers

HTTP Fetching and JSON Parsing

#### Currency Types

#### Currency Imbalances

#### Generating Randomness

#### Controlling Access

### Runtime

#### Runtime APIs

Each Substrate node contains a runtime. The runtime contains the business logic of the chain. It defines what transactions are valid and invalid and determines how the chain's state changes in response to transactions. The runtime is compiled to Wasm to facilitate runtime upgrades. The "outer node", everything other than the runtime, does not compile to Wasm, only to native. The outer node is responsible for handling peer discovery, transaction pooling, block and transaction gossiping, consensus, and answering RPC calls from the outside world. While performing these tasks, the outer node sometimes needs to query the runtime for information, or provide information to the runtime. A Runtime API facilitates this kind of communication between the outer node and the runtime.