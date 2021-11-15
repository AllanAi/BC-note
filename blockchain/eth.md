## Go Ethereum

https://github.com/ethereum/go-ethereum

https://blog.csdn.net/gduyt_gduyt/category_8794456.html

### 编译

make geth
cp build/bin/geth $GOPATH/bin/

### 命令行

**本地测试链**

geth --datadir eth init eth-genesis.json

```json
{
  "config": {
    "chainId": 22,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0
  },
  "alloc": {
    "0x0000000000000000000000000000000000000001": {
      "balance": "111111111"
    },
    "0x0000000000000000000000000000000000000002": {
      "balance": "222222222"
    }
  },
  "coinbase": "0x0000000000000000000000000000000000000000",
  "difficulty": "0x20000",
  "extraData": "",
  "gasLimit": "0x2fefd8",
  "nonce": "0x0000000000000042",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp": "0x00"
}
```


geth --datadir eth --nodiscover console 2>>geth.log

| 参数         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| datadir      | 指明当前区块链私钥和网络数据存放的位置                       |
| port         | 指定以太坊网络监听端口，默认为30303                          |
| rpc          | 开启HTTP-RPC服务，可以进行智能合约的部署和调试               |
| rpcaddr      | 指定HTTP-RPC服务监听地址，默认为“localhost"                  |
| rpcapi       | 设置允许连接的rpc的客户端，一般为db、eth、net、web3          |
| rpcport      | 指定HTTP-RPC服务监听端口，默认为8545                         |
| networkid    | 指定以太坊id，其实就是区块链网络的身份标识，共有链为1，测试链为3，默认启动id为1 |
| etherbase    | 指定矿工帐号，默认为keystory中首个帐号                       |
| mine         | 开启挖矿，默认为CPU挖矿                                      |
| minerthreads | 挖矿占用CPU线程数，默认为4                                   |
| nodiscover   | 关闭自动连接节点，但可以手动添加节点，在搭建私有链时，为避免其他节点连入私有链，可使用该命令 |
| maxpeers     | 设置允许最大节点数，默认为25                                 |
| console      | 启动命令行模式，可以在geth中执行命令                         |

```
geth --datadir "./db1" account new //创建新帐号
geth --datadir "./db1" account list //列举已存在帐号
geth --datadir "./db1" account update b1d70e94ffcba142fd024ece374e0be3cd9c08ad //修改账户密码
geth --datadir "./db1" account import ecc.key //导入密钥文件
geth --datadir "./db1" export ./bak //导出区块数据
geth --datadir "./db1" removedb //移除区块数据
geth --datadir "./db1" init gensis.json //导入区块数据之前要用gensis.json文件执行初始化
geth --datadir "./db1" import ./bak //初始化完成后就可以导入区块数据
//通过attach命令连接已启动节点，当通过geth命令启动了一个以太坊私有链时，会在数据目录下生成一个geth.ipc文件
geth --datadir "./db" attach ipc:./db/geth.ipc
```

## Web3

https://web3js.readthedocs.io/

以太坊的javascript控制台中内置了一些以太坊对象，通过这些对象我们可以很方便的与以太坊交互：

- ​    eth：提供了操作区块链相关的方法
- ​    net：提供了查看p2p网络状态的方法
- ​    admin：提供了管理节点相关的方法
- ​    miner：提供启动和停止挖矿的方法
- ​    personal：提供了管理账户的方法
- ​    txpool：提供了查看交易内存池的方法
- ​    web3：除了包含以上对象中有的方法，还包含了一些单位换算的方法

```js
eth.accounts //查看账户
personal.newAccount(password) //创建账户，password为空时，会提示输入密码
eth.getBalance(eth.accounts[0]) //查看余额，1 ether = 10^18 wei
miner.setEtherbase(eth.accounts[0]) //在挖矿之前要先设置挖矿奖励地址，默认为创建的第一个账户地址
eth.coinbase //设置完成后，查看是否设置成功
miner.start() //开始挖矿
personal.unlockAccount(eth.accounts[2][,password,time])//参数中传入账户地址、密码、账户解锁状态持续时间
eth.sendTransaction({from: eth.accounts[2], to: eth.accounts[3],value: web3.toWei(1,"ether")})
txpool.status //查看交易池
txpool.inspect.pending //查看pending交易详情
miner.start(1);admin.sleepBlocks(1);miner.stop(); //挖到一个区块之后就停止挖矿
eth.getTransaction("0x28f7e6989893d6e8b1cd26d5d7a285654f5a3c8eff7d6b2029817496deb8bda0") //查看指定交易哈希值 所对应交易 被发起时的交易详情
eth.getTransactionReceipt("0x28f7e6989893d6e8b1cd26d5d7a285654f5a3c8eff7d6b2029817496deb8bda0") //查看指定交易哈希值 所对应交易 被打包进区块时的详细信息
eth.blockNumber //查看当前区块总数
eth.getBlock("latest") //查询最新区块
eth.getBlock(blockNumber/blockHash) //根据区块Number或Hash查询区块

admin.nodeInfo //查看节点信息
admin.nodeInfo.enode //获取encode信息
admin.addPeer(enode) //添加其他节点
admin.peers //查看已连接的远程节点
```

## Remix

https://remix-ide.readthedocs.io/en/latest/index.html

https://remix.ethereum.org/

https://github.com/ethereum/remix-project

**Enable optimization**: defines if the compiler should enable optimization during compilation. Enabling this option saves execution gas. It is useful to enable optimization for contracts ready to be deployed in production but could lead to some inconsistencies when debugging such a contract.

**Gas Limit**: This sets the maximum amount of gas that will be allowed for all the transactions created in Remix.
**Value**: This sets the amount of ETH, WEI, GWEI etc that is sent to a contract or a payable function. ( Note: payable functions have a red button). The value is always reset to 0 after each transaction execution). The Value field is NOT for gas.

**At Address**: this is used at access a contract that has already been deployed. It assumes that the given address is an instance of the selected contract. Note: There’s no check at this point, so be careful when using this feature, and be sure you trust the contract at that address.

**Using the ABI**
Using Deploy or At Address is a classic use case of Remix. However, it is possible to interact with a contract by using its ABI. The ABI is a JSON array which describe its interface.
To interact with a contract using the ABI, create a new file in Remix with extension *.abi and copy the ABI content to it. Then, in the input next to At Address, put the Address of the contract you want to interact with. Click on At Address, a new “connection” with the contract will popup below.

**Inputting parameters in the collapsed view** (Inputting all the parameters in a single input box)
The input box tells you what type each parameter needs to be.
Numbers and addresses do not need to be wrapped in double quotes.
Strings need to be wrapped.
Parameters are separated by commas.

**Debug**: If you add a breakpoint to a line that declares a variable, it might be triggered twice: Once for initializing the variable to zero and second time for assigning the actual value. 

**Importing Source Files in Solidity** https://remix-ide.readthedocs.io/en/latest/import.html

**Example of passing nested struct to a function**

Consider a nested struct defined like this:

```
struct gardenPlot {
    uint slugCount;
    uint wormCount;
    Flower[] theFlowers;
}
struct Flower {
    uint flowerNum;
    string color;
}
```

If a function has the signature `fertilizer(Garden memory gardenPlot)` then the correct syntax is:

```
[1,2,[[3,"Petunia"]]]
```

### remixd

```
npm install -g remixd
remixd -s gopath/src/github.com/cosmos/peggy/testnet-contracts/ --remix-ide http://localhost:8080
remixd -s gopath/src/github.com/cosmos/peggy/testnet-ctracts/ --remix-ide https://remix.ethereum.org
```

### 连接到本地节点

To run Remix using https://remix.ethereum.org & a local test node, use this Geth command:

```
geth --rpc --rpccorsdomain="https://remix.ethereum.org" --rpcapi web3,eth,debug,personal,net --vmdebug --datadir <path/to/local/folder/for/test/chain> --dev console

geth --rpc --rpccorsdomain=* --rpcapi web3,eth,debug,personal,net --vmdebug --datadir <path/to/local/folder/for/test/chain> --dev console
```

To run Remix Desktop & a local test node, use this Geth command:

```
geth --rpc --rpccorsdomain="package://a7df6d3c223593f3550b35e90d7b0b1f.mod" --rpcapi web3,eth,debug,personal,net --vmdebug --datadir <path/to/local/folder/for/test/chain> --dev console
```

The Web3 Provider Endpoint for a local node is http://localhost:8545



geth --datadir data/ --dev init eth-genesis.json
geth --datadir data/ --nodiscover --rpc --rpccorsdomain=* --rpcapi web3,eth,debug,personal,net --vmdebug --dev console 2>>geth.log
