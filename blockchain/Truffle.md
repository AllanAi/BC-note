# Truffle

文档：https://www.trufflesuite.com/docs/truffle/overview



## Problems

- Compile contract path into bytecode: https://github.com/trufflesuite/truffle-compile/issues/77 间接导致不同电脑上编译出来的bytecode不一致，对于需要使用`CREATE2`的合约很不友好。
- "callback already called" error：https://github.com/trufflesuite/truffle/issues/3008 将node降级到LTS版本



## Verify

Verify your deployed smart contracts on Etherscan from the Truffle CLI

https://github.com/rkalis/truffle-plugin-verify

https://kalis.me/verify-truffle-smart-contracts-etherscan/



## Box

A set of boilerplates to help developers quickly build distributed applications on the Ethereum blockchain with Truffle.

 https://github.com/truffle-box

[Error while unboxing...](https://github.com/trufflesuite/truffle/issues/2692)：由于网络问题，无法使用`truffle unbox`命令时，可以直接使用`git clone`对应的项目。



## Contract Size

Displays the size of a truffle contracts in kilobytes

https://github.com/IoBuilders/truffle-contract-size

减少合约的大小的方法：https://soliditydeveloper.com/max-contract-size



## Flattener

Truffle Flattener concats solidity files from Truffle and Buidler projects with all of their dependencies.

https://github.com/nomiclabs/truffle-flattener

使用方法：https://www.sitepoint.com/flattening-contracts-debugging-remix/

`truffle-flattener contracts/SimpleToken.sol > FlattenedSimpleToken.sol`



## Deploy

Deploy with Infura：https://medium.com/coinmonks/deploy-your-smart-contract-directly-from-truffle-with-infura-ba1e1f1d40c2