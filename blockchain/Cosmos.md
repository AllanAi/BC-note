# Blockchain Architecture

## State machine

At its core, a blockchain is a [replicated deterministic state machine](https://en.wikipedia.org/wiki/State_machine_replication).

A state machine is a computer science concept whereby a machine can have multiple states, but only one at any given time. There is a `state`, which describes the current state of the system, and `transactions`, that trigger state transitions.

In a blockchain context, the state machine is deterministic. This means that if a node is started at a given state and replays the same sequence of transactions, it will always end up with the same final state.

# Main Components of the Cosmos SDK

SDK modules are defined in the `x/` folder of the SDK. Some core modules include:

- `x/auth`: Used to manage accounts and signatures.
- `x/bank`: Used to enable tokens and token transfers.
- `x/staking` + `x/slashing`: Used to build Proof-Of-Stake blockchains.

# Anatomy of an SDK Application

## Handler

The [`handler`](https://docs.cosmos.network/v0.39/building-modules/handler.html) refers to the part of the module responsible for processing the `message` after it is routed by `baseapp`. `handler` functions of modules are only executed if the transaction is relayed from Tendermint by the `DeliverTx` ABCI message. If the transaction is relayed by `CheckTx`, only stateless checks and fee-related stateful checks are performed. To better understand the difference between `DeliverTx`and `CheckTx`, as well as the difference between stateful and stateless checks, click [here](https://docs.cosmos.network/v0.39/basics/tx-lifecycle.html).

