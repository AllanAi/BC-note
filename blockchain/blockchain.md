# 以太坊黄皮书

https://github.com/wanshan1024/ethereum_yellowpaper

工作量证明是区块链的安全基础，也是一个恶意节点不能用其新创建的区块来覆盖（“重写”）历史数据的重要原因。因为这个随机数必须满足这些条件，且因为条件依赖于这个区块的内容和相关交易，创建新的合法的区块是困难且耗时的，需要超过所有诚实矿工的算力总和。

工作量证明不仅是在未来使区块链的权威的获得保障的安全信任方法，同时也是一个利益分配机制。

ASIC 抗性的设计有两个方向：第一是去让它成为内存困难（memory-hard）的问题所产生结果，即，设计一个需要大量的内存和带宽，来使这些内存不能被用于并行地计算nonce。第二个方向是让计算变得更有通用目的（generalpurpose）；一个为通用目的“定制的硬件”的意思，就是使得类似于普通的桌面计算机这样的硬件可以适合这种计算。在以太坊1.0 中，我们选择了第一个方向。

权威的区块链是在整个的区块树中从根节点到叶子节点的路径。为了形成路径共识，概念上我们选择具有最大的计算量的，或者说是最重的路径来识别它。帮助我们识别最重路径的一个明显事实就是叶子节点的区块数量，等于在路径中除了没有挖矿的创世区块以外的区块数量。路径越长，到达叶子节点的总体挖矿工作量就越大。这和已有的比特币及其衍生协议类似。

以太坊虚拟机（Ethereum Virtual Machine - EVM）。它是一个准图灵机，这个“准”的限定来源于其中的运算是通过参数gas 来限制的，也就是限定了可以执行的运算总量。

EVM 是一个简单的基于栈的架构。其中字（Word）的大小（也就是栈中数据项的大小）是256 位。这是为了便于执行Keccak-256 位哈希和椭圆曲线计算。其内存（memory）模型是简单地基于字寻址（word-addressed）的字节数组。栈的最大深度为1024。EVM 也有一个独立的存储（storage）模型；它类似于内存的概念，但并不是一个字节数组，而是一个基于字寻址（word-addressable）的字数组（word array）。

# 精通比特币

https://github.com/inoutcode/bitcoin_book_2nd

## 术语

地址（address）
比特币的地址看起来是这样的： 1DSrfJdB2AnWaFNgSbv3MZC2m74996JafV。它由一串字母和数字组成。它实际上是一个公共密钥160位哈希的base58check编码版本。

确认（confirmations）
一旦交易包含在一个区块中，它就有一个确认。只要有另一个区块在同一个区块链上开采，交易就会有两个确认信息，依此类推。六个或更多的确认被认为足以证明交易无法撤销。

难度重新计算（difficulty retargeting）
每产生2,016个块，全网重新计算一次挖矿难度。

难度目标（difficulty target）
使网络中平均10分钟生产一个区块的难度。

椭圆曲线数字签名算法（ECDSA）
ECDSA (Elliptic Curve Digital Signature Algorithm）是比特币使用的一种加密算法，以确保资金只能由其合法所有者使用。

交易费（fees）
交易的发起人通常会向网络提供交易处理的费用。大多数交易需要0.5mBTC的最低费用。

哈希锁（hashlocks）
哈希锁是一种限制一笔输出在指定的数据公开前不能被消费的财产留置权。哈希锁非常有用，一旦一把哈希锁被打开，任何其他使用相同密钥保护的哈希锁也会被打开。这使得我们可以创建多个输出，这些输出都被同一个哈希锁留置，并且可以在同一时间变成可消费的。

哈希时间锁定合约（HTLC）
哈希时间合约（Hashed TimeLock Contract）或HTLC是一种支付类型，它使用哈希锁和时间锁来要求一笔支付的收款方要么在指定日期之前通过生成加密收款证明，要么放弃接受支付的权力，将其返还给支付方。

闪电网络（Lightning Networks）
闪电网络是带有双向支付渠道的哈希时间锁合约（HTLC）的建议实现，其允许多笔支付在多个点对点支付渠道上安全路由。这样就可以形成一个网络，网络中的任何一点都可以向任何其他点发起支付，即使他们之间没有直接通道。

脚本（Script）
比特币使用脚本系统进行交易。脚本很简单，基于堆栈，并且从左到右进行处理。它故意设计成不是图灵完备的，不支持循环。

未花费交易输出（unspent transaction output，UTXO）
UTXO是一项未花费的交易输出，可以作为新交易的输入使用。

## 一、概述

比特币代表了数十年密码学和分布式系统研究的一个高峰，以独特而强大的方式将四项关键创新融合在一起：

- 去中心化的点对点网络（比特币协议）

- 公开的账本（区块链）

- 一套独立交易验证和货币发行的规则（共识协议）

- 在有效的区块链上达成全球分散共识的机制（PoW工作证明机制）



纸币发行者使用日益复杂的纸张和印刷技术来对抗伪造问题。物理货币可以轻易解决双重支付问题，因为同一张纸币不可能在两个地方出现。当然，传统货币也经常以数字方式存储和传输。 在这种情况下，伪造和双重支付问题的方法是通过中央当局来清算所有电子交易，他们对流通中的货币具有全局视野。

## 二、比特币如何运转

许多比特币交易的输出既引用新所有者的地址，又引用当前所有者的地址（这称为_找零_地址）。

在汇集输入以执行用户的支付请求时，不同的钱包可以使用不同的策略。他们可能会汇集很多小的输入，或者使用等于或大于期望付款的输入。除非钱包能够按照付款和交易费用的总额精确汇集输入，否则钱包将需要产生一些零钱。这与人们处理现金非常相似。如果你总是使用口袋里最大的钞票，那么最终你会得到一个充满零钱的口袋。如果你只使用零钱，你将永远只有大额账单。人们潜意识地在这两个极端之间寻找平衡点，比特币钱包开发者努力编程实现这种平衡。

来自一个交易的输出可以用作新交易的输入，因此当价值从一个所有者转移到另一个所有者时会产生一个所有权链。

作为完整节点客户端运行的比特币钱包应用实际上包含区块链中每笔交易的未使用输出的副本。这允许钱包创建交易输入，以及快速验证传入的交易具有正确的输入。但是，由于全节点客户端占用大量磁盘空间，所以大多数用户钱包运行“轻量级”客户端，仅跟踪用户自己未使用的输出。

任何遵守比特币协议，加入到比特币网络的系统，如服务器，桌面应用程序或钱包，都称为_比特币节点（bitcoin node）_。任何比特币节点接收到一个它没见过的有效交易之后，会立即转发到它连接到的所有其他节点，这被称为_泛洪（flooding）_传播技术。因此，事务在点对点网络中迅速传播，可在几秒钟内达到大部分节点。

即使Alice的钱包通过其他节点发送交易，它也会在几秒钟内到达Bob的钱包。Bob的钱包会立即将Alice的交易识别为收款，因为它包含可由Bob的私钥提取的输出。Bob的钱包应用还可以独立验证交易数据是格式正确的，使用的是之前未花费的输入，并且包含足够的交易费用以包含在下一个区块中。此时，鲍勃可以认为风险很小，即交易将很快包含在一个区块中并得到确认。

> 关于比特币交易的一个常见误解是，它们必须等待10分钟新区块的产生才能被“确认”，或者最多60分钟才能完成6个确认。虽然确认确保交易已被整个网络所接受，但对于诸如一杯咖啡等小值物品，这种延迟是不必要的。商家可以接受没有确认的有效小额交易。没有比没有身份或签名的信用卡支付风险更大的了，商家现在也经常接受。

## 四、密钥和地址

大多数情况下，比特币地址是从公钥生成的并且对应于公钥。但是，并非所有的比特币地址都代表公钥；他们也可以代表其他受益者，如脚本。

发现了一些合适的数学函数，例如素数指数运算和椭圆曲线乘法。这些数学函数实际上是不可逆的，这意味着它们很容易在一个方向上计算，但在相反方向上计算是不可行的。

私钥（k）是一个数字，通常随机选取。我们使用椭圆曲线乘法（单向加密函数）通过私钥生成公钥（K）。从公钥（K）中，我们使用单向加密哈希函数来生成比特币地址（A）。

为什么在比特币中使用非对称加密技术？因为它不是用来“加密”（保密）交易的。相反，非对称加密的有用特性是产生数字签名。私钥可用于为交易生成指纹（数字签名）。这个签名只能由知道私钥的人制作。但是，任何有权访问公钥和交易指纹的人都可以使用它们来验证签名确实是私钥的拥有者生成的。非对称加密的这一有用特性使任何人都可以验证每笔交易的每个签名，同时确保只有私钥所有者才能生成有效的签名。

生成密钥的第一步也是最重要的一步是找到一个安全的熵源或随机数。创建比特币私钥本质上与“选择一个1到2^256之间的数字”相同。只要保证不可预测性和不可重复性，用于选择该数字的确切方法并不重要。

从公钥 *K* 开始，我们计算它的SHA256哈希值，然后再计算结果的RIPEMD160哈希值，产生一个160位（20字节）的数字，最后用Base58Check编码生产地址。

Base64表示使用26个小写字母，26个大写字母，10个数字和另外2个字符（如 “+” 和 “/” ）在基于文本的媒体（如电子邮件）上传输二进制数据。Base64最常用于向电子邮件添加二进制附件。Base58是一种基于文本的二进制编码格式，用于比特币和许多其他加密货币。它在紧凑表示，可读性和错误检测与预防之间提供了平衡。Base58是Base64的一个子集，使用大小写字母和数字，省略了一些经常被混淆的，或在使用某些字体显示时看起来相同的。 具体来说，相比Base64，Base58没有0（数字0），O（大写o），l（小写L），I（大写i）和符号“`+`”和“/”。

为了增加防范输入错误或转录错误的额外安全性，Base58Check是有内置错误校验代码的Base58编码格式，经常在比特币中使用。校验和是添加到编码数据末尾的四个字节。校验和来自编码数据的哈希散列值，可用于检测和防止转录和输入错误。当使用Base58Check代码时，解码软件将计算数据的校验和并将其与代码中包含的校验和进行比较。如果两者不匹配，则会引入错误并且Base58Check数据无效。这可以防止错误的比特币地址被钱包软件接收，导致资金损失。

公钥是由一对坐标+（x，y）组成的椭圆曲线上的一个点。它通常带有前缀+04，后跟两个256位数字：一个是该点的_x_坐标，另一个是_y_坐标。前缀+04+表示未压缩的公钥，+02+或+03+开头表示压缩的公钥。

这是我们先前创建的私钥生成的公钥，显示为坐标 x 和 y ：

```
x = F028892BAD7ED57D2FB57BF33081D5CFCF6F9ED3D3D7F159C2E2FFF579DC341A
y = 07CF33DA18BD734C600B96A72BBC4749D5141C90EC8AC328AE52DDFE2E505BDB
```

这是以520位数字（130十六进制数字）表示的公钥，结构为 04 x y ：

```
K = 04F028892BAD7ED57D2FB57BF33081D5CFCF6F9ED3D3D7F159C2E2FFF579DC341A↵
07CF33DA18BD734C600B96A72BBC4749D5141C90EC8AC328AE52DDFE2E505BDB
```

压缩的公钥

压缩公钥被引入到比特币中，以减少交易处理的大小并节省存储空间。大多数交易包括公钥，这是验证所有者凭证并花费比特币所需的。每个公钥需要520位（ 前缀 + x + y ），每个块有几百个交易，每天产生千上万的交易时，会将大量数据添加到区块链中。

正如我们在[公钥](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第四章.asciidoc#pubkey)看到的那样，公钥是椭圆曲线上的一个点（x，y）。因为曲线表达了一个数学函数，所以曲线上的一个点代表该方程的一个解，因此，如果我们知道_x_坐标，我们可以通过求解方程来计算_y_坐标 y2 mod p =（ x3 + 7 ）mod p。这允许我们只存储公钥的_x_坐标，省略_y_坐标并减少密钥的大小和所需的256位空间。在每次交易中，几乎减少了50％的尺寸，加起来可以节省大量的数据！

未压缩的公钥的前缀为+04+，压缩的公钥以+02+或+03+前缀开头。让我们看看为什么有两个可能的前缀：因为方程的左边是 *y*2，所以_y_的解是一个平方根，它可以具有正值或负值。从视觉上来说，这意味着生成的_y_坐标可以在x轴的上方或下方。从[An elliptic curve](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第四章.asciidoc#ecc-curve)中的椭圆曲线图可以看出，曲线是对称的，这意味着它在x轴上像镜子一样反射。因此，虽然我们可以省略_y_坐标，但我们必须存储_y_的_sign_（正数或负数）；换句话说，我们必须记住它高于或低于x轴，因为每个选项代表不同的点和不同的公钥。当在素数阶p的有限域上以二进制算法计算椭圆曲线时，_y_坐标是偶数或奇数，如前所述，它对应于正/负号。因此，为了区分_y_的两个可能值，我们存储一个压缩公钥，如果_y_是偶数，则前缀为+02+；如果是奇数，则存储前缀为+03+，从而允许软件从_x_坐标正确推导出_y_坐标，并将公钥解压为该点的完整坐标。[Public key compression](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第四章.asciidoc#pubkey_compression)中说明了公钥的压缩。

如果我们使用双散列函数（ RIPEMD160（SHA256（K） ）将此压缩公钥转换为比特币地址，它将生成一个_不同的_比特币地址。这可能会造成混淆，因为这意味着单个私钥可以产生以两种不同格式（压缩和未压缩）表示的公钥，这两种格式产生两个不同的比特币地址。但是，两个比特币地址的私钥是相同的。

压缩公钥正在逐渐成为比特币客户端的默认设置，这对减少交易和区块链的规模具有重大影响。但是，并非所有客户端都支持压缩的公钥。支持压缩公钥的较新客户端必须考虑来自不支持压缩公钥的较旧客户端的交易。当钱包应用从另一个比特币钱包应用导入私钥时，这一点尤其重要，因为新钱包需要扫描区块链以查找与这些导入的密钥相对应的交易。比特币钱包应该扫描哪些比特币地址？由未压缩的公钥生成的比特币地址，还是由压缩公钥生成的比特币地址？两者都是有效的比特币地址，并且可以用私钥签名，但它们是不同的地址！

术语“压缩私钥”是一种用词不当，因为当私钥以"WIF-compressed"的形式导出时，它实际上比“未压缩”的私钥多一个字节。这是因为私钥增加了一个字节的后缀（在[Example: Same key, different formats](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第四章.asciidoc#table_4-4)中以十六进制显示为01），表示私钥来自较新的钱包并且应该仅用于产生压缩的公钥。私钥本身并不压缩，也不能被压缩。术语“压缩私钥”实际上是指“只能从私钥导出压缩的公钥”，而“未压缩的私钥”实际上是指“只能从私钥导出未压缩的公钥”。

支付给脚本的哈希（Pay-to-Script Hash，P2SH）和多重签名地址

以数字“3”开头的比特币地址是支付给脚本的哈希（P2SH）地址，有时被错误地称为多重签名或多重地址。对P2SH地址进行编码使用与创建比特币地址时用到的相同的双重哈希函数，只是应用于脚本而不是公钥:

```
script hash = RIPEMD160(SHA256(script))
```

目前，P2SH功能最常见的实现是多重签名地址脚本。顾名思义，底层脚本需要多个签名才能证明所有权，才能花费资金。比特币多重签名特征被设计为需要来自总共N个密钥的M个签名（也称为“阈值”），称为M-N多重签名，其中M等于或小于N. 

## 五、钱包

用户使用密钥签署交易，从而证明他们拥有交易的输出（他们的比特币）。比特币以交易输出的形式（通常记作vout或txout）存储在区块链中。

根据包含的密钥是否彼此相关划分，主要有两种类型的钱包。

第一种是 *非确定性钱包* *nondeterministic wallet* ，其中每个密钥都是从随机数中独立生成的。密钥不相互关联。这种类型的钱包也被称JBOK钱包（Just a Bunch Of Keys）。

第二种是 *确定性钱包* *deterministic wallet*，其中所有密钥都来自单个主密钥，称为 *种子* *seed* 。这种钱包中的所有密钥都是相互关联的，如果有原始种子，可以再次生成。确定性钱包中使用了许多的 *密钥派生* *key derivation* 方法。 最常用的派生方法使用树状结构，并称为 *分层确定性* *hierarchical deterministic* 钱包或_HD_钱包。

非确定性（随机）钱包

在第一个比特币钱包（现在称为Bitcoin Core）中，钱包是随机生成的私钥集合。例如，Bitcoin Core客户端首次启动时生成100个随机私钥，并根据需要生成更多的密钥，每个密钥只使用一次。这些钱包正在被确定性的钱包取代，因为它们的管理，备份和导入很麻烦。随机密钥的缺点是，如果你生成了很多密钥，你必须保留所有密钥的副本，这意味着钱包必须经常备份。每个密钥都必须备份，否则，如果钱包变得不可用，则其控制的资金将不可撤销地丢失。这与避免地址重用的原则直接冲突，即每个比特币地址仅用于一次交易。地址重用将多个交易和地址相互关联来，会减少隐私。

分层确定性钱包（HD Wallets）(BIP-32/BIP-44)

HD钱包包含以树结构导出的密钥，父密钥可以导出一系列的子密钥，每个子密钥可以导出一系列孙子密钥等等，可达到无限深度。

与随机（非确定性）密钥相比，HD钱包具有两大优势。首先，树结构可以用来表达额外的组织含义，例如，使用子密钥的特定分支来接收传入的支付，使用另一个分支来接收支付时的零钱。分支的密钥也可用于组织机构设置，将不同分支分配给部门，子公司，特定功能或会计类别。

HD钱包的第二个优点是用户可以创建一系列公钥而无需访问相应的私钥。这允许HD钱包用于不安全的服务器或仅作为接收用途，为每次交易发出不同的公钥。公钥不需要事先预加载或派生，服务器也没有可以花费资金的私钥。

通用标准是：

- 助记词（mnemonic code words）, 基于BIP-39
- 分层确定性钱包（HD wallets）, 基于BIP-32
- 多用途分层确定性结构（Multipurpose HD wallet structure）, 基于BIP-43
- 多币种和多帐户钱包（Multicurrency and multiaccount wallets），基于BIP-44

生成助记词

1. 创建一个128到256位（128、160、192、224、256）的随机序列（熵）。
2. 通过取其SHA256散列的第一个（熵长度/ 32）位创建随机序列的校验和。
3. 将校验和添加到随机序列的末尾。
4. 将结果拆分为11位长的多个段。
5. 将每个11位值映射到有2048个单词的预定义字典中的一个单词。
6. 助记词就是这些单词的序列。

通过种子创建HD钱包

将根种子作为 HMAC-SHA512 算法的输入，生成的哈希结果用来生成 *主私钥* *master private key* (m) 和 *主链码* *master chain code* (c)。

![HDWalletFromRootSeed](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0509.png)

子密钥派生方法基于单向散列函数，该函数结合：

- 一个父级私钥或公钥 (ECDSA未压缩密钥)
- 一个称作链码(chain code)的种子（256 bits）
- 一个索引数字（32 bits）

链码用于向过程中引入确定性随机数据，所以只知道索引和子密钥不足以派生其他子密钥。除非有链码，否则知道一个子钥匙不能找到它的兄弟姐妹。初始链码种子（树的根部）由种子制成，而后续子链码则从每个父链码中导出。

使用HMAC-SHA512算法将父公钥，链码和索引组合并散列，以产生512位散列。这个512位散列平分为两部分。右半部分256位作为后代的链码，左半部分256位被添加到父私钥以生成子私钥。

![ChildPrivateDerivation](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0510.png)

子私钥与非确定性（随机）密钥没有区别。因为派生函数是单向函数，不能使用子项来寻找父项和寻找任何兄弟姐妹。不能通过第n个子项找到它的兄弟姐妹，如第 n-1 个子项或者第 n+1 个子项，或者任何这个序列上的子项。只能通过父密钥和链码派生所有的孩子。如果没有子链码，子密钥也不能派生任何孙项。你需要子私钥和子链码来启动一个新分支并派生孙项。

扩展密钥简单地表示为由256位的密钥和256位的链码串联成的512位序列。有两种类型的扩展密钥：扩展私钥是私钥和链码的组合，可用于派生子私钥（从它们产生子公钥）；扩展公钥是公钥和链码，可用于创建子公钥（ *只有子公钥* ）

将扩展密钥视为HD钱包树形结构中分支的根。可以通过分支的根，派生出其他分支。扩展私钥可以创建一个完整的分支，而扩展公钥只能创建一个公钥分支。

扩展密钥使用Base58Check编码，可以轻松导出导入BIP-32兼容的钱包。扩展密钥的Base58Check编码使用特殊的版本号，当使用Base58字符进行编码时，其前缀为“xprv”和“xpub”，以使其易于识别。因为扩展的密钥是512或513位，所以它比我们以前见过的其他Base58Check编码的字符串要长得多。

> Gabriel 的 HD 钱包通过在不知道私钥的情况下派生子公钥的能力提供了更好的解决方案。Gabriel 可以在他的网站上加载一个扩展公钥（xpub），用来为每个客户订单派生一个唯一的地址。Gabriel 可以从他的Trezor花费资金，但在网站上加载的 xpub 只能生成地址并获得资金。HD钱包的这个特点是一个很好的安全功能。Gabriel 的网站不包含任何私钥，因此不需要高度的安全性。

强化的子密钥派生

从 xpub 派生公钥的分支是非常有用的，但有潜在的风险。访问 xpub 不会访问子私钥。但是，因为 xpub 包含链码，所以如果某个子私钥已知，或者以某种方式泄漏，则可以与链式代码一起使用，派生所有其他子私钥。一个泄露的子私钥和一个父链码可以生成所有其他的子私钥。更糟的是，可以使用子私钥和父链码来推导父私钥。

为了应对这种风险，HD钱包使用一种称为 *hardened derivation* 的替代派生函数，该函数“破坏”父公钥和子链码之间的关系。强化派生函数使用父私钥来派生子链码，而不是父公钥。这会在父/子序列中创建一个“防火墙”，链码不能危害父级或同级的私钥。

![ChildHardPrivateDerivation](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0513.png)

当使用强化的私有派生函数时，生成的子私钥和链码与正常派生函数所产生的完全不同。由此产生的“分支”密钥可用于生成不易受攻击的扩展公钥，因为它们所包含的链码不能用于揭示任何私钥。因此，强化派生用于在继承树上使用扩展公钥的级别之上创建“屏障”。

简而言之，如果你想使用 xpub 的便利性来派生分支公钥，而不想面临泄漏链码的风险，应该从强化的父项派生。作为最佳实践，主密钥的1级子密钥始终使用强化派生，以防止主密钥受到破坏。

在派生函数中使用的索引号是一个32位整数。为了便于区分通过常规推导函数派生的密钥与通过强化派生派生的密钥，该索引号分为两个范围。 0到2^31 - 1（0x0到0x7FFFFFFF）之间的索引号仅用于常规推导。 2^31 和 2^32 - 1（0x80000000到0xFFFFFFFF）之间的索引号仅用于硬化派生。因此，如果索引号小于2^31，则子密钥是常规的，而如果索引号等于或大于 2^31，则子密钥是强化派生的。

为了使索引号码更容易阅读和显示，强化子密钥的索引号从零开始显示，但带有一个符号。第一个常规子密钥表示成0，第一个强化子秘钥（ 索引号是 0x80000000 ）表示成0'。以此类推，第二个强化子密钥（ 0x80000001 ) 表示成1'。当你看到HD钱包索引i’时，它表示2^31+i.

HD钱包中的密钥使用“路径(path)”命名约定来标识，树的每个级别都用斜杠（/）字符分隔（请参见 [HD wallet path examples](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第五章.asciidoc#table_4-8)）。从主密钥派生的私钥以“m”开头。从主公钥派生的公钥以“M”开始。因此，主私钥的第一个子私钥为 m/0。第一个子公钥是 M/0。第一个子私钥的第二个子私钥是 m/0/1，依此类推。

从右向左读取一个密钥的“祖先”，直到到达派生出它的主密钥。例如，标识符 m/x/y/z 描述了私钥 m/x/y 的第z个子私钥，m/x/y 是私钥 m/x 的第y个子私钥，m/x 是 m 的第x个子私钥。

BIP-44定义了包含五个预定义树级的结构：

```
m / purpose' / coin_type' / account' / change / address_index
```

## 六、交易

比特币交易的基本构建块是 *交易的输出* *transaction output* 。交易输出是不可分割的比特币货币，记录在区块链中，被整个网络识别为有效的。比特币完整节点跟踪所有可用和可花费的输出，称为 *未花费的交易输出* *unspent transaction outputs* 或 *UTXO* 。所有UTXO的集合被称为 *UTXO set* ，目前有数以百万的UTXO。UTXO集的大小随着新UTXO的增加而增长，并在UTXO被消耗时缩小。每个交易都表示UTXO集中的更改（状态转移）。

当我们说用户的钱包“收到”比特币时，意思是钱包检测到一个可以使用该钱包控制的密钥来花费的UTXO。因此，用户的比特币“余额”是用户钱包可以花费的所有UTXO的总和，可以分散在数百个交易和数百个块中。余额的概念是由钱包应用创建的。钱包扫描区块链并将钱包可以使用它的密钥花费的任何UTXO汇总计算用户的余额。大多数钱包维护数据库或使用数据库服务来存储它们可以花费的所有UTXO的快照。

输出是 *不连续的* 和 *不可分割的* 的价值，以整数satoshis为单位。未使用的输出只能由交易全部花费。

如果UTXO大于交易的期望值，它仍然必须全部使用，并且必须在交易中生成零钱。换句话说，如果你有一个价值20比特币的UTXO，并且只需要支付1比特币，那么你的交易必须消费整个20比特币的UTXO，并产生两个输出：一个支付1比特币给你想要的收款人，另一个支付19比特币回到你的钱包。由于交易输出的不可分割性，大多数比特币交易将不得不产生零钱。

用户的钱包应用通常会从用户的可用UTXO中进行选择，使组合的金额大于或等于期望交易金额。

输出和输入链的例外是称为 *币基* *coinbase* 交易的特殊类型的交易，它是每个块中的第一个交易。这笔交易由“获胜”的矿工设置，创建全新的比特币并支付给该矿工作为挖矿奖励。此特殊的coinbase交易不消费UTXO，相反，它有一种称为“coinbase”的特殊输入类型。

序列化是将数据结构的内部表示转换为可以一次传输一个字节的格式（也称为字节流）的过程。序列化最常用于对通过网络传输或存储在文件中的数据结构进行编码。

大多数交易包括交易费用，以奖励比特币矿工，保证网络安全。费用本身也可以作为一种安全机制，因为攻击者通过大量交易充斥网络在经济上是不可行的。

随着时间的推移，交易费用的计算方式以及它们对交易优先级的影响已经发生了变化。起初，交易费用在整个网络中是固定不变的。逐渐地，收费结构放松，并可能受到基于网络容量和交易量的市场力量的影响。至少从2016年初开始，比特币的容量限制已经造成了交易之间的竞争，导致了更高的费用，使免费的交易成为了历史。免费或低费用的交易很少能被开采，有时甚至不会通过网络传播。

交易费用是以交易数据的大小（KB）计算的，而不是比特币交易的价值。

交易的数据结构没有费用字段。相反，费用隐含表示为输入总和与输出总和的差额。从所有输入中扣除所有输出后剩余的金额都是矿工收取的费用。

图灵不完备

比特币交易脚本语言包含许多操作符，但是故意在一个重要方面进行了限制 - 除了条件控制外，没有循环或复杂的流程控制功能。这确保语言不是 *图灵完备* *Turing Complete* 的，这意味着脚本具有有限的复杂性和可预测的执行时间。脚本不是通用语言。这些限制确保了该语言不能用于创建无限循环或其他形式的“逻辑炸弹”，这种“逻辑炸弹”可能嵌入交易中，导致对比特币网络的拒绝服务攻击。

无状态验证

比特币交易脚本语言是无状态的，在执行脚本之前没有状态，在执行脚本之后也不保存状态。因此，执行脚本所需的所有信息都包含在脚本中。脚本在任何系统上都能可预测地执行。

比特币的脚本语言称为基于堆栈的语言，因为它使用称为 *栈* *stack* 的数据结构。脚本语言通过从左向右处理每个项目来执行脚本。"数字"（数据常量）被push进入堆栈。"操作"从堆栈中pop一个或多个参数，执行操作，并可能将结果push到堆栈。

支付到公钥哈希 Pay-to-Public-Key-Hash (P2PKH)

在比特币网络上处理的绝大多数交易花费由支付到公钥哈希（P2PKH）锁定的输出这些输出包含一个锁定脚本。这些输出包含将它们锁定到公钥哈希（比特币地址）的脚本。由P2PKH脚本锁定的输出可以通过出示公钥，和由相应私钥创建的数字签名来解锁（花费）

## 七、高级交易和脚本

多重签名脚本设置了一个条件，N 个公钥记录在脚本中，并且需要其中至少 M 个提供签名才能解锁资金。这也被称为 M-of-N 方案，其中 N 是密钥的总数，M 是验证所需签名个数的阈值。

M-of-N 多重签名条件的锁定脚本设置通常形式如下：

```
M <Public Key 1> <Public Key 2> ... <Public Key N> N CHECKMULTISIG
```

其中 N 是列出的公钥数量，M 是花费这笔支出所需的签名个数。

两个脚本组合起来形成下面的验证脚本

```
0 <Signature B> <Signature C> 2 <Public Key A> <Public Key B> <Public Key C> 3 CHECKMULTISIG
```

执行时，只有在解锁脚本与锁定脚本设置的条件匹配时，此组合脚本才会评估为TRUE。

支付到脚本哈希 Pay-to-Script-Hash (P2SH)

Table 1. Complex script without P2SH

| Locking Script   | 2 PubKey1 PubKey2 PubKey3 PubKey4 PubKey5 5 CHECKMULTISIG |
| ---------------- | --------------------------------------------------------- |
| Unlocking Script | Sig1 Sig2                                                 |

Table 2. Complex script as P2SH

| Redeem Script    | 2 PubKey1 PubKey2 PubKey3 PubKey4 PubKey5 5 CHECKMULTISIG |
| ---------------- | --------------------------------------------------------- |
| Locking Script   | HASH160 <20-byte hash of redeem script> EQUAL             |
| Unlocking Script | Sig1 Sig2 <redeem script>                                 |

P2SH功能的另一个重要部分是将脚本哈希编码为地址的能力，如BIP-13中所定义的那样。P2SH地址是脚本的20字节散列的Base58Check编码，就像比特币地址是公钥的20字节散列的Base58Check编码一样。 P2SH地址使用版本前缀“5”，这导致以“3”开头的Base58Check编码地址。

现在，几乎可以使用任何比特币钱包进行简单付款，就像它是一个比特币地址一样。前缀3给他们一个暗示，这是一种特殊的地址类型，对应于脚本而不是公钥，但是它的作用方式与支付比特币地址的方式完全相同。

P2SH具有以下优点：

- 复杂的脚本在交易输出中被更短的指纹代替，从而使交易数据更小。
- 脚本可以编码为地址，发件人和发件人的钱包不需要复杂的工程来实现P2SH。
- P2SH将构建脚本的负担转移给收件人，而不是发件人。
- P2SH将长脚本的数据存储负担从输出中（存储在区块链中的UTXO集中）转移到输入中（仅存储在区块链中）。
- P2SH将长文件的数据存储负担从当前时间（支付）转移到未来时间（花费时间）。
- P2SH将长脚本的交易费用从发件人转移到收件人，收件人必须包含很长的兑换脚本才能使用。

数据记录输出 (RETURN)

在Bitcoin Core客户端的0.9版本中，通过引入 RETURN 运算符达成了一个折衷方案。 RETURN 允许开发人员将80个字节的非付款数据添加到交易输出中。但是，与使用“假”UTXO不同，RETURN 运算符会创建一个显式的 *可验证不可消费* 的输出，该输出不需要存储在UTXO集合中。 RETURN 输出记录在区块链中，因此它们消耗磁盘空间并会导致区块链大小的增加，但它们不存储在UTXO集中，因此不会使UTXO内存池膨胀，完整节点页不用承担昂贵的内存负担。

时间锁 Timelocks
时间锁是对交易或输出的限制，只允许在某个时间点之后花费。

交易时间锁 (nLocktime)

交易锁定时间是交易级别的设置（交易数据结构中的一个字段），用于定义交易有效的最早时间，并且可以在网络上中转或添加到区块链。

如果 nLocktime 非零且低于5亿，会被解释为为区块高度，表示交易无效并且不会在指定块高度之前中转或包含在区块链中。如果它超过5亿，它会被解释为Unix纪元时间戳（自1970年1月1日以来的秒数），表示交易在指定时间之前无效。使用 nLocktime 指定未来区块或时间的交易必须由发起的系统持有，只有在它们生效后才传输到比特币网络。如果交易在指定的 nLocktime 之前传输到网络，交易将被第一个节点认为无效并拒绝，不会被中转到其他节点。 nLocktime 的使用等同于推迟日期的纸质支票。

交易锁定时间限制

Alice签署了一笔交易，将其的一个输出指定到Bob的地址，并将 nLocktime 设置为3个月之后。Alice将该交易发送给了Bob。通过这次交易，Alice和Bob知道：

- 在3个月过去之前，Bob不能发起赎回资金的交易。
- Bob可能会在3个月后发起交易。

但是:

- Alice可以创建另一个交易，在没有锁定时间的情况下重复使用相同的输入。因此，Alice可以在3个月过去之前花费相同的UTXO。
- Bob无法保证Alice不这么做。

唯一的保证是鲍勃在3个月之前不能赎回，而无法保证鲍勃将获得资金。要达到这样的保证，时间限制必须放在UTXO上，并成为锁定脚本的一部分，而不是交易的一部分。

Check Lock Time Verify (CLTV)

CLTV 是每个输出的时间锁，而不是 使用 nLocktime 情况下的每个交易的时间锁。

nLocktime 是交易级别的时间锁，CLTV 是基于输出的时间锁。

为了用 CLTV 锁定输出，可以在创建这笔输出的交易中，将其插入到输出的赎回脚本中。例如，如果Alice正在向Bob的地址支付，输出通常会包含如下所示的P2PKH脚本：

```
DUP HASH160 <Bob's Public Key Hash> EQUALVERIFY CHECKSIG
```

为了将其锁定一段时间，比如从现在开始3个月，这笔交易将带有如下的赎回脚本：

```
<now + 3 months> CHECKLOCKTIMEVERIFY DROP DUP HASH160 <Bob's Public Key Hash> EQUALVERIFY CHECKSIG
```

当Bob尝试花费这个UTXO时，构建一个以UTXO作为输入的交易，在输入的解锁脚本中使用他的签名和公钥，并将交易的 nLocktime 设置为等于或大于 CHECKLOCKTIMEVERIFY 中Alice设置的 timelock，然后在比特币网络上广播交易。

Bob的交易被进行如下的评估，如果Alice设置的 CHECKLOCKTIMEVERIFY 的参数小于或等于消费交易的 nLocktime，则脚本执行继续（如同执行 "no operation" 或NOP操作码一样）。否则，脚本执行会停止，并且交易被视为无效。

CLTV 和 nLocktime 使用相同的格式来描述时间锁，可以是区块高度，也可以是自Unix纪元以来的秒数。重要的是，当一起使用时，nLocktime 的格式必须与输出中的 CLTV 的格式匹配 —— 它们都必须表示区块高度，或以秒为单位的时间。

相对时间锁

允许两个或多个相互依赖的交易组成的交易链进行脱链处理，对一个依赖于前一个交易确认后一段时间的交易施加时间限制。换句话说，直到UTXO被记录在区块链上时，时钟才会开始计数。这个功能在双向状态通道（bidirectional state channels）和闪电网络（Lightning Networks）中特别有用

相对时间锁与绝对时间锁一样，都是通过交易级功能和脚本级操作码实现的。交易级别的相对时间锁实现为 nSequence（每个交易输入中设置的字段）值的共识规则。脚本级别的相对时间锁使用 CHECKSEQUENCEVERIFY（CSV）操作码实现。

自BIP-68启用以来，新的共识规则适用于包含 nSequence 值小于231的输入的任何交易。从编程的角度来说，这意味着如果最高有效位（第1<<31位）未设置为1，则表示“相对锁定时间”。否则（1<<31设置为1），nSequence 的值被保留用于其他用途，例如启用 CHECKLOCKTIMEVERIFY，nLocktime，Opt-In-Replace-By-Fee以及其他未来的开发。

nSequence 值以块或秒为单位，但与我们在 nLocktime 中使用的格式略有不同。类型标志（type-flag）用于区分表示区块数还是表示时间（以秒为单位）。类型标志被设置在第23个最低有效位（即值1 << 22）中。如果类型标志为1，则 nSequence 值被解释为512秒的倍数。如果类型标志为0，则 nSequence 值将被解释为区块数。

当将 nSequence 解释为相对时间锁时，仅考虑16个最低有效位。一旦标志位（比特32和23）检测完成，通常使用 nSequence 的16位掩码（例如，nSequence ＆ 0x0000FFFF ）。

![BIP-68 definition of nSequence encoding](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0701.png)

在UTXO的赎回脚本中执行时，CSV 操作码仅允许输入的 nSequence 值大于或等于 CSV 参数的交易。从本质上讲，这限制了UTXO直到相对于UTXO开采的时间已经过去了一定数量的区块或秒之后才能被花费。

隔离见证 Segregated Witness (segwit)

当交易花费UTXO时，它必须提供见证。在传统的UTXO中，锁定脚本要求在花费UTXO的交易的输入部分提供 *在线的* 见证数据。然而，隔离见证UTXO指定了一个锁定脚本，它可以被输入之外的（隔离的）见证数据满足。

Pay-to-Witness-Public-Key-Hash (P2WPKH)

Alice创建了一笔交易，向Bob购买一杯咖啡。该交易创建了一个值为0.015 BTC的P2PKH输出，该输出可由Bob使用。输出的脚本如下所示：

Example P2PKH output script

```
DUP HASH160 ab68025513c3dbd2f7b92a94e0581f5d50f654e7 EQUALVERIFY CHECKSIG
```

使用隔离见证，Alice将创建一个 Pay-to-Witness-Public-Key-Hash (P2WPKH) 脚本, 看起来如下:

Example P2WPKH output script

```
0 ab68025513c3dbd2f7b92a94e0581f5d50f654e7
```

第一个数字（0）被解释为版本号（*witness version*），第二部分（20字节）相当于被称为 *witness program* 的锁定脚本。20字节的见证程序就是公钥的散列，就像在P2PKH脚本中一样。

对于原始脚本（nonsegwit），Bob的交易必须在交易输入中包含签名

Decoded transaction showing a P2PKH output being spent with a signature

```
[...]
“Vin” : [
"txid": "0627052b6f28912f2703066a912ea577f2ce4da4caa5a5fbd8a57286c345c2f2",
"vout": 0,
     	 "scriptSig": “<Bob’s scriptSig>”,
]
[...]
```

但是，要花费隔离见证的输出，交易在那个输入上没有签名。相反，Bob的交易包含一个空的 scriptSig 和一个在交易之外的隔离见证。

Decoded transaction showing a P2WPKH output being spent with separate witness data

```
[...]
“Vin” : [
"txid": "0627052b6f28912f2703066a912ea577f2ce4da4caa5a5fbd8a57286c345c2f2",
"vout": 0,
     	 "scriptSig": “”,
]
[...]
“witness”: “<Bob’s witness data>”
[...]
```

Pay-to-Witness-Script-Hash (P2WSH)

Mohammed的公司使用P2SH来表示多重签名脚本。对Mohammed的公司的付款用这种锁定脚本编码：

Example P2SH output script

```
HASH160 54c557e07dde5bb6cb791c7a540e0a4796f5e97e EQUAL
```

该P2SH脚本引用了 *赎回脚本* *redeem_script* 的散列，该脚本定义了花费资金的 2-of-3 多重签名要求。为了使用这种输出，Mohammed的公司将在交易输入中提供赎回脚本（其哈希与P2SH输出中的脚本哈希匹配）以及满足赎回脚本所需的签名：

Decoded transaction showing a P2SH output being spent

```
[...]
“Vin” : [
"txid": "abcdef12345...",
"vout": 0,
     	 "scriptSig": “<SigA> <SigB> <2 PubA PubB PubC PubD PubE 5 CHECKMULTISIG>”,
]
```

现在，让我们看看整个示例如何升级到segwit。如果Mohammed的客户使用兼容segwit的钱包，他们将创建一个付款，包含一个Pay-to-Witness-Script-Hash（P2WSH）输出，看起来像这样：

Example P2WSH output script

```
0 a9b7b38d972cabc7961dbfbcb841ad4508d133c47ba87457b4a0e8aae86dbb89
```

同样，与P2WPKH的例子一样，你可以看到隔离见证等效脚本更简单，并且省略了你在P2SH脚本中看到的各种脚本操作数。相反，隔离见证程序由推送到堆栈的两个值组成：见证版本（0）和赎回脚本的32字节SHA256散列。

> 虽然P2SH使用20字节 RIPEMD160( SHA256(script) ) 散列，P2WSH见证程序使用32字节 +SHA256(script)+散列。这种散列算法选择上的差异是故意的，用于区分两种类型的见证程序（P2WPKH和P2WSH）之间的哈希长度，并为P2WSH提供更强的安全性（P2WSH中的128位安全性，对比P2SH中的80位安全性）。

Decoded transaction showing a P2WSH output being spent with separate witness data

```
[...]
“Vin” : [
"txid": "abcdef12345...",
"vout": 0,
     	 "scriptSig": “”,
]
[...]
“witness”: “<SigA> <SigB> <2 PubA PubB PubC PubD PubE 5 CHECKMULTISIG>”
[...]
```

P2WPKH 和 P2WSH 的区别

两种见证程序都由一个单字节版本号和一个较长的散列组成。它们看起来非常相似，但是却有着不同的解释：一个被解释为一个公钥的哈希，它被签名满足，另一个被解释为脚本的哈希，被一个赎回脚本满足。它们之间的关键区别在于哈希的长度：

- P2WPKH 中公钥的哈希是 20 字节
- P2WSH 中脚本的哈希是 32 字节

这是允许钱包区分两种见证程序的一个区别。通过查看散列的长度，钱包可以确定它是什么类型的见证程序，P2WPKH或P2WSH。

两种形式的见证脚本，P2WPKH 和 P2WSH，都可以嵌入到P2SH地址中。第一个被记作P2SH（P2WPKH），第二个被记作P2SH（P2WSH）。

## 八、比特币网络

简单支付验证（simplified payment_verification, SPV）

SPV节点仅下载区块头，而不下载每个块中包含的交易。由此产生的区块链，比完整区块链小1000倍。 SPV节点无法构建可用于支出的所有UTXO的完整画面，因为他们不知道网络上的所有交易。 SPV节点使用一种不同的方法验证交易，这种方法依赖对等节点按需提供区块链相关部分的部分视图。

SPV节点通过请求merkle路径证明，并验证区块链中的工作量证明，来建立交易存在于区块中的证明。SPV节点可以明确证明交易存在，但无法验证交易（例如同一个UTXO的双重花费）不存在，因为它没有所有交易的记录。此漏洞可用于拒绝服务攻击或针对SPV节点的双重支出攻击。为了防止这种情况发生，SPV节点需要随机地连接到多个节点，以增加与至少一个诚实节点接触的概率。这种随机连接的需要意味着SPV节点也容易遭受网络分区攻击或Sybil攻击，即它们连接到了假节点或假网络，并且无法访问诚实节点或真正的比特币网络。

由于SPV节点需要检索特定交易以选择性地验证它们，因此它们也会产生隐私风险。与收集每个区块内所有交易的完整区块链节点不同，SPV节点对特定数据的请求可能会无意中泄露其钱包中的地址。例如，监控网络的第三方可以跟踪SPV节点上的钱包所请求的所有交易，并使用它们将比特币地址与该钱包的用户相关联，从而破坏用户的隐私。

在引入SPV/轻量级节点后不久，比特币开发人员添加了一项名为 *布隆过滤器* *布隆_filters* 的功能，以解决SPV节点的隐私风险。布隆过滤器允许SPV节点通过使用概率而不是固定模式的过滤机制来接收交易子集，从而无需精确地揭示他们感兴趣的地址。

交易池

几乎比特币网络上的每个节点都维护一个名为 *memory pool*，*mempool_或_transaction pool* 的未确认交易的临时列表。节点使用该池来跟踪网络已知但尚未包含在区块链中的交易。例如，钱包节点将使用交易池来追踪已经在网络上接收但尚未确认的到用户钱包的传入支付。

交易被接收和验证后，会被添加到交易池并被中继到相邻节点以在网络上传播。

一些节点实现还维护一个单独的孤儿交易池。如果交易的投入引用尚未知晓的交易，好像遗失了父母，那么孤儿交易将临时存储在孤儿池中，直至父交易到达。

将交易添加到交易池时，将检查孤儿交易池是否有任何引用此交易输出的孤儿（后续交易）。然后验证任何匹配的孤儿。如果有效，它们将从孤儿交易池中删除并添加到交易池中，从而完成从父交易开始的链。鉴于不再是孤儿的新增交易，该过程重复递归地寻找更多后代，直到找不到更多的后代。通过这个过程，父交易的到来触发了整个链条相互依赖的交易的级联重建，将孤儿与他们的父母重新整合在一起。

## 九、区块链

区块的主要标识符是它的加密哈希值，这是一种数字指纹，通过SHA256算法将块头两次散列获得。得到的32字节哈希被称为 *block hash* ，更准确地说是 *block header hash*，因为只有区块头用于计算。

The structure of the block header

| Size     | Field               | Description                                                  |
| -------- | ------------------- | ------------------------------------------------------------ |
| 4 bytes  | Version             | A version number to track software/protocol upgrades         |
| 32 bytes | Previous Block Hash | A reference to the hash of the previous (parent) block in the chain |
| 32 bytes | Merkle Root         | A hash of the root of the merkle tree of this block’s transactions |
| 4 bytes  | Timestamp           | The approximate creation time of this block (seconds from Unix Epoch) |
| 4 bytes  | Difficulty Target   | The Proof-of-Work algorithm difficulty target for this block |
| 4 bytes  | Nonce               | A counter used for the Proof-of-Work algorithm               |

块高度也不是区块的数据结构的一部分；它不存储在区块中。

默克尔树 Merkle Trees

*默克尔树* *merkle tree*, 也叫做 *二叉哈希树* *binary hash tree*, 是一种用于有效汇总和验证大型数据集的完整性的数据结构。

比特币的merkle树中使用的加密哈希算法是将SHA256应用两次，也称为double-SHA256。

我们从四个交易开始，A，B，C和D，它们构成了merkle树的 *叶子* *leaves*，如 [Calculating the nodes in a merkle tree](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第九章.asciidoc#simple_merkle) 所示。交易不存储在merkle树中；相反，它们的数据被散列并且所得到的哈希值被存储在每个叶节点中，如 HA，HB，HC 和 HD：

```
HA = SHA256(SHA256(Transaction A))
```

然后将连续的叶节点对汇总到父节点中，方法是连接两个哈希值并对它们进行散列。例如，要构造父节点 HAB，将子节点的两个32字节哈希值连接起来，以创建一个64字节的字符串。然后对该字符串进行双重散列来产生父节点的哈希值：

```
HAB = SHA256(SHA256(HA + HB))
```

继续该过程，直到顶部只有一个节点，该节点被称为merkle根。

由于merkle树是二叉树，它需要偶数个叶节点。如果要汇总的交易数量为奇数，则最后一个交易的哈希值将被复制以创建偶数个叶节点，这称为 *平衡的树* *balanced tree*。

![merkle_tree](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0902.png)

为了证明一个块中包含一个特定的交易，一个节点只需要产生 log2~(N) 个32个字节的哈希值，构成一个认证 *path* 或 *merkle_path*，将特定的交易连接到树的根。

![merkle_tree_path](https://github.com/inoutcode/bitcoin_book_2nd/raw/master/images/mbc2_0905.png)

节点可以通过产生只有四个32字节哈希长（总共128字节）的merkle路径来证明交易K包含在该块中。该路径由四个哈希值组成（ 在 << merkle_tree_path>> 带蓝色背景的 ），HL，HIJ，HMNOP 和 HABCDEFGH。通过提供这四个哈希值作为验证路径，任何节点都可以通过计算四个额外的哈希值来证明 HK（底部黑色背景的）包含在Merkle根中：HKL，HIJKL，HIJKLMNOP 和merkle树根（在图中用虚线表示）。

Merkle 树和简单支付验证节点

例如，考虑一个SPV节点，它对付款到它的钱包中地址的交易感兴趣。SPV节点将在其与对等节点的连接上建立一个布隆过滤器（参见 [[bloom_filters\]](https://github.com/inoutcode/bitcoin_book_2nd/blob/master/第九章.asciidoc#bloom_filters) ），将接收到的交易限制为那些只包含其感兴趣地址的交易。当对等节点看到与bloom过滤器匹配的交易时，它将使用 merkleblock 消息发送该块。merkleblock 消息包含区块的头，以及将感兴趣的交易链接到区块中的merkle根的merkle路径。SPV节点可以使用此Merkle路径将交易连接到区块并验证交易是否包含在块中。 SPV节点还使用区块头将区块链接到区块链的其余部分。交易和区块之间以及区块和区块链之间的这两个链接的组合证明交易记录在区块链中。总而言之，SPV节点将接收到少于一千字节的数据块头和merkle路径，其数据量比完整块（当前大约1兆字节）少一千倍以上。

## 十、挖矿和共识

新铸造的硬币和交易费用奖励是一种激励计划，它将矿工的行为与网络的安全保持一致，同时实施货币供应。

挖矿的目的不是创造新的比特币。这是激励机制。挖矿是使比特币的 安全性 security 去中心化 _decentralized_的机制。

解决PoW算法赢得奖励以及在区块链上记录交易的权利的竞争是比特币安全模型的基础。

大约每四年（或正好每210,000块），一个矿工可以添加到区块的最大新增比特币数量减少一半。2009年1月开始每个区块50比特币，2012年11月每个区块减半到25比特币，2016年7月再次减少到12.5比特币。按照指数规律进行32次“减半”，直到6,720,000块（大约在2137年开采），达到最低货币单位，1 satoshi。大约2140年之后，将有690万个区块，发行近2,099,999,997,690,000个satoshis，即将近2100万比特币。

通货紧缩的货币

固定和递减的货币发行的最重要的和有争议的后果是，货币倾向于内在地 *通货紧缩*。通货紧缩是由于供求失衡导致货币价值（和汇率）升值的现象。与通货膨胀相反，价格通缩意味着这些资金随着时间的推移具有更多的购买力。

许多经济学家认为，通货紧缩的经济是一场应该不惜一切代价避免的灾难。这是因为在通货紧缩时期，人们倾向于囤积钱而不是花钱，希望价格会下跌。日本“失去的十年”期间，这种现象就显现出来了，当时需求的彻底崩溃将货币推向通缩螺旋。

比特币专家认为，通货紧缩本身并不糟糕。当然，通货紧缩与需求崩溃有关，因为这是我们必须研究的唯一的通缩的例子。在可以无限印刷的法定货币下，进入通货紧缩螺旋是非常困难的，除非需求完全崩溃并且政府不愿印钞。比特币的通货紧缩不是由需求崩溃引起的，而是由可预见的供应紧张造成的。

比特币的去中心化共识来自四个独立于网络节点的过程的相互作用：

- 每笔交易由完整节点独立验证，基于一份全面的标准清单
- 通过挖矿节点将交易独立地聚合到新的区块中，并通过PoW算法证明计算。
- 每个节点独立验证新的区块，并组装到区块链中
- 通过工作流程证明，每个节点独立选择具有最多累积计算量证明的链

币基交易

区块中的第一笔交易是一笔特殊的交易，叫做 *币基交易* *coinbase transaction*。开采一个区块所收取的奖励总额是币基奖励(25个新比特币)和该区块所有交易的费用（0.09094928）之和。与一般交易不同，币基交易不消耗（花费）UTXO作为输入。它只有一个输入，叫做 *coinbase* 币基，从无创造比特币。币基交易有一笔输出，可支付给矿工自己的比特币地址。

构建区块头

随着所有其他字段被填充，区块头现在已经完成，挖矿过程开始。目标是找到一个随机数的值，使区块头的哈希值小于 target。

每产生2016个区块，所有节点都重新设定PoW目标。重新设定目标的公式衡量了找到最后2016个区块所需的时间，并与预期的20160分钟(2016个区块乘以期望的10分钟区块间隔)进行了比较。计算实际时间间隔和期望时间间隔的比例，并对目标按比例进行调整(向上或向下)。简单地说：如果网络发现区块的速度比每10分钟快，难度就会增加(目标范围缩小)。如果区块发现速度比预期的要慢，那么难度就会降低(目标范围增加)。

Jing的挖矿节点立即将这个区块发送到它的对等节点。它们接收，验证，并传播这个新的区块。随着这个区块在网络上涟漪般传播，每个节点都将其添加到自己的区块链上，将区块链的高度扩展到 277,316 个区块。挖矿节点接收并验证区块，放弃自己尝试挖掘相同区块的努力，并立即开始计算链上的下一个区块，将Jing的区块作为“父块”。通过在Jing新发现的区块之上构建，其他的矿工实质上使用它们的算力“投票”，认可Jing的区块和它扩展的区块。

每个节点总是选择并尝试扩展表示最大工作量证明的块的链，也称为最长链或最大累计工作量链。

矿池

让我们回到骰子游戏的比喻。如果骰子玩家投掷骰子的目标是投掷出少于4（整体网络难度）的值，则游戏池将设定更容易的目标，计算游戏池玩家投掷结果小于8的次数。当选手掷出少于8（池目标）的值时，他们将获得份额，但是他们没有赢得比赛，因为他们没有达到比赛目标（少于4）。池玩家可以更频繁地获得更容易的池目标，即使他们没有实现赢得比赛的更难的目标，也可以非常经常地赢得他们的份额。无论何时，其中一名球员将掷出少于4的值，并且赢得比赛。然后，收入可以根据他们获得的份额分配给池玩家。尽管8或更少的目标没有获胜，但这是测量玩家掷骰子的公平方法，偶尔会产生少于4的投掷。

对等矿池 (P2Pool)

一个没有中央运营商的对等矿池。

P2Pool通过分散池服务器的功能工作，实现了称为 *股份链* *share_chain* 的并行的类似区块链的系统。股份链是比比特币区块链难度更低的区块链。股份链允许矿池矿工通过以每30秒一个块的速度挖掘链上的份额，在去中心化的池中进行合作。股份链上的每个区块都会为参与工作的池矿工记录相应的股份回报，并将股份从前一个股份块中向前移动。当其中一个股份区块也实现比特币网络目标时，它会被传播并包含在比特币区块链中，奖励所有为获胜股份区块之前的所有股份作出贡献的矿池矿工。本质上，与池服务器跟踪池矿工的股份和奖励不同，股份链允许所有池矿工使用类似比特币的区块链共识机制的去中心化共识机制来跟踪所有股份。

共识攻击

重要的是要注意到，共识攻击只能影响未来的共识，或者至多影响最近的过去(几十个块)。随着时间的流逝，比特币的账本变得越来越不可改变。虽然在理论上，分叉可以在任何深度上实现，但在实践中，强制执行一个非常深的分叉所需的计算能力是巨大的，这使得旧的块实际上是不可变的。

## 十二、区块链应用

当比特币系统长期稳定运行时，它就提供了一定的保证，可以作为开发模块来创建应用程序。 这些包括：

杜绝双重支付

比特币去中心化共识算法的最根本保证是确保同一UTXO不会被花费两次。

不可篡改性

一旦交易被记录在区块中，并且随后的区块中添加了足够的工作量，该交易数据就变得不可篡改。不可篡改性是由能源进行保证的，因为重写区块链需要花费能源才能产生工作量证明。随着在包含交易的区块之后被提交的工作量增加，所需的能源以及由此带来的不篡改的程度也在增加。

中立

去中心化的比特币网络传播有效的交易，而不管这些交易的来源或内容如何。这意味着任何人都可以支付足够的费用来创建有效的交易，并相信别人会随时传输该交易并将其包含在区块链中。

安全时间戳

共识规则拒绝任何时间戳距离现在太远的区块，包括过去或将来。这可以确保区块上的时间戳是可信的。区块上的时间戳意味着一种保证，保证交易包含的全部输入之前都是未花费的。

哈希时间锁合约 Hash Time Lock Contracts (HTLC)

To create an HTLC, the intended recipient of the payment will first create a secret R. They then calculate the hash of this secret H:

```
H = Hash(R)
```

The script implementing an HTLC might look like this:

```
IF
    # Payment if you have the secret R
    HASH160 <H> EQUALVERIFY
ELSE
    # Refund after timeout.
    <locktime> CHECKLOCKTIMEVERIFY DROP
    <Payer Public Key> CHECKSIG
ENDIF
```

Anyone who knows the secret R, which when hashed equals to H, can redeem this output by exercising the first clause of the IF flow.

If the secret is not revealed and the HTLC claimed, after a certain number of blocks the payer can claim a refund using the second clause in the IF flow.

路由支付通道（闪电网络）

The Lightning Network is a proposed routed network of bidirectional payment channels connected end-to-end. A network like this can allow any participant to route a payment from channel to channel without trusting any of the intermediaries.

基本闪电网络示例

![Step-by-step payment routing through a Lightning Network](https://github.com/bitcoinbook/bitcoinbook/raw/develop/images/mbc2_1207.png)

闪电网络传输和路由

每当一个节点希望将支付发送给另一个节点时，它必须首先通过连接具有足够容量的支付通道来通过网络构建 *路径* *path*。节点公布路由信息，包括他们已经打开了哪些通道，每个通道有多少容量，以及他们收取的路由支付费用。路由信息可以以各种方式共享，随着闪电网络技术的发展，可能会出现不同的路由协议。

闪电网络根据称为 [Sphinx](http://bit.ly/2q6ZDrP) 的方案实施洋葱路由（onion-routed）协议。此路由协议可确保付款发起人可以通过 Lightning Network 构建和传递路径，以便：

- 中间节点可以验证和解密路由信息中属于他们的部分并找到下一跳。
- 除了上一跳和下一跳之外，他们无法了解路径中的任何其他节点。
- 他们无法识别付款路径的长度，或他们在该路径中的位置。
- 路径的每个部分都被加密，使得网络层的攻击者无法将来自路径不同部分的数据包相互关联。
- 与Tor（互联网上的洋葱路由匿名协议）不同，没有可以置于监控之下的“出口节点”。付款不需要传送到比特币区块链；节点只是更新通道余额。

使用这种洋葱路由协议，Alice将路径中的每个元素都封装在一个加密层中，从结尾开始并向后工作。因此，Alice已经构建了这种加密的多层“洋葱”消息。她将此发送给Bob，他只能解密和解包外层。在里面，Bob发现一封给Carol的信，他可以转发给Carol，但不能自己破译。沿着路径，消息被转发，解密，转发等，一直到Eric。每个参与者只知道每跳中的前一个和下一个节点。

路径的每个元素都包含有关必须扩展到下一跳的HTLC信息，正在发送的金额，要包含的费用以及使HTLC过期的CLTV锁定时间（以区块为单位）。随着路由信息的传播，这些节点将HTLC承诺转发到下一跳。

此时，你可能想知道节点为何不知道路径的长度及其在该路径中的位置？毕竟，他们收到一条消息并将其转发到下一跳。根据它是否变短了，他们能够推断出路径大小和位置？为了防止这种情况，路径总是固定为20跳，并填充随机数据。每个节点都会看到下一跳和一个固定长度的加密消息来转发。只有最终收件人看到没有下一跳。对于其他人来说，总是还有20跳。

闪电网络的好处

闪电网络是次层路由技术。它可以应用于任何支持一些基本功能的区块链，例如多重签名交易，时间锁定和基本智能合约。

如果闪电网络位于比特币网络之上，那么比特币网络可以在不牺牲无中介无信任运转原则的情况下，大幅提升容量，隐私，粒度和速度：

隐私 Privacy
闪电网络支付比比特币区块链上的支付私有得多，因为它们不公开。虽然路线中的参与者可以看到通过其通道传播的付款，但他们不知道发件人或收件人。

可互换性 Fungibility
闪电网络使得在比特币上应用监视和黑名单变得更加困难，从而增加了货币的可互换性。

速度 Speed
使用Lightning Network的比特币交易以毫秒为单位进行结算，而不是以分钟为单位，因为在不提交交易给区块的情况下清算HTLC。

粒度 Granularity
闪电网络可以使支付至少与比特币“灰尘”限制一样小，可能甚至更小。一些提案允许subsatoshi（次聪）增量。

容量 Capacity
闪电网络将比特币系统的容量提高了几个数量级。闪电网络路由的每秒支付数量没有实际的上限，因为它仅取决于每个节点的容量和速度。

无信任运作 Trustless Operation
闪电网络在节点之间使用比特币交易，节点之间作为对等运作而无需信任。因此，闪电网络保留了比特币系统的原理，同时显着扩大了其运行参数。



附录 Bitcoin Whitepaper

https://github.com/tianmingyun/MasterBitcoin2CN/blob/master/appdx-bitcoinwhitepaper.md

[摘要]：本文提出了一种完全通过点对点技术实现的电子现金系统，它使得在线支付能够直接由一方发起并支付给另外一方，中间不需要通过任何的金融机构。虽然数字签名（Digital signatures）部分解决了这个问题，但是如果仍然需要第三方的支持才能防止双重支付（double-spending）的话，那么这种系统也就失去了存在的价值。我们(we)在此提出一种解决方案，使现金系统在点对点的环境下运行，并防止双重支付问题。该网络通过随机散列（hashing）对全部交易加上时间戳（timestamps），将它们合并入一个不断延伸的基于随机散列的工作量证明（proof-of-work）的链条作为交易记录，除非重新完成全部的工作量证明，形成的交易记录将不可更改。最长的链条不仅将作为被观察到的事件序列（sequence）的证明，而且被看做是来自CPU计算能力最大的池（pool）。只要大多数的CPU计算能力都没有打算合作起来对全网进行攻击，那么诚实的节点将会生成最长的、超过攻击者的链条。这个系统本身需要的基础设施非常少。信息尽最大努力在全网传播即可，节点(nodes)可以随时离开和重新加入网络，并将最长的工作量证明链条作为在该节点离线期间发生的交易的证明。

工作量证明机制还解决了在集体投票表决时，谁是大多数的问题。如果决定大多数的方式是基于IP地址的，一IP地址一票，那么如果有人拥有分配大量IP地址的权力，则该机制就被破坏了。而工作量证明机制的本质则是一CPU一票。“大多数”的决定表达为最长的链，因为最长的链包含了最大的工作量。如果大多数的CPU为诚实的节点控制，那么诚实的链条将以最快的速度延长，并超越其他的竞争链条。如果想要对业已出现的区块进行修改，攻击者必须重新完成该区块的工作量外加该区块之后所有区块的工作量，并最终赶上和超越诚实节点的工作量。

激励系统也有助于鼓励节点保持诚实。如果有一个贪婪的攻击者能够调集比所有诚实节点加起来还要多的CPU计算力，那么他就面临一个选择：要么将其用于诚实工作产生新的电子货币，或者将其用于进行二次支付攻击。那么他就会发现，按照规则行事、诚实工作是更有利可图的。因为该等规则使得他能够拥有更多的电子货币，而不是破坏这个系统使得其自身财富的有效性受损。

一个攻击者能做的，最多是更改他自己的交易信息，并试图拿回他刚刚付给别人的钱。 

节点通过自己的CPU计算力进行投票，表决他们对有效区块的确认，他们不断延长有效的区块链来表达自己的确认，并拒绝在无效的区块之后延长区块以表示拒绝。
