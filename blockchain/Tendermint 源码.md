# Tendermint 源码

https://txiner.top/post/

```js
//去掉代码行号
$('table.lntable').each(function(){$(this).find('td.lntd').first().remove();});
//将代码块移到table之外
$('table.lntable').each(function(){let that = $(this); let pre = that.find('pre');pre.insertAfter(that);that.remove();});
//添加语言标签
$('code').each(function(){$(this).attr('lang',$(this).attr('data-lang'))});
//调整小标题层级
$('h4').each(function(){$(this).replaceWith(this.outerHTML.replace('<h4','<h3').replace('</h4>','</h3>'))});
```

## 入口

Tendermint的使用流程不做过多介绍了，写自己的abci程序，启动tendermint core和abci连接，然后用户和core通信，abci app进行共识，结果传递给core，完成共识并记录。 所以，正常情况下，程序有两个main函数，一个是abci的启动函数，一个是core的启动函数。我们从core的开始看起。

### rootCmd

我们打开文件cmd/tendermint/main.go可以发现创建了一个rootCmd，下面为rootCmd添加了一系列支持的函数

```go
rootCmd.AddCommand(
		cmd.GenValidatorCmd,
		cmd.InitFilesCmd,
		cmd.ProbeUpnpCmd,
		cmd.LiteCmd,
		cmd.ReplayCmd,
		cmd.ReplayConsoleCmd,
		cmd.ResetAllCmd,
		cmd.ResetPrivValidatorCmd,
		cmd.ShowValidatorCmd,
		cmd.TestnetFilesCmd,
		cmd.ShowNodeIDCmd,
		cmd.GenNodeKeyCmd,
		cmd.VersionCmd)
```

实际上，每一个支持的功能都可以理解为一个模块，功能定义都在cmd/tendermint/commands文件夹中。 由于我们要分析tendermint的启动流程，因此就忽略例如`version`,`lite`之类的功能。

### cobra

如果我们对rootCmd进行溯源的话，会发现是cobra.Command的实例化变量。 cobra是一位大神的项目，主要实现的是便捷的命令行参数管理运行。 同样有名的是他的viper项目，主要实现的是对配置的处理。 这两个项目在Fabric中也有广泛的使用。

```go
var RootCmd = &cobra.Command{
	Use:   "tendermint",
	Short: "Tendermint Core (BFT Consensus) in Go",
	PersistentPreRunE: func(cmd *cobra.Command, args []string) (err error) {
		if cmd.Name() == VersionCmd.Name() {
			return nil
		}
		config, err = ParseConfig()
		if err != nil {
			return err
		}
		if config.LogFormat == cfg.LogFormatJSON {
			logger = log.NewTMJSONLogger(log.NewSyncWriter(os.Stdout))
		}
		logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel())
		if err != nil {
			return err
		}
		if viper.GetBool(cli.TraceFlag) {
			logger = log.NewTracingLogger(logger)
		}
		logger = logger.With("module", "main")
		return nil
	},
}
```

我们可以发现，RootCmd的根命令是tendermint,然后再执行其他程序之前，先执行下这个匿名函数。主要的流程就是获取下节点的配置（如果我们沿着ParseConfig看下去就会发现viper了），然后配置一系列生成记录日志的方式。具体的日志管理和我们要看的内容无关，我们以后再深入了解。

### init

使用过tendermint的同学都知道，第一步是`tendermint init`创建所需要的配置文件,让我们追随着`init`的脚步，解开tendermint的面纱吧。 根据函数名称，是`InitFilesCmd`变量执行初始化操作

```go
var InitFilesCmd = &cobra.Command{
	Use:   "init",
	Short: "Initialize Tendermint",
	RunE:  initFiles,
}
```

默认执行的函数是`initFiles`,进一步查找得到执行的逻辑是

```go
func initFilesWithConfig(config *cfg.Config) error {
	// private validator
	privValFile := config.PrivValidatorFile()
	var pv *privval.FilePV
	if cmn.FileExists(privValFile) {
		pv = privval.LoadFilePV(privValFile)
		logger.Info("Found private validator", "path", privValFile)
	} else {
		pv = privval.GenFilePV(privValFile)
		pv.Save()
		logger.Info("Generated private validator", "path", privValFile)
	}

	nodeKeyFile := config.NodeKeyFile()
	if cmn.FileExists(nodeKeyFile) {
		logger.Info("Found node key", "path", nodeKeyFile)
	} else {
		if _, err := p2p.LoadOrGenNodeKey(nodeKeyFile); err != nil {
			return err
		}
		logger.Info("Generated node key", "path", nodeKeyFile)
	}

	// genesis file
	genFile := config.GenesisFile()
	if cmn.FileExists(genFile) {
		logger.Info("Found genesis file", "path", genFile)
	} else {
		genDoc := types.GenesisDoc{
			ChainID:         fmt.Sprintf("test-chain-%v", cmn.RandStr(6)),
			GenesisTime:     tmtime.Now(),
			ConsensusParams: types.DefaultConsensusParams(),
		}
		genDoc.Validators = []types.GenesisValidator{{
			Address: pv.GetPubKey().Address(),
			PubKey:  pv.GetPubKey(),
			Power:   10,
		}}

		if err := genDoc.SaveAs(genFile); err != nil {
			return err
		}
		logger.Info("Generated genesis file", "path", genFile)
	}

	return nil
}
```

首先是检查有没有针对本validator的私钥文件，也就是`priv_validator.json`，然后获取本validator的配置。 然后是配置本节点的相关公共信息，生成对应的文件`node_key.json`，里面存储了相关的公钥信息。 接下来就是生成创始区块文件，首先是配置基本的创始区块信息，包括chainId，创建时间以及共识相关的参数。 我们都知道，PBFT是确定人数的共识，因此需要指定好validators，再本次启动的时候，没有输入其他的validator，因此只有本节点的信息。 到目前为止，节点初始化完成。

### node

我们回去看main函数，发现有这么两句

```go
nodeFunc := nm.DefaultNewNode
rootCmd.AddCommand(cmd.NewRunNodeCmd(nodeFunc))
```

首先是创建默认的节点，如果用户有其他的需求，可以写自己的节点函数。

```go
func DefaultNewNode(config *cfg.Config, logger log.Logger) (*Node, error) {
	// Generate node PrivKey
	nodeKey, err := p2p.LoadOrGenNodeKey(config.NodeKeyFile())
	if err != nil {
		return nil, err
	}
	return NewNode(config,
		privval.LoadOrGenFilePV(config.PrivValidatorFile()),
		nodeKey,
		proxy.DefaultClientCreator(config.ProxyApp, config.ABCI, config.DBDir()),
		DefaultGenesisDocProviderFunc(config),
		DefaultDBProvider,
		DefaultMetricsProvider(config.Instrumentation),
		logger,
	)
}
```

可以发现主要的功能都是在NewNode中实现的,我们把函数源码粘贴过来

```go
func NewNode(config *cfg.Config,
	privValidator types.PrivValidator,
	nodeKey *p2p.NodeKey,
	clientCreator proxy.ClientCreator,
	genesisDocProvider GenesisDocProvider,
	dbProvider DBProvider,
	metricsProvider MetricsProvider,
	logger log.Logger) (*Node, error) {

	// 获取数据库blostore.db的操作对象
	blockStoreDB, err := dbProvider(&DBContext{"blockstore", config})
	if err != nil {
		return nil, err
	}
	blockStore := bc.NewBlockStore(blockStoreDB)

	// 获取状态数据库state.db的操作对象
	stateDB, err := dbProvider(&DBContext{"state", config})
	if err != nil {
		return nil, err
	}

    // 读取最开始的系统配置状态
    // 因为有可能state数据库刚创建，里面还没有状态信息
	genDoc, err := loadGenesisDoc(stateDB)
	if err != nil {
		genDoc, err = genesisDocProvider()
		if err != nil {
			return nil, err
		}
		saveGenesisDoc(stateDB, genDoc)
	}

    // 如果数据库里面有状态，就读取系统状态，没有的话就使用创世区块的数据作为当前的状态
	state, err := sm.LoadStateFromDBOrGenesisDoc(stateDB, genDoc)
	if err != nil {
		return nil, err
	}

    // 需要和我们自己的abci程序进行交互，也叫做proxyApp(代理app)
    // 然后启动和我们abci的连接
	proxyApp := proxy.NewAppConns(clientCreator)
	proxyApp.SetLogger(logger.With("module", "proxy"))
	if err := proxyApp.Start(); err != nil {
		return nil, fmt.Errorf("Error starting proxy app connections: %v", err)
	}

	// 创建相关的握手过程，和我们的proxyApp程序通信
	consensusLogger := logger.With("module", "consensus")
	handshaker := cs.NewHandshaker(stateDB, state, blockStore, genDoc)
	handshaker.SetLogger(consensusLogger)
	if err := handshaker.Handshake(proxyApp); err != nil {
		return nil, fmt.Errorf("Error during handshake: %v", err)
	}

	// 由于区块状态已经改变，因此我们需要重新加载下区块状态
	state = sm.LoadState(stateDB)

    // 相关日志忽略
	...

    // 如果本验证节点开启了socket连接，就需要开启监听
	if config.PrivValidatorListenAddr != "" {
		privValidator, err = createAndStartPrivValidatorSocketClient(config.PrivValidatorListenAddr, logger)
		if err != nil {
			return nil, errors.Wrap(err, "Error with private validator socket client")
		}
	}

	// 是否开启快速同步
	fastSync := config.FastSync
	if state.Validators.Size() == 1 {
		addr, _ := state.Validators.GetByIndex(0)
		if bytes.Equal(privValidator.GetAddress(), addr) {
			fastSync = false
		}
	}

	...

    // 创建共识，P2P以及内存池
	csMetrics, p2pMetrics, memplMetrics, smMetrics := metricsProvider()

	// 创建内存池
	mempool := mempl.NewMempool(
		config.Mempool,
		proxyApp.Mempool(),
		state.LastBlockHeight,
		mempl.WithMetrics(memplMetrics),
		mempl.WithPreCheck(sm.TxPreCheck(state)),
		mempl.WithPostCheck(sm.TxPostCheck(state)),
	)
	mempoolLogger := logger.With("module", "mempool")
	mempool.SetLogger(mempoolLogger)
	if config.Mempool.WalEnabled() {
		mempool.InitWAL() // no need to have the mempool wal during tests
	}
	mempoolReactor := mempl.NewMempoolReactor(config.Mempool, mempool)
	mempoolReactor.SetLogger(mempoolLogger)

	if config.Consensus.WaitForTxs() {
		mempool.EnableTxsAvailable()
	}

	// 创建另一个数据库evidence.db
	evidenceDB, err := dbProvider(&DBContext{"evidence", config})
	if err != nil {
		return nil, err
	}
	evidenceLogger := logger.With("module", "evidence")
	evidenceStore := evidence.NewEvidenceStore(evidenceDB)
	evidencePool := evidence.NewEvidencePool(stateDB, evidenceStore)
	evidencePool.SetLogger(evidenceLogger)
	evidenceReactor := evidence.NewEvidenceReactor(evidencePool)
	evidenceReactor.SetLogger(evidenceLogger)

	blockExecLogger := logger.With("module", "state")
	// make block executor for consensus and blockchain reactors to execute blocks
	blockExec := sm.NewBlockExecutor(
		stateDB,
		blockExecLogger,
		proxyApp.Consensus(),
		mempool,
		evidencePool,
		sm.BlockExecutorWithMetrics(smMetrics),
	)

	// Make BlockchainReactor
	bcReactor := bc.NewBlockchainReactor(state.Copy(), blockExec, blockStore, fastSync)
	bcReactor.SetLogger(logger.With("module", "blockchain"))

	// Make ConsensusReactor
	consensusState := cs.NewConsensusState(
		config.Consensus,
		state.Copy(),
		blockExec,
		blockStore,
		mempool,
		evidencePool,
		cs.StateMetrics(csMetrics),
	)
	consensusState.SetLogger(consensusLogger)
	if privValidator != nil {
		consensusState.SetPrivValidator(privValidator)
	}
	consensusReactor := cs.NewConsensusReactor(consensusState, fastSync, cs.ReactorMetrics(csMetrics))
	consensusReactor.SetLogger(consensusLogger)

	eventBus := types.NewEventBus()
	eventBus.SetLogger(logger.With("module", "events"))

	// services which will be publishing and/or subscribing for messages (events)
	// consensusReactor will set it on consensusState and blockExecutor
	consensusReactor.SetEventBus(eventBus)

	// Transaction indexing
	var txIndexer txindex.TxIndexer
	switch config.TxIndex.Indexer {
	case "kv":
		store, err := dbProvider(&DBContext{"tx_index", config})
		if err != nil {
			return nil, err
		}
		if config.TxIndex.IndexTags != "" {
			txIndexer = kv.NewTxIndex(store, kv.IndexTags(splitAndTrimEmpty(config.TxIndex.IndexTags, ",", " ")))
		} else if config.TxIndex.IndexAllTags {
			txIndexer = kv.NewTxIndex(store, kv.IndexAllTags())
		} else {
			txIndexer = kv.NewTxIndex(store)
		}
	default:
		txIndexer = &null.TxIndex{}
	}

	indexerService := txindex.NewIndexerService(txIndexer, eventBus)
	indexerService.SetLogger(logger.With("module", "txindex"))

	p2pLogger := logger.With("module", "p2p")
	nodeInfo, err := makeNodeInfo(
		config,
		nodeKey.ID(),
		txIndexer,
		genDoc.ChainID,
		p2p.NewProtocolVersion(
			version.P2PProtocol, // global
			state.Version.Consensus.Block,
			state.Version.Consensus.App,
		),
	)
	if err != nil {
		return nil, err
	}

	// Setup Transport.
	var (
		mConnConfig = p2p.MConnConfig(config.P2P)
		transport   = p2p.NewMultiplexTransport(nodeInfo, *nodeKey, mConnConfig)
		connFilters = []p2p.ConnFilterFunc{}
		peerFilters = []p2p.PeerFilterFunc{}
	)

	if !config.P2P.AllowDuplicateIP {
		connFilters = append(connFilters, p2p.ConnDuplicateIPFilter())
	}

	// Filter peers by addr or pubkey with an ABCI query.
	// If the query return code is OK, add peer.
	if config.FilterPeers {
		connFilters = append(
			connFilters,
			// ABCI query for address filtering.
			func(_ p2p.ConnSet, c net.Conn, _ []net.IP) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/addr/%s", c.RemoteAddr().String()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)

		peerFilters = append(
			peerFilters,
			// ABCI query for ID filtering.
			func(_ p2p.IPeerSet, p p2p.Peer) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/id/%s", p.ID()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)
	}

	p2p.MultiplexTransportConnFilters(connFilters...)(transport)

	// Setup Switch.
	sw := p2p.NewSwitch(
		config.P2P,
		transport,
		p2p.WithMetrics(p2pMetrics),
		p2p.SwitchPeerFilters(peerFilters...),
	)
	sw.SetLogger(p2pLogger)
	sw.AddReactor("MEMPOOL", mempoolReactor)
	sw.AddReactor("BLOCKCHAIN", bcReactor)
	sw.AddReactor("CONSENSUS", consensusReactor)
	sw.AddReactor("EVIDENCE", evidenceReactor)
	sw.SetNodeInfo(nodeInfo)
	sw.SetNodeKey(nodeKey)

	p2pLogger.Info("P2P Node ID", "ID", nodeKey.ID(), "file", config.NodeKeyFile())

	// Optionally, start the pex reactor
	//
	// TODO:
	//
	// We need to set Seeds and PersistentPeers on the switch,
	// since it needs to be able to use these (and their DNS names)
	// even if the PEX is off. We can include the DNS name in the NetAddress,
	// but it would still be nice to have a clear list of the current "PersistentPeers"
	// somewhere that we can return with net_info.
	//
	// If PEX is on, it should handle dialing the seeds. Otherwise the switch does it.
	// Note we currently use the addrBook regardless at least for AddOurAddress
	addrBook := pex.NewAddrBook(config.P2P.AddrBookFile(), config.P2P.AddrBookStrict)

	// Add ourselves to addrbook to prevent dialing ourselves
	addrBook.AddOurAddress(nodeInfo.NetAddress())

	addrBook.SetLogger(p2pLogger.With("book", config.P2P.AddrBookFile()))
	if config.P2P.PexReactor {
		// TODO persistent peers ? so we can have their DNS addrs saved
		pexReactor := pex.NewPEXReactor(addrBook,
			&pex.PEXReactorConfig{
				Seeds:    splitAndTrimEmpty(config.P2P.Seeds, ",", " "),
				SeedMode: config.P2P.SeedMode,
			})
		pexReactor.SetLogger(logger.With("module", "pex"))
		sw.AddReactor("PEX", pexReactor)
	}

	sw.SetAddrBook(addrBook)

	// run the profile server
	profileHost := config.ProfListenAddress
	if profileHost != "" {
		go func() {
			logger.Error("Profile server", "err", http.ListenAndServe(profileHost, nil))
		}()
	}

	node := &Node{
		config:        config,
		genesisDoc:    genDoc,
		privValidator: privValidator,

		transport: transport,
		sw:        sw,
		addrBook:  addrBook,
		nodeInfo:  nodeInfo,
		nodeKey:   nodeKey,

		stateDB:          stateDB,
		blockStore:       blockStore,
		bcReactor:        bcReactor,
		mempoolReactor:   mempoolReactor,
		consensusState:   consensusState,
		consensusReactor: consensusReactor,
		evidencePool:     evidencePool,
		proxyApp:         proxyApp,
		txIndexer:        txIndexer,
		indexerService:   indexerService,
		eventBus:         eventBus,
	}
	node.BaseService = *cmn.NewBaseService(logger, "Node", node)
	return node, nil
}
```

由于后面创建的很多我们还没有看到，因此分析下创建node的主要流程

```
创建或者打开blockstore.db 最终创建bc实例和bcReactor  
创建或者打开state.db 最终更新state对象  
创建mempool实例 加载本地持久化的交易池 并创建mempoolReactor  
创建与APP之间的客户端 并重返所有保存的区块  
创建证据内存池和证据Reactor  
根据上面已经创建的对象开始创建consensus对象和Reactor  
开始创建p2p的Switch 将所有reactor加入  
开启tx_index服务  
返回node实例
```

后面还有内容，我们以后继续分析。

## service、reactor接口

首先我们来看service，tendermint中所有的服务都是实现了这个接口定义的函数。

```go
type Service interface {
	// 默认的启动函数
	Start() error
	OnStart() error

	// 停止运行函数
	Stop() error
	OnStop()

	// 重启服务
	Reset() error
	OnReset() error

	// 服务是否在运行
	IsRunning() bool

	// 返回一个通道，结束服务
	Quit() <-chan struct{}

	// 返回服务的标志
	String() string

	// 日志记录
	SetLogger(log.Logger)
}
```

可以很清楚的理解service定义的这几个接口，并不难，对service的基本的操作也就这几个函数。 我们再看一看reactor接口。 说到reactor，总会想到RxJava，或者spring中的reactor，也就是异步式编程。

```go
type Reactor interface {
	cmn.Service // 默认继承了service基本接口

	// 添加一个路由
	SetSwitch(*Switch)

	// 获取channel
	GetChannels() []*conn.ChannelDescriptor

	// 添加以恶搞peer
	AddPeer(peer Peer)

	// 移除一个peer
	RemovePeer(peer Peer, reason interface{})

	// 处理receive函数
	Receive(chID byte, peer Peer, msgBytes []byte)
}
```

这个reactor就是和整个P2P网络进行交互的组件模块。

## peer

我们先看一下peer的定义

```go
type peer struct {
	cmn.BaseService

	peerConn
	mconn *tmconn.MConnection

	nodeInfo NodeInfo
	channels []byte

	Data *cmn.CMap

	metrics       *Metrics
	metricsTicker *time.Ticker
}
```

我们先关注peer的通信模块。我们根据内部成员变量，猜测主要和通信相关的是peerConn以及mconn,点进去一看，确实没错。

### peerConn

这个定义比较简单，就是简单的原声通信，记录了通信的一些内容

```go
type peerConn struct {
	outbound   bool
	persistent bool
	conn       net.Conn // source connection

	originalAddr *NetAddress // nil for inbound connections

	// cached RemoteIP()
	ip net.IP
}
```

可以看着，peerConn维护了一个连接，和别人的通信地址。没有太难理解的内容。

### mconn

```go
type MConnection struct {
	cmn.BaseService

	conn          net.Conn
	bufConnReader *bufio.Reader
	bufConnWriter *bufio.Writer
	sendMonitor   *flow.Monitor
	recvMonitor   *flow.Monitor
	send          chan struct{}
	pong          chan struct{}
	channels      []*Channel
	channelsIdx   map[byte]*Channel
	onReceive     receiveCbFunc
	onError       errorCbFunc
	errored       uint32
	config        MConnConfig

	// Closing quitSendRoutine will cause
	// doneSendRoutine to close.
	quitSendRoutine chan struct{}
	doneSendRoutine chan struct{}

	flushTimer *cmn.ThrottleTimer // flush writes as necessary but throttled.
	pingTimer  *cmn.RepeatTimer   // send pings periodically

	// close conn if pong is not received in pongTimeout
	pongTimer     *time.Timer
	pongTimeoutCh chan bool // true - timeout, false - peer sent pong

	chStatsTimer *cmn.RepeatTimer // update channel stats periodically

	created time.Time // time of creation

	_maxPacketMsgSize int
}
```

这个好复杂，我们根据自带的注释，可以理解为，这个实现了peer和其他节点的多工通信。对于每一个通信有一个通道和通道ID。 我们来看一看具体的实现。

```go
func NewMConnectionWithConfig(conn net.Conn, chDescs []*ChannelDescriptor, onReceive receiveCbFunc, onError errorCbFunc, config MConnConfig) *MConnection {
	if config.PongTimeout >= config.PingInterval {
		panic("pongTimeout must be less than pingInterval (otherwise, next ping will reset pong timer)")
	}

	mconn := &MConnection{
		conn:          conn,
		bufConnReader: bufio.NewReaderSize(conn, minReadBufferSize),
		bufConnWriter: bufio.NewWriterSize(conn, minWriteBufferSize),
		sendMonitor:   flow.New(0, 0),
		recvMonitor:   flow.New(0, 0),
		send:          make(chan struct{}, 1),
		pong:          make(chan struct{}, 1),
		onReceive:     onReceive,
		onError:       onError,
		config:        config,
		created:       time.Now(),
	}

	// Create channels
	var channelsIdx = map[byte]*Channel{}
	var channels = []*Channel{}

	for _, desc := range chDescs {
		channel := newChannel(mconn, *desc)
		channelsIdx[channel.desc.ID] = channel
		channels = append(channels, channel)
	}
	mconn.channels = channels
	mconn.channelsIdx = channelsIdx

	mconn.BaseService = *cmn.NewBaseService(nil, "MConnection", mconn)

	// maxPacketMsgSize() is a bit heavy, so call just once
	mconn._maxPacketMsgSize = mconn.maxPacketMsgSize()

	return mconn
}
```

我们可以看到几个回调函数，onReceive和onError，而且还对conn做了二次io封装，方便进行读写。并且在这创建了好几个通道，最后创建了一个MConnection的基础服务。 通道是怎么实现的呢？

```go
type Channel struct {
	conn          *MConnection
	desc          ChannelDescriptor
	sendQueue     chan []byte
	sendQueueSize int32 // atomic.
	recving       []byte
	sending       []byte
	recentlySent  int64 // exponential moving average

	maxPacketMsgPayloadSize int

	Logger log.Logger
}
```

每一个通道都维持着一个MConnection，然后有个发送队列，看内容应该是会把发送队列的内容放到sending缓冲区发出去，然后recving是接受缓冲区。根据channel的成员函数，我们可以看到，会有goroutine调用来读写sending和recving缓冲区进行数据收发。 我们知道OnStart是开始方法，我们看下OnStart是怎么启动的吧

```go
func (c *MConnection) OnStart() error {
	if err := c.BaseService.OnStart(); err != nil {
		return err
	}
	c.quitSendRoutine = make(chan struct{})
	c.doneSendRoutine = make(chan struct{})
	c.flushTimer = cmn.NewThrottleTimer("flush", c.config.FlushThrottle)
	c.pingTimer = cmn.NewRepeatTimer("ping", c.config.PingInterval)
	c.pongTimeoutCh = make(chan bool, 1)
	c.chStatsTimer = cmn.NewRepeatTimer("chStats", updateStats)
	go c.sendRoutine()
	go c.recvRoutine()
	return nil
}
```

主要语句就两个，创建一个发协程，一个收协程。 先看一看数据如何发出去

```go
func (c *MConnection) sendRoutine() {
	defer c._recover()

FOR_LOOP:
	for {
		var _n int64
		var err error
	SELECTION:
		select {
		case <-c.flushTimer.Ch:
			// NOTE: flushTimer.Set() must be called every time
			// something is written to .bufConnWriter.
			c.flush()
		case <-c.chStatsTimer.Chan():
			for _, channel := range c.channels {
				channel.updateStats()
			}
		case <-c.pingTimer.Chan():
			c.Logger.Debug("Send Ping")
			_n, err = cdc.MarshalBinaryLengthPrefixedWriter(c.bufConnWriter, PacketPing{})
			if err != nil {
				break SELECTION
			}
			c.sendMonitor.Update(int(_n))
			c.Logger.Debug("Starting pong timer", "dur", c.config.PongTimeout)
			c.pongTimer = time.AfterFunc(c.config.PongTimeout, func() {
				select {
				case c.pongTimeoutCh <- true:
				default:
				}
			})
			c.flush()
		case timeout := <-c.pongTimeoutCh:
			if timeout {
				c.Logger.Debug("Pong timeout")
				err = errors.New("pong timeout")
			} else {
				c.stopPongTimer()
			}
		case <-c.pong:
			c.Logger.Debug("Send Pong")
			_n, err = cdc.MarshalBinaryLengthPrefixedWriter(c.bufConnWriter, PacketPong{})
			if err != nil {
				break SELECTION
			}
			c.sendMonitor.Update(int(_n))
			c.flush()
		case <-c.quitSendRoutine:
			close(c.doneSendRoutine)
			break FOR_LOOP
		case <-c.send:
			// Send some PacketMsgs
			eof := c.sendSomePacketMsgs()
			if !eof {
				// Keep sendRoutine awake.
				select {
				case c.send <- struct{}{}:
				default:
				}
			}
		}

		if !c.IsRunning() {
			break FOR_LOOP
		}
		if err != nil {
			c.Logger.Error("Connection failed @ sendRoutine", "conn", c, "err", err)
			c.stopForError(err)
			break FOR_LOOP
		}
	}

	// Cleanup
	c.stopPongTimer()
}
```

上面有ping,pong，就是和别人通信是否稳定的，不进去看了。我们看下是如何发送出去自己的数据的。也就是c.send收到了内容的时候

```go
func (c *MConnection) sendPacketMsg() bool {
	// 挑选一个channel，这个channel是最近发送比率最低的
	...

	// Nothing to send?
	if leastChannel == nil {
		return true
	}
	// c.Logger.Info("Found a msgPacket to send")

	// Make & send a PacketMsg from this channel
	_n, err := leastChannel.writePacketMsgTo(c.bufConnWriter)
	if err != nil {
		c.Logger.Error("Failed to write PacketMsg", "err", err)
		c.stopForError(err)
		return true
	}
	c.sendMonitor.Update(int(_n))
	c.flushTimer.Set()
	return false
}
```

然后我们沿着`writePacketMsgTo`看下去的话，会看到函数

```go
func (ch *Channel) nextPacketMsg() PacketMsg {
	packet := PacketMsg{}
	packet.ChannelID = byte(ch.desc.ID)
	maxSize := ch.maxPacketMsgPayloadSize
	packet.Bytes = ch.sending[:cmn.MinInt(maxSize, len(ch.sending))]
	if len(ch.sending) <= maxSize {
		packet.EOF = byte(0x01)
		ch.sending = nil
		atomic.AddInt32(&ch.sendQueueSize, -1) // decrement sendQueueSize
	} else {
		packet.EOF = byte(0x00)
		ch.sending = ch.sending[cmn.MinInt(maxSize, len(ch.sending)):]
	}
	return packet
}
```

很简单，就是如果这个包还没发送完，就从缓冲区里面继续读，继续发，如果发完了，队列就减少1.并且再发送的过程中设置结束变量。 所以呢，整体的流程就是，有包的话就从缓冲区里面读出来，封装packet然后发出去，如果包多，就一直`sendSomePackets`. 接下来再看看怎么收的。

```go
func (c *MConnection) recvRoutine() {
	defer c._recover()

FOR_LOOP:
	for {
		// Block until .recvMonitor says we can read.
		c.recvMonitor.Limit(c._maxPacketMsgSize, atomic.LoadInt64(&c.config.RecvRate), true)


		// Read packet type
		var packet Packet
		var _n int64
		var err error
		_n, err = cdc.UnmarshalBinaryLengthPrefixedReader(c.bufConnReader, &packet, int64(c._maxPacketMsgSize))
		c.recvMonitor.Update(int(_n))
		if err != nil {
			if c.IsRunning() {
				c.Logger.Error("Connection failed @ recvRoutine (reading byte)", "conn", c, "err", err)
				c.stopForError(err)
			}
			break FOR_LOOP
		}

		// Read more depending on packet type.
		switch pkt := packet.(type) {
		case PacketPing:
			// TODO: prevent abuse, as they cause flush()'s.
			// https://github.com/tendermint/tendermint/issues/1190
			c.Logger.Debug("Receive Ping")
			select {
			case c.pong <- struct{}{}:
			default:
				// never block
			}
		case PacketPong:
			c.Logger.Debug("Receive Pong")
			select {
			case c.pongTimeoutCh <- false:
			default:
				// never block
			}
		case PacketMsg:
			channel, ok := c.channelsIdx[pkt.ChannelID]
			if !ok || channel == nil {
				err := fmt.Errorf("Unknown channel %X", pkt.ChannelID)
				c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "err", err)
				c.stopForError(err)
				break FOR_LOOP
			}

			msgBytes, err := channel.recvPacketMsg(pkt)
			if err != nil {
				if c.IsRunning() {
					c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "err", err)
					c.stopForError(err)
				}
				break FOR_LOOP
			}
			if msgBytes != nil {
				c.Logger.Debug("Received bytes", "chID", pkt.ChannelID, "msgBytes", fmt.Sprintf("%X", msgBytes))
				// NOTE: This means the reactor.Receive runs in the same thread as the p2p recv routine
				c.onReceive(pkt.ChannelID, msgBytes)
			}
		default:
			err := fmt.Errorf("Unknown message type %v", reflect.TypeOf(packet))
			c.Logger.Error("Connection failed @ recvRoutine", "conn", c, "err", err)
			c.stopForError(err)
			break FOR_LOOP
		}
	}

	// Cleanup
	close(c.pong)
	for range c.pong {
		// Drain
	}
}
```

对于ping和pong我们就不看了，就是检测连接的。主要看一下PacketMsg类型的处理。再最前面发的时候，我们看到包如果没发完，标志量EOF不是01，这个地方的`channel.recvPacketMsg(pkt)`就是把一个完整的封装起来的。然后我们发现了onReceive函数，还记得最开始说的回调函数吗？就是这！

所以，MConnection的工作就是收发信息，channel是通信的渠道，发就调用底层的发出去，收就进行回调。那回调函数在哪呢？我们接下来继续看。

### peer

现在我们再看一个peer是怎么创建的

```go
func newPeer(
	pc peerConn,
	mConfig tmconn.MConnConfig,
	nodeInfo NodeInfo,
	reactorsByCh map[byte]Reactor,
	chDescs []*tmconn.ChannelDescriptor,
	onPeerError func(Peer, interface{}),
	options ...PeerOption,
) *peer {
	p := &peer{
		peerConn:      pc,
		nodeInfo:      nodeInfo,
		channels:      nodeInfo.(DefaultNodeInfo).Channels, // TODO
		Data:          cmn.NewCMap(),
		metricsTicker: time.NewTicker(metricsTickerDuration),
		metrics:       NopMetrics(),
	}

	p.mconn = createMConnection(
		pc.conn,
		p,
		reactorsByCh,
		chDescs,
		onPeerError,
		mConfig,
	)
	p.BaseService = *cmn.NewBaseService(nil, "Peer", p)
	for _, option := range options {
		option(p)
	}

	return p
}
```

等等回调函数呢？ 我们看下`createMConnection`

```go
func createMConnection(
	conn net.Conn,
	p *peer,
	reactorsByCh map[byte]Reactor,
	chDescs []*tmconn.ChannelDescriptor,
	onPeerError func(Peer, interface{}),
	config tmconn.MConnConfig,
) *tmconn.MConnection {

	onReceive := func(chID byte, msgBytes []byte) {
		reactor := reactorsByCh[chID]
		if reactor == nil {
			// Note that its ok to panic here as it's caught in the conn._recover,
			// which does onPeerError.
			panic(fmt.Sprintf("Unknown channel %X", chID))
		}
		p.metrics.PeerReceiveBytesTotal.With("peer_id", string(p.ID())).Add(float64(len(msgBytes)))
		reactor.Receive(chID, p, msgBytes)
	}

	onError := func(r interface{}) {
		onPeerError(p, r)
	}

	return tmconn.NewMConnectionWithConfig(
		conn,
		chDescs,
		onReceive,
		onError,
		config,
	)
}
```

看到了吧，在这里面,reactor的receive函数做进一步的处理。每一个channel就有一个对应的reactor来处理。 需要注意的就是这创建了一个peer的基础服务。我们看一下peer怎么工作的

```go
func (p *peer) OnStart() error {
	if err := p.BaseService.OnStart(); err != nil {
		return err
	}

	if err := p.mconn.Start(); err != nil {
		return err
	}

	go p.metricsReporter()
	return nil
}
```

这里面启动的就是基本的服务，然后是通信服务，最后还有一个统计的服务。 到这似乎就结束了。 不对！MConnection是通过Peer维护的，所以收发得靠Peer来做。看看peer怎么发的

```go
func (p *peer) Send(chID byte, msgBytes []byte) bool {
	if !p.IsRunning() {
		return false
	} else if !p.hasChannel(chID) {
		return false
	}
	res := p.mconn.Send(chID, msgBytes)
	if res {
		p.metrics.PeerSendBytesTotal.With("peer_id", string(p.ID())).Add(float64(len(msgBytes)))
	}
	return res
}

func (p *peer) TrySend(chID byte, msgBytes []byte) bool {
	if !p.IsRunning() {
		return false
	} else if !p.hasChannel(chID) {
		return false
	}
	res := p.mconn.TrySend(chID, msgBytes)
	if res {
		p.metrics.PeerSendBytesTotal.With("peer_id", string(p.ID())).Add(float64(len(msgBytes)))
	}
	return res
}
```

还是调用了底层的mconn的处理。收的处理还在reactor中，我们还没看到那一层。

## Switch

我们之前看了peer的基本操作，现在看一下更上一层的工作内容。

通过看switch的注释，不难理解，他就是个分发功能，把不同的消息分发给不同的reactor。

```go
type Switch struct {
	cmn.BaseService

	config       *config.P2PConfig
	reactors     map[string]Reactor
	chDescs      []*conn.ChannelDescriptor
	reactorsByCh map[byte]Reactor
	peers        *PeerSet
	dialing      *cmn.CMap
	reconnecting *cmn.CMap
	nodeInfo     NodeInfo // our node info
	nodeKey      *NodeKey // our node privkey
	addrBook     AddrBook

	transport Transport

	filterTimeout time.Duration
	peerFilters   []PeerFilterFunc

	rng *cmn.Rand // seed for randomizing dial times and orders

	metrics *Metrics
}
```

里面游客一个channel的集合，和reactor的对应map，石锤了！我们之前看到，根据channelId找到对应的reacotr处理onReceive，在这发现了关系。 这个transporter会有用，我们稍后介绍。 newSwitch没有什么内容，我们就不贴代码了。 看一看怎么启动的

```go
func (sw *Switch) OnStart() error {
	// Start reactors
	for _, reactor := range sw.reactors {
		err := reactor.Start()
		if err != nil {
			return cmn.ErrorWrap(err, "failed to start %v", reactor)
		}
	}

	// Start accepting Peers.
	go sw.acceptRoutine()

	return nil
}
```

首先是启动所有的reactor，然后开始了一个接受peer的协程。 能够启动reactor，前提是添加进来了，我们看下是怎么添加的

```go
func (sw *Switch) AddReactor(name string, reactor Reactor) Reactor {
	reactorChannels := reactor.GetChannels()
	for _, chDesc := range reactorChannels {
		chID := chDesc.ID
		if sw.reactorsByCh[chID] != nil {
			cmn.PanicSanity(fmt.Sprintf("Channel %X has multiple reactors %v & %v", chID, sw.reactorsByCh[chID], reactor))
		}
		sw.chDescs = append(sw.chDescs, chDesc)
		sw.reactorsByCh[chID] = reactor
	}
	sw.reactors[name] = reactor
	reactor.SetSwitch(sw)
	return reactor
}
```

主要的目标就是往map里面填数据，不过有个地方

```go
reactor.SetSwitch(sw)
```

添加了对switch的引用，方便了reactor和switch的操作。 接下来是接受peer的协程

```go
func (sw *Switch) acceptRoutine() {
	for {
		p, err := sw.transport.Accept(peerConfig{
			chDescs:      sw.chDescs,
			onPeerError:  sw.StopPeerForError,
			reactorsByCh: sw.reactorsByCh,
			metrics:      sw.metrics,
		})
		if err != nil {
			switch err.(type) {
			case ErrRejected:
				rErr := err.(ErrRejected)

				if rErr.IsSelf() {
					// Remove the given address from the address book and add to our addresses
					// to avoid dialing in the future.
					addr := rErr.Addr()
					sw.addrBook.RemoveAddress(&addr)
					sw.addrBook.AddOurAddress(&addr)
				}

				sw.Logger.Info(
					"Inbound Peer rejected",
					"err", err,
					"numPeers", sw.peers.Size(),
				)

				continue
			case *ErrTransportClosed:
				sw.Logger.Error(
					"Stopped accept routine, as transport is closed",
					"numPeers", sw.peers.Size(),
				)
			default:
				sw.Logger.Error(
					"Accept on transport errored",
					"err", err,
					"numPeers", sw.peers.Size(),
				)
				panic(fmt.Errorf("accept routine exited: %v", err))
			}

			break
		}

		// Ignore connection if we already have enough peers.
		_, in, _ := sw.NumPeers()
		if in >= sw.config.MaxNumInboundPeers {
			sw.Logger.Info(
				"Ignoring inbound connection: already have enough inbound peers",
				"address", p.NodeInfo().NetAddress().String(),
				"have", in,
				"max", sw.config.MaxNumInboundPeers,
			)

			_ = p.Stop()

			continue
		}

		if err := sw.addPeer(p); err != nil {
			_ = p.Stop()
			sw.Logger.Info(
				"Ignoring inbound connection: error while adding peer",
				"err", err,
				"id", p.ID(),
			)
		}
	}
}
```

可以看到，首先接受一个peer的连接，具体如何接受的我们稍后分析。首先就是如果连接失败的一系列处理。如果连接成功了，但是peer已经连接足够多的节点了，就不用再连了。否则，添加进去。 我们接着看如何添加进去一个新的peer的。

```go
func (sw *Switch) addPeer(p Peer) error {
	if err := sw.filterPeer(p); err != nil {
		return err
	}

	p.SetLogger(sw.Logger.With("peer", p.NodeInfo().NetAddress()))

	// All good. Start peer
	if sw.IsRunning() {
		if err := sw.startInitPeer(p); err != nil {
			return err
		}
	}

	if err := sw.peers.Add(p); err != nil {
		return err
	}

	sw.Logger.Info("Added peer", "peer", p)
	sw.metrics.Peers.Add(float64(1))

	return nil
}
```

过滤peer的操作是看看peer是否符合要求，我们不进去细看了。如果一切正常的话，就添加到peer列表中，并且初始化peer并运行。

```go
func (sw *Switch) startInitPeer(p Peer) error {
	err := p.Start() // spawn send/recv routines
	if err != nil {
		// Should never happen
		sw.Logger.Error(
			"Error starting peer",
			"err", err,
			"peer", p,
		)
		return err
	}

	for _, reactor := range sw.reactors {
		reactor.AddPeer(p)
	}

	return nil
}
```

首先启动peer没什么可说的，只是，这个地方还为reactors添加了需要服务的peer，方便reactor使用。 所以呢，一个服务端，接受别人的连接，然后检查完后创建一个peer，为别人提供服务。 现在只监听别人的连接，节点应该也有自己的连接

```go
func (sw *Switch) addOutboundPeerWithConfig(
	addr *NetAddress,
	cfg *config.P2PConfig,
	persistent bool,
) error {
	sw.Logger.Info("Dialing peer", "address", addr)

	// XXX(xla): Remove the leakage of test concerns in implementation.
	if cfg.TestDialFail {
		go sw.reconnectToPeer(addr)
		return fmt.Errorf("dial err (peerConfig.DialFail == true)")
	}

	p, err := sw.transport.Dial(*addr, peerConfig{
		chDescs:      sw.chDescs,
		onPeerError:  sw.StopPeerForError,
		persistent:   persistent,
		reactorsByCh: sw.reactorsByCh,
		metrics:      sw.metrics,
	})
	if err != nil {
		switch e := err.(type) {
		case ErrRejected:
			if e.IsSelf() {
				// Remove the given address from the address book and add to our addresses
				// to avoid dialing in the future.
				sw.addrBook.RemoveAddress(addr)
				sw.addrBook.AddOurAddress(addr)

				return err
			}
		}

		// retry persistent peers after
		// any dial error besides IsSelf()
		if persistent {
			go sw.reconnectToPeer(addr)
		}

		return err
	}

	if err := sw.addPeer(p); err != nil {
		_ = p.Stop()
		return err
	}

	return nil
}
```

差不多，前面是transporter的accept，接受别人的连接，现在换成dial，主动连接别人。 现在我们要看下transporter是怎么工作的了。 transporter是一个接口

```go
type Transport interface {
	// Accept returns a newly connected Peer.
	Accept(peerConfig) (Peer, error)

	// Dial connects to the Peer for the address.
	Dial(NetAddress, peerConfig) (Peer, error)
}
```

我们之前在看peer的时候知道，他们是多工工作机制，很简单的在transporter包里面看到了类型定义

```go
type MultiplexTransport struct {
	listener net.Listener

	acceptc chan accept
	closec  chan struct{}

	// Lookup table for duplicate ip and id checks.
	conns       ConnSet
	connFilters []ConnFilterFunc

	dialTimeout      time.Duration
	filterTimeout    time.Duration
	handshakeTimeout time.Duration
	nodeInfo         NodeInfo
	nodeKey          NodeKey
	resolver         IPResolver

	mConfig conn.MConnConfig
}
```

新建函数没有什么难以理解的骂我们就看一看对`accept`以及`dial`的实现吧

```go
func (mt *MultiplexTransport) Accept(cfg peerConfig) (Peer, error) {
	select {
	// This case should never have any side-effectful/blocking operations to
	// ensure that quality peers are ready to be used.
	case a := <-mt.acceptc:
		if a.err != nil {
			return nil, a.err
		}

		cfg.outbound = false

		return mt.wrapPeer(a.conn, a.nodeInfo, cfg, nil), nil
	case <-mt.closec:
		return nil, &ErrTransportClosed{}
	}
}
```

可以看到，一个连接过来的peer被创建了出来，并且返还回去，添加到reactor里面。 怎么主动和别的节点连接的呢？

```go
func (mt *MultiplexTransport) Dial(
	addr NetAddress,
	cfg peerConfig,
) (Peer, error) {
	c, err := addr.DialTimeout(mt.dialTimeout)
	if err != nil {
		return nil, err
	}

	// TODO(xla): Evaluate if we should apply filters if we explicitly dial.
	if err := mt.filterConn(c); err != nil {
		return nil, err
	}

	secretConn, nodeInfo, err := mt.upgrade(c)
	if err != nil {
		return nil, err
	}

	cfg.outbound = true

	p := mt.wrapPeer(secretConn, nodeInfo, cfg, &addr)

	return p, nil
}
```

几乎一样的操作，不过在生成peer之前，要先进行下连接的过滤，看看是否已经创建连接，连接是否有问题，然后更新下连接之后，就可以生成新的创建了。

## PEXReactor

我们之前看了Tendermint如何和别人连接，如何接受别人的连接，可是，Tendermint得怎么找到别人啊！ 不能手动一个一个的输入连接地址啊！ 这个就是pexreactor的功能了！

我们首先看下PEXReactor怎么定义的

```go
type PEXReactor struct {
	p2p.BaseReactor

	book              AddrBook
	config            *PEXReactorConfig
	ensurePeersPeriod time.Duration // TODO: should go in the config

	// maps to prevent abuse
	requestsSent         *cmn.CMap // ID->struct{}: unanswered send requests
	lastReceivedRequests *cmn.CMap // ID->time.Time: last time peer requested from us

	seedAddrs []*p2p.NetAddress

	attemptsToDial sync.Map // address (string) -> {number of attempts (int), last time dialed (time.Time)}
}
```

很简单，继承了基础的reactor，然后有一些地址表，我们具体的去看一看。

```go
func NewPEXReactor(b AddrBook, config *PEXReactorConfig) *PEXReactor {
	r := &PEXReactor{
		book:                 b,
		config:               config,
		ensurePeersPeriod:    defaultEnsurePeersPeriod,
		requestsSent:         cmn.NewCMap(),
		lastReceivedRequests: cmn.NewCMap(),
	}
	r.BaseReactor = *p2p.NewBaseReactor("PEXReactor", r)
	return r
}
```

还好，就是把传递的变量赋值进去，然后创建了一个reactor，就像我们之前看到的service一样。 是怎么启动起来的呢？

```go
func (r *PEXReactor) OnStart() error {
	err := r.book.Start()
	if err != nil && err != cmn.ErrAlreadyStarted {
		return err
	}

	numOnline, seedAddrs, err := r.checkSeeds()
	if err != nil {
		return err
	} else if numOnline == 0 && r.book.Empty() {
		return errors.New("Address book is empty, and could not connect to any seed nodes")
	}

	r.seedAddrs = seedAddrs

	// Check if this node should run
	// in seed/crawler mode
	if r.config.SeedMode {
		go r.crawlPeersRoutine()
	} else {
		go r.ensurePeersRoutine()
	}
	return nil
}
```

首先是启动地址簿，我们答题说几句，就是会从我们的配置文件中读取地址信息，然后保存下来。具体的我们以后再分析。 接下来是检查这些种子节点是否有错误。主要检查的什么呢？ 检查的就是种子节点的内容格式是否正确，是否能够联通，然后返回可用的种子 最后根据配置，看自己是否是个种子节点，如果是的话，就去抓取其他的节点，如果不是，就是保证节点连接就好了。 我们来看看作为种子节点是怎么工作的

```go
func (r *PEXReactor) crawlPeersRoutine() {
	// Do an initial crawl
	r.crawlPeers()

	// Fire periodically
	ticker := time.NewTicker(defaultCrawlPeersPeriod)

	for {
		select {
		case <-ticker.C:
			r.attemptDisconnects()
			r.crawlPeers()
		case <-r.Quit():
			return
		}
	}
}
```

很明显，先抓取一次其他的节点，然后定时轮询取获取其他的节点。

```go
func (r *PEXReactor) crawlPeers() {
	peerInfos := r.getPeersToCrawl()

	now := time.Now()
	// Use addresses we know of to reach additional peers
	for _, pi := range peerInfos {
		// Do not attempt to connect with peers we recently dialed
		if now.Sub(pi.LastAttempt) < defaultCrawlPeerInterval {
			continue
		}
		// Otherwise, attempt to connect with the known address
		err := r.Switch.DialPeerWithAddress(pi.Addr, false)
		if err != nil {
			r.book.MarkAttempt(pi.Addr)
			continue
		}
		// Ask for more addresses
		peer := r.Switch.Peers().Get(pi.Addr.ID)
		if peer != nil {
			r.RequestAddrs(peer)
		}
	}
}
```

整体流程就是获取新的节点去连接。怎么获取的新的节点呢？就是向已连接过的节点去请求新的节点，类似于gossip协议，一层一层的扩散出去找新的节点。 我们再看下普通节点时候如何工作的

```go
func (r *PEXReactor) ensurePeersRoutine() {
	var (
		seed   = cmn.NewRand()
		jitter = seed.Int63n(r.ensurePeersPeriod.Nanoseconds())
	)

	// Randomize first round of communication to avoid thundering herd.
	// If no potential peers are present directly start connecting so we guarantee
	// swift setup with the help of configured seeds.
	if r.hasPotentialPeers() {
		time.Sleep(time.Duration(jitter))
	}

	// fire once immediately.
	// ensures we dial the seeds right away if the book is empty
	r.ensurePeers()

	// fire periodically
	ticker := time.NewTicker(r.ensurePeersPeriod)
	for {
		select {
		case <-ticker.C:
			r.ensurePeers()
		case <-r.Quit():
			ticker.Stop()
			return
		}
	}
}
```

和作为种子节点差不多，一定时间情况下就去维持连接

```go
func (r *PEXReactor) ensurePeers() {
	var (
		out, in, dial = r.Switch.NumPeers()
		numToDial     = r.Switch.MaxNumOutboundPeers() - (out + dial)
	)
	r.Logger.Info(
		"Ensure peers",
		"numOutPeers", out,
		"numInPeers", in,
		"numDialing", dial,
		"numToDial", numToDial,
	)

	if numToDial <= 0 {
		return
	}

	// bias to prefer more vetted peers when we have fewer connections.
	// not perfect, but somewhate ensures that we prioritize connecting to more-vetted
	// NOTE: range here is [10, 90]. Too high ?
	newBias := cmn.MinInt(out, 8)*10 + 10

	toDial := make(map[p2p.ID]*p2p.NetAddress)
	// Try maxAttempts times to pick numToDial addresses to dial
	maxAttempts := numToDial * 3

	for i := 0; i < maxAttempts && len(toDial) < numToDial; i++ {
		try := r.book.PickAddress(newBias)
		if try == nil {
			continue
		}
		if _, selected := toDial[try.ID]; selected {
			continue
		}
		if r.Switch.IsDialingOrExistingAddress(try) {
			continue
		}
		// TODO: consider moving some checks from toDial into here
		// so we don't even consider dialing peers that we want to wait
		// before dialling again, or have dialed too many times already
		r.Logger.Info("Will dial address", "addr", try)
		toDial[try.ID] = try
	}

	// Dial picked addresses
	for _, addr := range toDial {
		go r.dialPeer(addr)
	}

	// If we need more addresses, pick a random peer and ask for more.
	if r.book.NeedMoreAddrs() {
		peers := r.Switch.Peers().List()
		peersCount := len(peers)
		if peersCount > 0 {
			peer := peers[cmn.RandInt()%peersCount] // nolint: gas
			r.Logger.Info("We need more addresses. Sending pexRequest to random peer", "peer", peer)
			r.RequestAddrs(peer)
		}
	}

	// If we are not connected to nor dialing anybody, fallback to dialing a seed.
	if out+in+dial+len(toDial) == 0 {
		r.Logger.Info("No addresses to dial nor connected peers. Falling back to seeds")
		r.dialSeeds()
	}
}
```

整体过程就是挑选出需要连接的peer节点，然后进行连接，如果本peer存的地址太少的话，就请求更多的节点来。 现在我们能够发现，节点可以连接到别人上面，别的节点应该能够连接到本节点，怎么处理呢？

```go
func (r *PEXReactor) Receive(chID byte, src Peer, msgBytes []byte) {
	msg, err := decodeMsg(msgBytes)
	if err != nil {
		r.Logger.Error("Error decoding message", "src", src, "chId", chID, "msg", msg, "err", err, "bytes", msgBytes)
		r.Switch.StopPeerForError(src, err)
		return
	}
	r.Logger.Debug("Received message", "src", src, "chId", chID, "msg", msg)

	switch msg := msg.(type) {
	case *pexRequestMessage:

		// NOTE: this is a prime candidate for amplification attacks,
		// so it's important we
		// 1) restrict how frequently peers can request
		// 2) limit the output size

		// If we're a seed and this is an inbound peer,
		// respond once and disconnect.
		if r.config.SeedMode && !src.IsOutbound() {
			id := string(src.ID())
			v := r.lastReceivedRequests.Get(id)
			if v != nil {
				// FlushStop/StopPeer are already
				// running in a go-routine.
				return
			}
			r.lastReceivedRequests.Set(id, time.Now())

			// Send addrs and disconnect
			r.SendAddrs(src, r.book.GetSelectionWithBias(biasToSelectNewPeers))
			go func() {
				// In a go-routine so it doesn't block .Receive.
				src.FlushStop()
				r.Switch.StopPeerGracefully(src)
			}()

		} else {
			// Check we're not receiving requests too frequently.
			if err := r.receiveRequest(src); err != nil {
				r.Switch.StopPeerForError(src, err)
				return
			}
			r.SendAddrs(src, r.book.GetSelection())
		}

	case *pexAddrsMessage:
		// If we asked for addresses, add them to the book
		if err := r.ReceiveAddrs(msg.Addrs, src); err != nil {
			r.Switch.StopPeerForError(src, err)
			return
		}
	default:
		r.Logger.Error(fmt.Sprintf("Unknown message type %v", reflect.TypeOf(msg)))
	}
}
```

就是对节点收到的pex信息进行处理，包括像别人请求时别的peer相应的信息以及本peer响应别的节点的信息。

如果我们整体来看PEXReactor的话，就是实现了连接的管理和维护，可以实现向别的节点请求连接，更新本地的连接，然后也可以响应别的节点的请求，本地做出回应。

## Mempool

我们看完了peer，看点稍微简单点的mempool实现吧。 这个和我们写的abci程序有直接的关联，因此，我们需要仔细阅读下。

废话不多说，我们先看下Mempool的定义

```go
type Mempool struct {
	config *cfg.MempoolConfig

	proxyMtx             sync.Mutex
	proxyAppConn         proxy.AppConnMempool
	txs                  *clist.CList    // concurrent linked-list of good txs
	height               int64           // the last block Update()'d to
	rechecking           int32           // for re-checking filtered txs on Update()
	recheckCursor        *clist.CElement // next expected response
	recheckEnd           *clist.CElement // re-checking stops here
	notifiedTxsAvailable bool
	txsAvailable         chan struct{} // fires once for each height, when the mempool is not empty
	preCheck             PreCheckFunc
	postCheck            PostCheckFunc

	// Keep a cache of already-seen txs.
	// This reduces the pressure on the proxyApp.
	cache txCache

	// A log of mempool txs
	wal *auto.AutoFile

	logger log.Logger

	metrics *Metrics
}
```

就是一堆和和交易相关的内容，还有一些和上层通信所需要的接口。 我们看下怎么定义的

```go
func NewMempool(
	config *cfg.MempoolConfig,
	proxyAppConn proxy.AppConnMempool,
	height int64,
	options ...MempoolOption,
) *Mempool {
	mempool := &Mempool{
		config:        config,
		proxyAppConn:  proxyAppConn,
		txs:           clist.New(),
		height:        height,
		rechecking:    0,
		recheckCursor: nil,
		recheckEnd:    nil,
		logger:        log.NewNopLogger(),
		metrics:       NopMetrics(),
	}
	if config.CacheSize > 0 {
		mempool.cache = newMapTxCache(config.CacheSize)
	} else {
		mempool.cache = nopTxCache{}
	}
	proxyAppConn.SetResponseCallback(mempool.resCb)
	for _, option := range options {
		option(mempool)
	}
	return mempool
}
```

里面有一个需要注意的，就是为proxyAppConn设置了一个回调。 因为我们知道，内存池把一个交易提交给我们的abci程序之后，根据返回的结果来决定下一步如何处理，所以这个地方定义了`mempool.resCb`。 我们看一下时怎么从内存池中取出来一个交易的。为了方便起见，不妨假设是按照交易笔数取出来的

```go
func (mem *Mempool) ReapMaxTxs(max int) types.Txs {
	mem.proxyMtx.Lock()
	defer mem.proxyMtx.Unlock()

	if max < 0 {
		max = mem.txs.Len()
	}

	for atomic.LoadInt32(&mem.rechecking) > 0 {
		// TODO: Something better?
		time.Sleep(time.Millisecond * 10)
	}

	txs := make([]types.Tx, 0, cmn.MinInt(mem.txs.Len(), max))
	for e := mem.txs.Front(); e != nil && len(txs) <= max; e = e.Next() {
		memTx := e.Value.(*mempoolTx)
		txs = append(txs, memTx.tx)
	}
	return txs
}
```

主要流程不复杂，就是从tx列表中按照要求的数目来读取交易，按照用户要求的数目读取出来，然后返回回去。 那怎么更新的呢？

```go
func (mem *Mempool) Update(
	height int64,
	txs types.Txs,
	preCheck PreCheckFunc,
	postCheck PostCheckFunc,
) error {
	// Set height
	mem.height = height
	mem.notifiedTxsAvailable = false

	if preCheck != nil {
		mem.preCheck = preCheck
	}
	if postCheck != nil {
		mem.postCheck = postCheck
	}

	// Add committed transactions to cache (if missing).
	for _, tx := range txs {
		_ = mem.cache.Push(tx)
	}

	// Remove committed transactions.
	txsLeft := mem.removeTxs(txs)

	// Either recheck non-committed txs to see if they became invalid
	// or just notify there're some txs left.
	if len(txsLeft) > 0 {
		if mem.config.Recheck {
			mem.logger.Info("Recheck txs", "numtxs", len(txsLeft), "height", height)
			mem.recheckTxs(txsLeft)
			// At this point, mem.txs are being rechecked.
			// mem.recheckCursor re-scans mem.txs and possibly removes some txs.
			// Before mem.Reap(), we should wait for mem.recheckCursor to be nil.
		} else {
			mem.notifyTxsAvailable()
		}
	}

	// Update metrics
	mem.metrics.Size.Set(float64(mem.Size()))

	return nil
}
```

重点就是`txsLeft := mem.removeTxs(txs)`会把已经提交的tx给移除出去，然后其他的就都无所谓了。 接下来再看看如何检查交易的，换句话说，如何把底层的交易传递给abci程序的

```go
func (mem *Mempool) CheckTx(tx types.Tx, cb func(*abci.Response)) (err error) {
	mem.proxyMtx.Lock()
	// use defer to unlock mutex because application (*local client*) might panic
	defer mem.proxyMtx.Unlock()

	if mem.Size() >= mem.config.Size {
		return ErrMempoolIsFull
	}

	if mem.preCheck != nil {
		if err := mem.preCheck(tx); err != nil {
			return ErrPreCheck{err}
		}
	}

	// CACHE
	if !mem.cache.Push(tx) {
		return ErrTxInCache
	}
	// END CACHE

	// WAL
	if mem.wal != nil {
		// TODO: Notify administrators when WAL fails
		_, err := mem.wal.Write([]byte(tx))
		if err != nil {
			mem.logger.Error("Error writing to WAL", "err", err)
		}
		_, err = mem.wal.Write([]byte("\n"))
		if err != nil {
			mem.logger.Error("Error writing to WAL", "err", err)
		}
	}
	// END WAL

	// NOTE: proxyAppConn may error if tx buffer is full
	if err = mem.proxyAppConn.Error(); err != nil {
		return err
	}
	reqRes := mem.proxyAppConn.CheckTxAsync(tx)
	if cb != nil {
		reqRes.SetCallback(cb)
	}

	return nil
}
```

可以看到，最后的方法就是`reqRes := mem.proxyAppConn.CheckTxAsync(tx)`来调用上层的内容，获取处理结果。 那么`checkTxAsync`具体怎么工作的呢？点进去我们发现proxyAppConn是一个接口？再同一个文件中查找，我们会发现只有一个实现了这个接口，就是`appConnMempool`，具体的定义如下

```go
type appConnMempool struct {
	appConn abcicli.Client
}
```

然后往下看程序继承实现

```go
func (app *appConnMempool) CheckTxAsync(tx []byte) *abcicli.ReqRes {
	return app.appConn.CheckTxAsync(tx)
}
```

再进去一看，又是接口实现。根据我们对abci的了解，有三种实现，socket,rpc以及直接golang交互，我们为了简单，看直接和golang交互的。 localClient的实现如下

```go
type localClient struct {
	cmn.BaseService

	mtx *sync.Mutex
	types.Application
	Callback
}
```

里面的types.Application就是我们自己的abci程序，看下checkTxAsync，可以看到

```go
func (app *localClient) CheckTxAsync(tx []byte) *ReqRes {
	app.mtx.Lock()
	defer app.mtx.Unlock()

	res := app.Application.CheckTx(tx)
	return app.callback(
		types.ToRequestCheckTx(tx),
		types.ToResponseCheckTx(res),
	)
}
```

最后面return了一下回调，因为我们之前看到代码是

```go
proxyAppConn.SetResponseCallback(mempool.resCb)
```

这个return就是使得回调产生了作用，那我们再看看resCb。

```go
func (mem *Mempool) resCb(req *abci.Request, res *abci.Response) {
	if mem.recheckCursor == nil {
		mem.resCbNormal(req, res)
	} else {
		mem.metrics.RecheckTimes.Add(1)
		mem.resCbRecheck(req, res)
	}
	mem.metrics.Size.Set(float64(mem.Size()))
}
```

正常情况下，我们应该去追踪Normal的情况

```go
func (mem *Mempool) resCbNormal(req *abci.Request, res *abci.Response) {
	switch r := res.Value.(type) {
	case *abci.Response_CheckTx:
		tx := req.GetCheckTx().Tx
		var postCheckErr error
		if mem.postCheck != nil {
			postCheckErr = mem.postCheck(tx, r.CheckTx)
		}
		if (r.CheckTx.Code == abci.CodeTypeOK) && postCheckErr == nil {
			memTx := &mempoolTx{
				height:    mem.height,
				gasWanted: r.CheckTx.GasWanted,
				tx:        tx,
			}
			mem.txs.PushBack(memTx)
			mem.logger.Info("Added good transaction",
				"tx", TxID(tx),
				"res", r,
				"height", memTx.height,
				"total", mem.Size(),
			)
			mem.metrics.TxSizeBytes.Observe(float64(len(tx)))
			mem.notifyTxsAvailable()
		} else {
			// ignore bad transaction
			mem.logger.Info("Rejected bad transaction", "tx", TxID(tx), "res", r, "err", postCheckErr)
			mem.metrics.FailedTxs.Add(1)
			// remove from cache (it might be good later)
			mem.cache.Remove(tx)
		}
	default:
		// ignore other messages
	}
}
```

看这段代码，我们就可以看到如果abci程序返回的结果正确，就可以添加到内存池中了。 可是问题来了，我们刚刚都是在考虑checkTx是如何执行的，那么是谁调用的呢？Reactor！

```go
type MempoolReactor struct {
	p2p.BaseReactor
	config  *cfg.MempoolConfig
	Mempool *Mempool
}
```

类型定义很简单。 正常情况下得有OnStart函数，我们点进去一看，很简单，就不看了。 再看一下Receive函数

```go
func (memR *MempoolReactor) Receive(chID byte, src p2p.Peer, msgBytes []byte) {
	msg, err := decodeMsg(msgBytes)
	if err != nil {
		memR.Logger.Error("Error decoding message", "src", src, "chId", chID, "msg", msg, "err", err, "bytes", msgBytes)
		memR.Switch.StopPeerForError(src, err)
		return
	}
	memR.Logger.Debug("Receive", "src", src, "chId", chID, "msg", msg)

	switch msg := msg.(type) {
	case *TxMessage:
		err := memR.Mempool.CheckTx(msg.Tx, nil)
		if err != nil {
			memR.Logger.Info("Could not check tx", "tx", TxID(msg.Tx), "err", err)
		}
		// broadcasting happens from go routines per peer
	default:
		memR.Logger.Error(fmt.Sprintf("Unknown message type %v", reflect.TypeOf(msg)))
	}
}
```

也不难理解，就是把接收到的消息添加到内存池当中，然后进行checkTx。 那么如何给别人发呢？就是在AddPeer中。

```go
func (memR *MempoolReactor) AddPeer(peer p2p.Peer) {
	go memR.broadcastTxRoutine(peer)
}
```

看下启动的协程是如何工作的。

```go
func (memR *MempoolReactor) broadcastTxRoutine(peer p2p.Peer) {
	if !memR.config.Broadcast {
		return
	}

	var next *clist.CElement
	for {
		// This happens because the CElement we were looking at got garbage
		// collected (removed). That is, .NextWait() returned nil. Go ahead and
		// start from the beginning.
		if next == nil {
			select {
			case <-memR.Mempool.TxsWaitChan(): // Wait until a tx is available
				if next = memR.Mempool.TxsFront(); next == nil {
					continue
				}
			case <-peer.Quit():
				return
			case <-memR.Quit():
				return
			}
		}

		memTx := next.Value.(*mempoolTx)

		// make sure the peer is up to date
		peerState, ok := peer.Get(types.PeerStateKey).(PeerState)
		if !ok {
			// Peer does not have a state yet. We set it in the consensus reactor, but
			// when we add peer in Switch, the order we call reactors#AddPeer is
			// different every time due to us using a map. Sometimes other reactors
			// will be initialized before the consensus reactor. We should wait a few
			// milliseconds and retry.
			time.Sleep(peerCatchupSleepIntervalMS * time.Millisecond)
			continue
		}
		if peerState.GetHeight() < memTx.Height()-1 { // Allow for a lag of 1 block
			time.Sleep(peerCatchupSleepIntervalMS * time.Millisecond)
			continue
		}

		// send memTx
		msg := &TxMessage{Tx: memTx.tx}
		success := peer.Send(MempoolChannel, cdc.MustMarshalBinaryBare(msg))
		if !success {
			time.Sleep(peerCatchupSleepIntervalMS * time.Millisecond)
			continue
		}

		select {
		case <-next.NextWaitChan():
			// see the start of the for loop for nil check
			next = next.Next()
		case <-peer.Quit():
			return
		case <-memR.Quit():
			return
		}
	}
}
```

还行，还可以看懂，就是看下peer的状态，然后封装mempool的交易，发送给peer。 到这块，内存池的内容就结束了，不难理解，这个内存池只是一个存储区，对交易的内容不做任何处理，来了新的交易就取出来发给上层abci程序，如果通过就放到内存池，然后等待共识层接受，给出去。所以是一笔一笔的交易的转移。

## crypto

看了两个复杂的，我们来看一个简单的模块，crypto模块。这个模块封装了很多常用的加密所用的包。我们就大体介绍一下。

这个包最重要的是两个接口定义`PubKey`和`PrivKey`的定义，分别如下

```go
type PubKey interface {
	Address() Address
	Bytes() []byte
	VerifyBytes(msg []byte, sig []byte) bool
	Equals(PubKey) bool
}

type PrivKey interface {
	Bytes() []byte
	Sign(msg []byte) ([]byte, error)
	PubKey() PubKey
	Equals(PrivKey) bool
}
```

后续的就是这个的实现及应用。我们一个包一个包的看。 `armor`包是一个对数据封装和解析的模块，把数据按照要求来封装和解析，没有难点。 `ed25519`是一种加密方式，实现了我们之前说到的这两个接口。 `encoding`是对公钥和私钥进行序列化的处理，是使用了amino提供的接口来处理的。 `merkle`就是常说的默克尔树了，这里面提供了常见的建立数，验证树的操作。想到我很久之前还写过一次生成默克尔树的源码呢！ `multisig`就是对信息的多人签名，这个在比特币中有过应用，在这由于还没看完源码，不清楚如何使用的。 `secp256k1`是两一种加密方式，这个地方是调用的btcsuite的库。看来btcd已经成为go常用的包之一了。 `tmhash`这个是对sha256hash的封装`，挺简单的。 `xchacha20poly1305`对称加密aead算法的一种实现,这个chacha20在ssr中目前应用的比较广泛，因为资源占用少，速度快。 `xsalsa20symmetric`也是一个对称加密。 看起来这个的实现实际比较少，也比较简单，如果需要调用库的话还不如直接去使用btcd的库了。

## blockchain

这次我们来看一个复杂的模块，blockchain模块，实际上我个人感觉，叫block模块似乎更好。

先看一下基本的区块池的定义

```go
type BlockPool struct {
	cmn.BaseService
	startTime time.Time

	mtx sync.Mutex
	// 请求下载区块的进程
	requesters map[int64]*bpRequester
	height     int64 // 最低的区块高度
	// 其他的对等节点
	peers         map[p2p.ID]*bpPeer
	maxPeerHeight int64

	// atomic
	numPending int32 // 等待分配下载区块请求或者等待相应的requester的数目

	requestsCh chan<- BlockRequest
	errorsCh   chan<- peerError
}
```

这里面的成员变量还能够看懂，就是等到看的时候不一定能够很清晰的弄明白。我们继续往后面看。 新建函数是这样的

```go
func NewBlockPool(start int64, requestsCh chan<- BlockRequest, errorsCh chan<- peerError) *BlockPool {
	bp := &BlockPool{
		peers: make(map[p2p.ID]*bpPeer),

		requesters: make(map[int64]*bpRequester),
		height:     start,
		numPending: 0,

		requestsCh: requestsCh,
		errorsCh:   errorsCh,
	}
	bp.BaseService = *cmn.NewBaseService(nil, "BlockPool", bp)
	return bp
}
```

很简单，没有复杂的功能。那我们看一下启动函数吧。

```go
func (pool *BlockPool) OnStart() error {
	go pool.makeRequestersRoutine()
	pool.startTime = time.Now()
	return nil
}
```

主要是启动了一个协程，看来主要负责下载区块的都在这了。

```go
func (pool *BlockPool) makeRequestersRoutine() {
	for {
		if !pool.IsRunning() {
			break
		}

		_, numPending, lenRequesters := pool.GetStatus()
		if numPending >= maxPendingRequests {
			// sleep for a bit.
			time.Sleep(requestIntervalMS * time.Millisecond)
			// check for timed out peers
			pool.removeTimedoutPeers()
		} else if lenRequesters >= maxTotalRequesters {
			// sleep for a bit.
			time.Sleep(requestIntervalMS * time.Millisecond)
			// check for timed out peers
			pool.removeTimedoutPeers()
		} else {
			// request for more blocks.
			pool.makeNextRequester()
		}
	}
}
```

可以看到，如果我们分配的request足够了，或者等待相应的进程够了，我们就不再请求新的了。否则的话，就创建进程，继续下载。

```go
func (pool *BlockPool) makeNextRequester() {
	pool.mtx.Lock()
	defer pool.mtx.Unlock()

	nextHeight := pool.height + pool.requestersLen()
	if nextHeight > pool.maxPeerHeight {
		return
	}

	request := newBPRequester(pool, nextHeight)

	pool.requesters[nextHeight] = request
	atomic.AddInt32(&pool.numPending, 1)

	err := request.Start()
	if err != nil {
		request.Logger.Error("Error starting request", "err", err)
	}
}
```

可以看到，每次都是创建一个requester来下载一个高度的区块，如果要下载的区块高度，超过了总的区块个数，那就不要下载了。 因为这个下载池肯定要给每个高度分配一个requester，所以创建了一个map来存储对应关系。最后启动起来进行下载。 看一看是怎么下载的 我们可以看到，下载时创建的BPRequester对象，然后下载时启动的Start函数

```go
func (bpr *bpRequester) OnStart() error {
	go bpr.requestRoutine()
	return nil
}
```

又是一个协程

```go
func (bpr *bpRequester) requestRoutine() {
OUTER_LOOP:
	for {
		// Pick a peer to send request to.
		var peer *bpPeer
	PICK_PEER_LOOP:
		for {
			if !bpr.IsRunning() || !bpr.pool.IsRunning() {
				return
			}
			peer = bpr.pool.pickIncrAvailablePeer(bpr.height)
			if peer == nil {
				//log.Info("No peers available", "height", height)
				time.Sleep(requestIntervalMS * time.Millisecond)
				continue PICK_PEER_LOOP
			}
			break PICK_PEER_LOOP
		}
		bpr.mtx.Lock()
		bpr.peerID = peer.id
		bpr.mtx.Unlock()

		// Send request and wait.
		bpr.pool.sendRequest(bpr.height, peer.id)
	WAIT_LOOP:
		for {
			select {
			case <-bpr.pool.Quit():
				bpr.Stop()
				return
			case <-bpr.Quit():
				return
			case peerID := <-bpr.redoCh:
				if peerID == bpr.peerID {
					bpr.reset()
					continue OUTER_LOOP
				} else {
					continue WAIT_LOOP
				}
			case <-bpr.gotBlockCh:
				// We got a block!
				// Continue the for-loop and wait til Quit.
				continue WAIT_LOOP
			}
		}
	}
}
```

首先，对于指定的区块高度，就是选择一个peer进行下载，如果找不到，就等等，只要能够等到。 前面的几个case，很好理解，就是对应的内容做对应的操作。最后一个没有做处理，只是continue，为什么呢？因为这个地方接收到了区块，会发送给reactor，交给共识处理去了。 接下来，我们就要看下reactor了。

```go
type BlockchainReactor struct {
	p2p.BaseReactor

	// immutable
	initialState sm.State

	blockExec *sm.BlockExecutor
	store     *BlockStore
	pool      *BlockPool
	fastSync  bool

	requestsCh <-chan BlockRequest
	errorsCh   <-chan peerError
}
```

这里面涉及到了其他的模块，我们暂时先忽略，就按照字面意思去理解。怎么创建的呢？

```go
func NewBlockchainReactor(state sm.State, blockExec *sm.BlockExecutor, store *BlockStore,
	fastSync bool) *BlockchainReactor {

	if state.LastBlockHeight != store.Height() {
		panic(fmt.Sprintf("state (%v) and store (%v) height mismatch", state.LastBlockHeight,
			store.Height()))
	}

	requestsCh := make(chan BlockRequest, maxTotalRequesters)

	const capacity = 1000                      // must be bigger than peers count
	errorsCh := make(chan peerError, capacity) // so we don't block in #Receive#pool.AddBlock

	pool := NewBlockPool(
		store.Height()+1,
		requestsCh,
		errorsCh,
	)

	bcR := &BlockchainReactor{
		initialState: state,
		blockExec:    blockExec,
		store:        store,
		pool:         pool,
		fastSync:     fastSync,
		requestsCh:   requestsCh,
		errorsCh:     errorsCh,
	}
	bcR.BaseReactor = *p2p.NewBaseReactor("BlockchainReactor", bcR)
	return bcR
}
```

检查区块高度是否一直，然后创建和blockpool通信的channel，然后还创建了和peer出错进行通信的channel。 怎么工作的呢？

```go
func (bcR *BlockchainReactor) OnStart() error {
	if bcR.fastSync {
		err := bcR.pool.Start()
		if err != nil {
			return err
		}
		go bcR.poolRoutine()
	}
	return nil
}
```

很简单，不是快同步的话就不执行了，然后启动下载池，再打开对接受信息处理的协程。

```go
func (bcR *BlockchainReactor) poolRoutine() {

	trySyncTicker := time.NewTicker(trySyncIntervalMS * time.Millisecond)
	statusUpdateTicker := time.NewTicker(statusUpdateIntervalSeconds * time.Second)
	switchToConsensusTicker := time.NewTicker(switchToConsensusIntervalSeconds * time.Second)

	blocksSynced := 0

	chainID := bcR.initialState.ChainID
	state := bcR.initialState

	lastHundred := time.Now()
	lastRate := 0.0

	didProcessCh := make(chan struct{}, 1)

FOR_LOOP:
	for {
		select {
		case request := <-bcR.requestsCh:
			peer := bcR.Switch.Peers().Get(request.PeerID)
			if peer == nil {
				continue FOR_LOOP // Peer has since been disconnected.
			}
			msgBytes := cdc.MustMarshalBinaryBare(&bcBlockRequestMessage{request.Height})
			queued := peer.TrySend(BlockchainChannel, msgBytes)
			if !queued {
				// We couldn't make the request, send-queue full.
				// The pool handles timeouts, just let it go.
				continue FOR_LOOP
			}

		case err := <-bcR.errorsCh:
			peer := bcR.Switch.Peers().Get(err.peerID)
			if peer != nil {
				bcR.Switch.StopPeerForError(peer, err)
			}

		case <-statusUpdateTicker.C:
			// ask for status updates
			go bcR.BroadcastStatusRequest() // nolint: errcheck

		case <-switchToConsensusTicker.C:
			height, numPending, lenRequesters := bcR.pool.GetStatus()
			outbound, inbound, _ := bcR.Switch.NumPeers()
			bcR.Logger.Debug("Consensus ticker", "numPending", numPending, "total", lenRequesters,
				"outbound", outbound, "inbound", inbound)
			if bcR.pool.IsCaughtUp() {
				bcR.Logger.Info("Time to switch to consensus reactor!", "height", height)
				bcR.pool.Stop()

				conR, ok := bcR.Switch.Reactor("CONSENSUS").(consensusReactor)
				if ok {
					conR.SwitchToConsensus(state, blocksSynced)
				} else {
					// should only happen during testing
				}

				break FOR_LOOP
			}

		case <-trySyncTicker.C: // chan time
			select {
			case didProcessCh <- struct{}{}:
			default:
			}

		case <-didProcessCh:
			// NOTE: It is a subtle mistake to process more than a single block
			// at a time (e.g. 10) here, because we only TrySend 1 request per
			// loop.  The ratio mismatch can result in starving of blocks, a
			// sudden burst of requests and responses, and repeat.
			// Consequently, it is better to split these routines rather than
			// coupling them as it's written here.  TODO uncouple from request
			// routine.

			// See if there are any blocks to sync.
			first, second := bcR.pool.PeekTwoBlocks()
			//bcR.Logger.Info("TrySync peeked", "first", first, "second", second)
			if first == nil || second == nil {
				// We need both to sync the first block.
				continue FOR_LOOP
			} else {
				// Try again quickly next loop.
				didProcessCh <- struct{}{}
			}

			firstParts := first.MakePartSet(types.BlockPartSizeBytes)
			firstPartsHeader := firstParts.Header()
			firstID := types.BlockID{first.Hash(), firstPartsHeader}
			// Finally, verify the first block using the second's commit
			// NOTE: we can probably make this more efficient, but note that calling
			// first.Hash() doesn't verify the tx contents, so MakePartSet() is
			// currently necessary.
			err := state.Validators.VerifyCommit(
				chainID, firstID, first.Height, second.LastCommit)
			if err != nil {
				bcR.Logger.Error("Error in validation", "err", err)
				peerID := bcR.pool.RedoRequest(first.Height)
				peer := bcR.Switch.Peers().Get(peerID)
				if peer != nil {
					// NOTE: we've already removed the peer's request, but we
					// still need to clean up the rest.
					bcR.Switch.StopPeerForError(peer, fmt.Errorf("BlockchainReactor validation error: %v", err))
				}
				peerID2 := bcR.pool.RedoRequest(second.Height)
				peer2 := bcR.Switch.Peers().Get(peerID2)
				if peer2 != nil && peer2 != peer {
					// NOTE: we've already removed the peer's request, but we
					// still need to clean up the rest.
					bcR.Switch.StopPeerForError(peer2, fmt.Errorf("BlockchainReactor validation error: %v", err))
				}
				continue FOR_LOOP
			} else {
				bcR.pool.PopRequest()

				// TODO: batch saves so we dont persist to disk every block
				bcR.store.SaveBlock(first, firstParts, second.LastCommit)

				// TODO: same thing for app - but we would need a way to
				// get the hash without persisting the state
				var err error
				state, err = bcR.blockExec.ApplyBlock(state, firstID, first)
				if err != nil {
					// TODO This is bad, are we zombie?
					cmn.PanicQ(fmt.Sprintf("Failed to process committed block (%d:%X): %v",
						first.Height, first.Hash(), err))
				}
				blocksSynced++

				if blocksSynced%100 == 0 {
					lastRate = 0.9*lastRate + 0.1*(100/time.Since(lastHundred).Seconds())
					bcR.Logger.Info("Fast Sync Rate", "height", bcR.pool.height,
						"max_peer_height", bcR.pool.MaxPeerHeight(), "blocks/s", lastRate)
					lastHundred = time.Now()
				}
			}
			continue FOR_LOOP

		case <-bcR.Quit():
			break FOR_LOOP
		}
	}
}
```

创建了几个定时器，主要是为了方便在后面进行时间判断。 首先时如果有别的peer的请求发送过来了，就获取下具体的peer，然后发送过去请求的区块内容。 接下来时如果有peer出错了，就停止这个peer吧。这个超时时间是从peer开始下载block开始算起的，正常情况下如果一下在，立马就有回应的。 接下来就是我们看到的定时器的工作了，包括获取别的peer的区块高度，是否达到最高块。 然后接下来的`case <-trySyncTicker.C`写的很好玩，每10ms出发一下，处理接收到的区块。不过为什么分开，是为了处理其他情况下的区块吧。 处理块这块是最主要的内容。首先是检查从peer那获取的区块是否正确，如果不正确，就得把这个peer移除，然后重新下载。如果正确的话，没问题了，可以处理区块了。 这些都是上层的处理，我们得看下具体信息过来的处理。

```go
func (bcR *BlockchainReactor) Receive(chID byte, src p2p.Peer, msgBytes []byte) {
	msg, err := decodeMsg(msgBytes)
	if err != nil {
		bcR.Logger.Error("Error decoding message", "src", src, "chId", chID, "msg", msg, "err", err, "bytes", msgBytes)
		bcR.Switch.StopPeerForError(src, err)
		return
	}

	if err = msg.ValidateBasic(); err != nil {
		bcR.Logger.Error("Peer sent us invalid msg", "peer", src, "msg", msg, "err", err)
		bcR.Switch.StopPeerForError(src, err)
		return
	}

	bcR.Logger.Debug("Receive", "src", src, "chID", chID, "msg", msg)

	switch msg := msg.(type) {
	case *bcBlockRequestMessage:
		if queued := bcR.respondToPeer(msg, src); !queued {
			// Unfortunately not queued since the queue is full.
		}
	case *bcBlockResponseMessage:
		bcR.pool.AddBlock(src.ID(), msg.Block, len(msgBytes))
	case *bcStatusRequestMessage:
		// Send peer our state.
		msgBytes := cdc.MustMarshalBinaryBare(&bcStatusResponseMessage{bcR.store.Height()})
		queued := src.TrySend(BlockchainChannel, msgBytes)
		if !queued {
			// sorry
		}
	case *bcStatusResponseMessage:
		// Got a peer status. Unverified.
		bcR.pool.SetPeerHeight(src.ID(), msg.Height)
	default:
		bcR.Logger.Error(fmt.Sprintf("Unknown message type %v", reflect.TypeOf(msg)))
	}
}
```

就像我们上面说的，给别人发送了消息，我们肯定也会收到消息，这里面就是接受到对消息的处理。 整体的流程就是首先获取别的peer的最新区块高度，然后为每一个区块高度分配一个peer，去找这个peer下载对应高度的区块。 然后每一个区块高度对应一个下载器，下载好了进行验证，然后存储，共识，结束。

## State

我们在上一篇文章中提到，block处理有处理state，以及相应的执行过程。我们这一节就来看一看。

我们先看一下State的定义

```go
type State struct {
	Version Version

	// immutable
	ChainID string

	// LastBlockHeight=0 at genesis (ie. block(H=0) does not exist)
	LastBlockHeight  int64
	LastBlockTotalTx int64
	LastBlockID      types.BlockID
	LastBlockTime    time.Time

	// LastValidators is used to validate block.LastCommit.
	// Validators are persisted to the database separately every time they change,
	// so we can query for historical validator sets.
	// Note that if s.LastBlockHeight causes a valset change,
	// we set s.LastHeightValidatorsChanged = s.LastBlockHeight + 1 + 1
	// Extra +1 due to nextValSet delay.
	NextValidators              *types.ValidatorSet
	Validators                  *types.ValidatorSet
	LastValidators              *types.ValidatorSet
	LastHeightValidatorsChanged int64

	// Consensus parameters used for validating blocks.
	// Changes returned by EndBlock and updated after Commit.
	ConsensusParams                  types.ConsensusParams
	LastHeightConsensusParamsChanged int64

	// Merkle root of the results from executing prev block
	LastResultsHash []byte

	// the latest AppHash we've received from calling abci.Commit()
	AppHash []byte
}
```

看这个结构，也可以发现State是一个记录上一个区块链的状态集合。包括上一个已经提交区块的高度，交易信息，上一个区块的时间和ID。然后是一系列的验证节点集合，包括下一次的验证节点集合，所有的验证节点以及上一个区块的验证节点。 根据要求，State的状态是不能直接修改的，要使用得自己Copy出去，或者用NextState生成一个新的状态。 这些都是一些基本的，重要的是生成区块，我们看下是怎么封装成区块的

```go
func (state State) MakeBlock(
	height int64,
	txs []types.Tx,
	commit *types.Commit,
	evidence []types.Evidence,
	proposerAddress []byte,
) (*types.Block, *types.PartSet) {

	// Build base block with block data.
	block := types.MakeBlock(height, txs, commit, evidence)

	// Set time.
	var timestamp time.Time
	if height == 1 {
		timestamp = state.LastBlockTime // genesis time
	} else {
		timestamp = MedianTime(commit, state.LastValidators)
	}

	// Fill rest of header with state data.
	block.Header.Populate(
		state.Version.Consensus, state.ChainID,
		timestamp, state.LastBlockID, state.LastBlockTotalTx+block.NumTxs,
		state.Validators.Hash(), state.NextValidators.Hash(),
		state.ConsensusParams.Hash(), state.AppHash, state.LastResultsHash,
		proposerAddress,
	)

	return block, block.MakePartSet(types.BlockPartSizeBytes)
}
```

这个看着还好理解，就是把区块的内容封装起来，生成一个完整的区块信息。这个地方使用了一个中值时间，主要的目标就是防止有恶意节点估计篡改区块生成时间，因为是所有的中间值，在出现之前很难修改成功。 我们再来看下block是怎么被执行的。

```go
type BlockExecutor struct {
	// save state, validators, consensus params, abci responses here
	db dbm.DB

	// execute the app against this
	proxyApp proxy.AppConnConsensus

	// events
	eventBus types.BlockEventPublisher

	// update these with block results after commit
	mempool Mempool
	evpool  EvidencePool

	logger log.Logger

	metrics *Metrics
}
```

没什么大问题，存储状态，更新内存池，和ABCI程序通信。 怎么执行的呢？

```go
func (blockExec *BlockExecutor) ApplyBlock(state State, blockID types.BlockID, block *types.Block) (State, error) {
    // 验证区块是否正确
	if err := blockExec.ValidateBlock(state, block); err != nil {
		return state, ErrInvalidBlock(err)
	}
    // 让ABCI程序验证
	startTime := time.Now().UnixNano()
	abciResponses, err := execBlockOnProxyApp(blockExec.logger, blockExec.proxyApp, block, state.LastValidators, blockExec.db)
	endTime := time.Now().UnixNano()
	blockExec.metrics.BlockProcessingTime.Observe(float64(endTime-startTime) / 1000000)
	if err != nil {
		return state, ErrProxyAppConn(err)
	}

	fail.Fail() // XXX

	// 把返回的结果记录到数据库
	saveABCIResponses(blockExec.db, block.Height, abciResponses)

	fail.Fail() // XXX

	// 更新验证节点集合
	abciValUpdates := abciResponses.EndBlock.ValidatorUpdates
	err = validateValidatorUpdates(abciValUpdates, state.ConsensusParams.Validator)
	if err != nil {
		return state, fmt.Errorf("Error in validator updates: %v", err)
	}
	validatorUpdates, err := types.PB2TM.ValidatorUpdates(abciValUpdates)
	if err != nil {
		return state, err
	}
	if len(validatorUpdates) > 0 {
		blockExec.logger.Info("Updates to validators", "updates", makeValidatorUpdatesLogString(validatorUpdates))
	}

	// 根据验证的结果，获取写一个区块链状态
	state, err = updateState(state, blockID, &block.Header, abciResponses, validatorUpdates)
	if err != nil {
		return state, fmt.Errorf("Commit failed for application: %v", err)
	}

	// 调用ABCI程序，获取APPHASH，并且更新下内存池
	appHash, err := blockExec.Commit(state, block)
	if err != nil {
		return state, fmt.Errorf("Commit failed for application: %v", err)
	}

	// 更新证据池
	blockExec.evpool.Update(block, state)

	fail.Fail() // XXX

	// 保存状态
	state.AppHash = appHash
	SaveState(blockExec.db, state)

	fail.Fail() // XXX

	// Events are fired after everything else.
	// NOTE: if we crash between Commit and Save, events wont be fired during replay
	fireEvents(blockExec.logger, blockExec.eventBus, block, abciResponses, validatorUpdates)

	return state, nil
}
```

看着我们代码里面的注释，还能够理解出来，主要的流程就是 1. 验证当前区块是否合法 2. 把数据提交给ABCI,获取对应的响应结果 3. 根绝当前区块信息还有ABCI的结果，生成新的区块 4. 调用下ABCI的commit返回APPHASH,这时候才算要保存了 5. 保存状态，生成下一个状态 怎么验证的呢？

```go
func validateBlock(stateDB dbm.DB, state State, block *types.Block) error {
	// Validate internal consistency.
	if err := block.ValidateBasic(); err != nil {
		return err
	}

	// Validate basic info.
	if block.Version != state.Version.Consensus {
		return fmt.Errorf("Wrong Block.Header.Version. Expected %v, got %v",
			state.Version.Consensus,
			block.Version,
		)
	}
	if block.ChainID != state.ChainID {
		return fmt.Errorf("Wrong Block.Header.ChainID. Expected %v, got %v",
			state.ChainID,
			block.ChainID,
		)
	}
	if block.Height != state.LastBlockHeight+1 {
		return fmt.Errorf("Wrong Block.Header.Height. Expected %v, got %v",
			state.LastBlockHeight+1,
			block.Height,
		)
	}

	// Validate prev block info.
	if !block.LastBlockID.Equals(state.LastBlockID) {
		return fmt.Errorf("Wrong Block.Header.LastBlockID.  Expected %v, got %v",
			state.LastBlockID,
			block.LastBlockID,
		)
	}

	newTxs := int64(len(block.Data.Txs))
	if block.TotalTxs != state.LastBlockTotalTx+newTxs {
		return fmt.Errorf("Wrong Block.Header.TotalTxs. Expected %v, got %v",
			state.LastBlockTotalTx+newTxs,
			block.TotalTxs,
		)
	}

	// Validate app info
	if !bytes.Equal(block.AppHash, state.AppHash) {
		return fmt.Errorf("Wrong Block.Header.AppHash.  Expected %X, got %v",
			state.AppHash,
			block.AppHash,
		)
	}
	if !bytes.Equal(block.ConsensusHash, state.ConsensusParams.Hash()) {
		return fmt.Errorf("Wrong Block.Header.ConsensusHash.  Expected %X, got %v",
			state.ConsensusParams.Hash(),
			block.ConsensusHash,
		)
	}
	if !bytes.Equal(block.LastResultsHash, state.LastResultsHash) {
		return fmt.Errorf("Wrong Block.Header.LastResultsHash.  Expected %X, got %v",
			state.LastResultsHash,
			block.LastResultsHash,
		)
	}
	if !bytes.Equal(block.ValidatorsHash, state.Validators.Hash()) {
		return fmt.Errorf("Wrong Block.Header.ValidatorsHash.  Expected %X, got %v",
			state.Validators.Hash(),
			block.ValidatorsHash,
		)
	}
	if !bytes.Equal(block.NextValidatorsHash, state.NextValidators.Hash()) {
		return fmt.Errorf("Wrong Block.Header.NextValidatorsHash.  Expected %X, got %v",
			state.NextValidators.Hash(),
			block.NextValidatorsHash,
		)
	}

	// Validate block LastCommit.
	if block.Height == 1 {
		if len(block.LastCommit.Precommits) != 0 {
			return errors.New("Block at height 1 can't have LastCommit precommits")
		}
	} else {
		if len(block.LastCommit.Precommits) != state.LastValidators.Size() {
			return fmt.Errorf("Invalid block commit size. Expected %v, got %v",
				state.LastValidators.Size(),
				len(block.LastCommit.Precommits),
			)
        }
        // 此处和其他人理解有点不一致。我认为是用状态记录的区块验证集合，跟区块中的
        // 验证集合进行对比。而不是用这个的信息去验证上一个区块。
		err := state.LastValidators.VerifyCommit(
			state.ChainID, state.LastBlockID, block.Height-1, block.LastCommit)
		if err != nil {
			return err
		}
	}

	// Validate block Time
	if block.Height > 1 {
		if !block.Time.After(state.LastBlockTime) {
			return fmt.Errorf("Block time %v not greater than last block time %v",
				block.Time,
				state.LastBlockTime,
			)
		}

		medianTime := MedianTime(block.LastCommit, state.LastValidators)
		if !block.Time.Equal(medianTime) {
			return fmt.Errorf("Invalid block time. Expected %v, got %v",
				medianTime,
				block.Time,
			)
		}
	} else if block.Height == 1 {
		genesisTime := state.LastBlockTime
		if !block.Time.Equal(genesisTime) {
			return fmt.Errorf("Block time %v is not equal to genesis time %v",
				block.Time,
				genesisTime,
			)
		}
	}

	// Limit the amount of evidence
	maxEvidenceBytes := types.MaxEvidenceBytesPerBlock(state.ConsensusParams.BlockSize.MaxBytes)
	evidenceBytes := int64(len(block.Evidence.Evidence)) * types.MaxEvidenceBytes
	if evidenceBytes > maxEvidenceBytes {
		return types.NewErrEvidenceOverflow(maxEvidenceBytes, evidenceBytes)
	}

	// Validate all evidence.
	for _, ev := range block.Evidence.Evidence {
		if err := VerifyEvidence(stateDB, state, ev); err != nil {
			return types.NewErrEvidenceInvalid(ev, err)
		}
	}

	// NOTE: We can't actually verify it's the right proposer because we dont
	// know what round the block was first proposed. So just check that it's
	// a legit address and a known validator.
	if len(block.ProposerAddress) != crypto.AddressSize ||
		!state.Validators.HasAddress(block.ProposerAddress) {
		return fmt.Errorf("Block.Header.ProposerAddress, %X, is not a validator",
			block.ProposerAddress,
		)
	}

	return nil
}
```

主要的就是验证区块的各块信息是否正确，能否达成一致。不一致就返回错误信息。 接下来我们要看一下是怎么传给ABCI程序执行的

```go
func execBlockOnProxyApp(
	logger log.Logger,
	proxyAppConn proxy.AppConnConsensus,
	block *types.Block,
	lastValSet *types.ValidatorSet,
	stateDB dbm.DB,
) (*ABCIResponses, error) {
	var validTxs, invalidTxs = 0, 0

	txIndex := 0
	abciResponses := NewABCIResponses(block)

	// Execute transactions and get hash.
	proxyCb := func(req *abci.Request, res *abci.Response) {
		switch r := res.Value.(type) {
		case *abci.Response_DeliverTx:
			// TODO: make use of res.Log
			// TODO: make use of this info
			// Blocks may include invalid txs.
			txRes := r.DeliverTx
			if txRes.Code == abci.CodeTypeOK {
				validTxs++
			} else {
				logger.Debug("Invalid tx", "code", txRes.Code, "log", txRes.Log)
				invalidTxs++
			}
			abciResponses.DeliverTx[txIndex] = txRes
			txIndex++
		}
	}
	proxyAppConn.SetResponseCallback(proxyCb)

	commitInfo, byzVals := getBeginBlockValidatorInfo(block, lastValSet, stateDB)

	// Begin block
	var err error
	abciResponses.BeginBlock, err = proxyAppConn.BeginBlockSync(abci.RequestBeginBlock{
		Hash:                block.Hash(),
		Header:              types.TM2PB.Header(&block.Header),
		LastCommitInfo:      commitInfo,
		ByzantineValidators: byzVals,
	})
	if err != nil {
		logger.Error("Error in proxyAppConn.BeginBlock", "err", err)
		return nil, err
	}

	// Run txs of block.
	for _, tx := range block.Txs {
		proxyAppConn.DeliverTxAsync(tx)
		if err := proxyAppConn.Error(); err != nil {
			return nil, err
		}
	}

	// End block.
	abciResponses.EndBlock, err = proxyAppConn.EndBlockSync(abci.RequestEndBlock{Height: block.Height})
	if err != nil {
		logger.Error("Error in proxyAppConn.EndBlock", "err", err)
		return nil, err
	}

	logger.Info("Executed block", "height", block.Height, "validTxs", validTxs, "invalidTxs", invalidTxs)

	return abciResponses, nil
}
```

整体的流程是，首先通知abci程序，需要进行生成区块了，等待ABCI程序的结果。所以，我们如果了解ABCI的话，我们会发现，我们可以自己实现BeginBlock的操作，返回结果，不过，官方例都没有做。 接下来是把TX一个一个的发给ABCI，这个官方历程都做了，最后是EndBlock，嗯，没做示例。 到这个地方，和abci的通信已经做完了，接下来就是更新自己的状态内容了。跟ABCI没关系了。 接下来是updateState

```go
func updateState(
	state State,
	blockID types.BlockID,
	header *types.Header,
	abciResponses *ABCIResponses,
	validatorUpdates []*types.Validator,
) (State, error) {

	// Copy the valset so we can apply changes from EndBlock
	// and update s.LastValidators and s.Validators.
	nValSet := state.NextValidators.Copy()

	// Update the validator set with the latest abciResponses.
	lastHeightValsChanged := state.LastHeightValidatorsChanged
	if len(validatorUpdates) > 0 {
		err := updateValidators(nValSet, validatorUpdates)
		if err != nil {
			return state, fmt.Errorf("Error changing validator set: %v", err)
		}
		// Change results from this height but only applies to the next next height.
		lastHeightValsChanged = header.Height + 1 + 1
	}

	// Update validator proposer priority and set state variables.
	nValSet.IncrementProposerPriority(1)

	// Update the params with the latest abciResponses.
	nextParams := state.ConsensusParams
	lastHeightParamsChanged := state.LastHeightConsensusParamsChanged
	if abciResponses.EndBlock.ConsensusParamUpdates != nil {
		// NOTE: must not mutate s.ConsensusParams
		nextParams = state.ConsensusParams.Update(abciResponses.EndBlock.ConsensusParamUpdates)
		err := nextParams.Validate()
		if err != nil {
			return state, fmt.Errorf("Error updating consensus params: %v", err)
		}
		// Change results from this height but only applies to the next height.
		lastHeightParamsChanged = header.Height + 1
	}

	// TODO: allow app to upgrade version
	nextVersion := state.Version

	// NOTE: the AppHash has not been populated.
	// It will be filled on state.Save.
	return State{
		Version:                          nextVersion,
		ChainID:                          state.ChainID,
		LastBlockHeight:                  header.Height,
		LastBlockTotalTx:                 state.LastBlockTotalTx + header.NumTxs,
		LastBlockID:                      blockID,
		LastBlockTime:                    header.Time,
		NextValidators:                   nValSet,
		Validators:                       state.NextValidators.Copy(),
		LastValidators:                   state.Validators.Copy(),
		LastHeightValidatorsChanged:      lastHeightValsChanged,
		ConsensusParams:                  nextParams,
		LastHeightConsensusParamsChanged: lastHeightParamsChanged,
		LastResultsHash:                  abciResponses.ResultsHash(),
		AppHash:                          nil,
	}, nil
}
```

首先是更新验证节点集合，更新策略就是这样的 如果当前不存在则直接加入一个验证者 如果当前存在并且投票权为0则删除 如果当前存在其投票权不为0则更新 然后更新共识参数，生成state，但是，这个时候还没有APPHASH呢，因为还没有调用完。 然后执行commit

```go
func (blockExec *BlockExecutor) Commit(
	state State,
	block *types.Block,
) ([]byte, error) {
	blockExec.mempool.Lock()
	defer blockExec.mempool.Unlock()

	// while mempool is Locked, flush to ensure all async requests have completed
	// in the ABCI app before Commit.
	err := blockExec.mempool.FlushAppConn()
	if err != nil {
		blockExec.logger.Error("Client error during mempool.FlushAppConn", "err", err)
		return nil, err
	}

	// Commit block, get hash back
	res, err := blockExec.proxyApp.CommitSync()
	if err != nil {
		blockExec.logger.Error(
			"Client error during proxyAppConn.CommitSync",
			"err", err,
		)
		return nil, err
	}
	// ResponseCommit has no error code - just data

	blockExec.logger.Info(
		"Committed state",
		"height", block.Height,
		"txs", block.NumTxs,
		"appHash", fmt.Sprintf("%X", res.Data),
	)

	// Update mempool.
	err = blockExec.mempool.Update(
		block.Height,
		block.Txs,
		TxPreCheck(state),
		TxPostCheck(state),
	)

	return res.Data, err
}
```

不复杂，就是通知ABCI程序要commit到数据库了，调用了ABCI程序的COMMIT函数。然后从mempool中移除本次提交的交易。 最后一步，把状态写入区块。

```go
func saveState(db dbm.DB, state State, key []byte) {
	nextHeight := state.LastBlockHeight + 1
	// If first block, save validators for block 1.
	if nextHeight == 1 {
		// This extra logic due to Tendermint validator set changes being delayed 1 block.
		// It may get overwritten due to InitChain validator updates.
		lastHeightVoteChanged := int64(1)
		saveValidatorsInfo(db, nextHeight, lastHeightVoteChanged, state.Validators)
	}
	// Save next validators.
	saveValidatorsInfo(db, nextHeight+1, state.LastHeightValidatorsChanged, state.NextValidators)
	// Save next consensus params.
	saveConsensusParamsInfo(db, nextHeight, state.LastHeightConsensusParamsChanged, state.ConsensusParams)
	db.SetSync(key, state.Bytes())
}
```

我们可以看明白了，通过这个流程，最最最最主要的就是ApplyBlock程序了。 这几个环节我们中间也进行了分析，再写ABCI程序也更清晰了。

## EvidencePool

我们现在要啃一个硬骨头了，共识模块。在开始之前，我们先看一下EvidencePool模块，一个合法证据池模块。

先看一下证据池

```go
type EvidencePool struct {
	logger log.Logger

	evidenceStore *EvidenceStore
	evidenceList  *clist.CList // concurrent linked-list of evidence

	// needed to load validators to verify evidence
	stateDB dbm.DB

	// latest state
	mtx   sync.Mutex
	state sm.State
}
```

跟我们自己想的结果差不多，需要验证节点集合验证证据，存到证据库里面。 新建函数很简单。这个证据池的函数也不是太多，主要就几个，我们来主要关注下添加证据和移除证据。

```go
func (evpool *EvidencePool) AddEvidence(evidence types.Evidence) (err error) {

	// TODO: check if we already have evidence for this
	// validator at this height so we dont get spammed

	if err := sm.VerifyEvidence(evpool.stateDB, evpool.State(), evidence); err != nil {
		return err
	}

	// fetch the validator and return its voting power as its priority
	// TODO: something better ?
	valset, _ := sm.LoadValidators(evpool.stateDB, evidence.Height())
	_, val := valset.GetByAddress(evidence.Address())
	priority := val.VotingPower

	added := evpool.evidenceStore.AddNewEvidence(evidence, priority)
	if !added {
		// evidence already known, just ignore
		return
	}

	evpool.logger.Info("Verified new evidence of byzantine behaviour", "evidence", evidence)

	// add evidence to clist
	evpool.evidenceList.PushBack(evidence)

	return nil
}
```

首先是验证证据是否正确，回到了我们之前看到的State模块

```go
func VerifyEvidence(stateDB dbm.DB, state State, evidence types.Evidence) error {
	height := state.LastBlockHeight

	evidenceAge := height - evidence.Height()
	maxAge := state.ConsensusParams.Evidence.MaxAge
	if evidenceAge > maxAge {
		return fmt.Errorf("Evidence from height %d is too old. Min height is %d",
			evidence.Height(), height-maxAge)
	}

	valset, err := LoadValidators(stateDB, evidence.Height())
	if err != nil {
		// TODO: if err is just that we cant find it cuz we pruned, ignore.
		// TODO: if its actually bad evidence, punish peer
		return err
	}

	// The address must have been an active validator at the height.
	// NOTE: we will ignore evidence from H if the key was not a validator
	// at H, even if it is a validator at some nearby H'
	ev := evidence
	height, addr := ev.Height(), ev.Address()
	_, val := valset.GetByAddress(addr)
	if val == nil {
		return fmt.Errorf("Address %X was not a validator at height %d", addr, height)
	}

	if err := evidence.Verify(state.ChainID, val.PubKey); err != nil {
		return err
	}

	return nil
}
```

首先保证证据足够的新，不是很旧的节点产生的同意提案，然后这个证据是针对本区块高度，一个合理的验证者产生的 验证通过之后，就需要存到数据库中，以及添加到证据库中。 怎么移除的呢？

```go
func (evpool *EvidencePool) removeEvidence(height, maxAge int64, blockEvidenceMap map[string]struct{}) {
	for e := evpool.evidenceList.Front(); e != nil; e = e.Next() {
		ev := e.Value.(types.Evidence)

		// Remove the evidence if it's already in a block
		// or if it's now too old.
		if _, ok := blockEvidenceMap[evMapKey(ev)]; ok ||
			ev.Height() < height-maxAge {

			// remove from clist
			evpool.evidenceList.Remove(e)
			e.DetachPrev()
		}
	}
}
```

就是如果区块已经被提交，或者这个证据太旧了，就移除就可以了。但是，注意，只从证据库里面移除，不从数据库中移除。 evidencestore是和数据库打交道的，提供了各种所需要的功能，我们就不进去细看了。 最后，我们再来看下reactor。

```go
type EvidenceReactor struct {
	p2p.BaseReactor
	evpool   *EvidencePool
	eventBus *types.EventBus
}
```

这个reacotr就是处理证据池的证据的，再peer之间广播，把证据推给证据池，来进行处理。 这个类没有实现start，我们就先看一看Receive函数吧。

```go
func (evR *EvidenceReactor) Receive(chID byte, src p2p.Peer, msgBytes []byte) {
	msg, err := decodeMsg(msgBytes)
	if err != nil {
		evR.Logger.Error("Error decoding message", "src", src, "chId", chID, "msg", msg, "err", err, "bytes", msgBytes)
		evR.Switch.StopPeerForError(src, err)
		return
	}

	if err = msg.ValidateBasic(); err != nil {
		evR.Logger.Error("Peer sent us invalid msg", "peer", src, "msg", msg, "err", err)
		evR.Switch.StopPeerForError(src, err)
		return
	}

	evR.Logger.Debug("Receive", "src", src, "chId", chID, "msg", msg)

	switch msg := msg.(type) {
	case *EvidenceListMessage:
		for _, ev := range msg.Evidence {
			err := evR.evpool.AddEvidence(ev)
			if err != nil {
				evR.Logger.Info("Evidence is not valid", "evidence", msg.Evidence, "err", err)
				// punish peer
				evR.Switch.StopPeerForError(src, err)
			}
		}
	default:
		evR.Logger.Error(fmt.Sprintf("Unknown message type %v", reflect.TypeOf(msg)))
	}
}
```

首先验证消息是否正确，正确的话就添加到证据库中。否则有问题就停止发过来证据的peer。 我们再看一个函数，addPeer,因为这个会在其他的地方被调用到。

```go
func (evR *EvidenceReactor) broadcastEvidenceRoutine(peer p2p.Peer) {
	var next *clist.CElement
	for {
		// This happens because the CElement we were looking at got garbage
		// collected (removed). That is, .NextWait() returned nil. Go ahead and
		// start from the beginning.
		if next == nil {
			select {
			case <-evR.evpool.EvidenceWaitChan(): // Wait until evidence is available
				if next = evR.evpool.EvidenceFront(); next == nil {
					continue
				}
			case <-peer.Quit():
				return
			case <-evR.Quit():
				return
			}
		}

		ev := next.Value.(types.Evidence)
		msg, retry := evR.checkSendEvidenceMessage(peer, ev)
		if msg != nil {
			success := peer.Send(EvidenceChannel, cdc.MustMarshalBinaryBare(msg))
			retry = !success
		}

		if retry {
			time.Sleep(peerCatchupSleepIntervalMS * time.Millisecond)
			continue
		}

		afterCh := time.After(time.Second * broadcastEvidenceIntervalS)
		select {
		case <-afterCh:
			// start from the beginning every tick.
			// TODO: only do this if we're at the end of the list!
			next = nil
		case <-next.NextWaitChan():
			// see the start of the for loop for nil check
			next = next.Next()
		case <-peer.Quit():
			return
		case <-evR.Quit():
			return
		}
	}
}
```

可以看到，添加一个peer后，只要自己有证据，就会发送给其他人。 这个evidence模块感觉还不是太复杂，因此，我们就等着看consensus模块吧。

## 启动流程

我们现在来看一下Tendermint的工作流程，接上我们第一篇文章了。

为了方便说，我们再重新把NewNode拿过来，看一遍

```go
func NewNode(config *cfg.Config,
	privValidator types.PrivValidator,
	nodeKey *p2p.NodeKey,
	clientCreator proxy.ClientCreator,
	genesisDocProvider GenesisDocProvider,
	dbProvider DBProvider,
	metricsProvider MetricsProvider,
	logger log.Logger) (*Node, error) {

	// 获取我们一个区块存储的对象，我们在blockchain模块提到了这个功能
	blockStoreDB, err := dbProvider(&DBContext{"blockstore", config})
	if err != nil {
		return nil, err
	}
	blockStore := bc.NewBlockStore(blockStoreDB)

	// 系统状态的数据库
	stateDB, err := dbProvider(&DBContext{"state", config})
	if err != nil {
		return nil, err
	}

	// 默认的创始区块信息
	genDoc, err := loadGenesisDoc(stateDB)
	if err != nil {
		genDoc, err = genesisDocProvider()
		if err != nil {
			return nil, err
		}
		// save genesis doc to prevent a certain class of user errors (e.g. when it
		// was changed, accidentally or not). Also good for audit trail.
		saveGenesisDoc(stateDB, genDoc)
	}

    // 读取系统状态，这个我们在state模块提到了本功能
	state, err := sm.LoadStateFromDBOrGenesisDoc(stateDB, genDoc)
	if err != nil {
		return nil, err
	}

	// 和abci通信的程序
	proxyApp := proxy.NewAppConns(clientCreator)
	proxyApp.SetLogger(logger.With("module", "proxy"))
	if err := proxyApp.Start(); err != nil {
		return nil, fmt.Errorf("Error starting proxy app connections: %v", err)
	}

	// 会在proxyApp.start中调用handshake，进行区块重放进行APPHASH的校验
	consensusLogger := logger.With("module", "consensus")
	handshaker := cs.NewHandshaker(stateDB, state, blockStore, genDoc)
	handshaker.SetLogger(consensusLogger)
	if err := handshaker.Handshake(proxyApp); err != nil {
		return nil, fmt.Errorf("Error during handshake: %v", err)
	}

	// 重新加载一下区块的状态
	state = sm.LoadState(stateDB)

	// Log the version info.
	logger.Info("Version info",
		"software", version.TMCoreSemVer,
		"block", version.BlockProtocol,
		"p2p", version.P2PProtocol,
	)

	// If the state and software differ in block version, at least log it.
	if state.Version.Consensus.Block != version.BlockProtocol {
		logger.Info("Software and state have different block protocols",
			"software", version.BlockProtocol,
			"state", state.Version.Consensus.Block,
		)
	}

	// 创建一个验证角色，并且和其他的validator能够通信
	if config.PrivValidatorListenAddr != "" {
		// If an address is provided, listen on the socket for a connection from an
		// external signing process.
		// FIXME: we should start services inside OnStart
		privValidator, err = createAndStartPrivValidatorSocketClient(config.PrivValidatorListenAddr, logger)
		if err != nil {
			return nil, errors.Wrap(err, "Error with private validator socket client")
		}
	}

	// Decide whether to fast-sync or not
	// We don't fast-sync when the only validator is us.
	fastSync := config.FastSync
	if state.Validators.Size() == 1 {
		addr, _ := state.Validators.GetByIndex(0)
		if bytes.Equal(privValidator.GetAddress(), addr) {
			fastSync = false
		}
	}

	// Log whether this node is a validator or an observer
	if state.Validators.HasAddress(privValidator.GetAddress()) {
		consensusLogger.Info("This node is a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
	} else {
		consensusLogger.Info("This node is not a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
	}

	csMetrics, p2pMetrics, memplMetrics, smMetrics := metricsProvider()

	// 创建内存池
	mempool := mempl.NewMempool(
		config.Mempool,
		proxyApp.Mempool(),
		state.LastBlockHeight,
		mempl.WithMetrics(memplMetrics),
		mempl.WithPreCheck(sm.TxPreCheck(state)),
		mempl.WithPostCheck(sm.TxPostCheck(state)),
	)
	mempoolLogger := logger.With("module", "mempool")
	mempool.SetLogger(mempoolLogger)
	if config.Mempool.WalEnabled() {
		mempool.InitWAL() // no need to have the mempool wal during tests
	}
	mempoolReactor := mempl.NewMempoolReactor(config.Mempool, mempool)
	mempoolReactor.SetLogger(mempoolLogger)

	if config.Consensus.WaitForTxs() {
		mempool.EnableTxsAvailable()
	}

	// 创建证据Reactor
	evidenceDB, err := dbProvider(&DBContext{"evidence", config})
	if err != nil {
		return nil, err
	}
	evidenceLogger := logger.With("module", "evidence")
	evidenceStore := evidence.NewEvidenceStore(evidenceDB)
	evidencePool := evidence.NewEvidencePool(stateDB, evidenceStore)
	evidencePool.SetLogger(evidenceLogger)
	evidenceReactor := evidence.NewEvidenceReactor(evidencePool)
	evidenceReactor.SetLogger(evidenceLogger)

	blockExecLogger := logger.With("module", "state")
	// make block executor for consensus and blockchain reactors to execute blocks
	blockExec := sm.NewBlockExecutor(
		stateDB,
		blockExecLogger,
		proxyApp.Consensus(),
		mempool,
		evidencePool,
		sm.BlockExecutorWithMetrics(smMetrics),
	)

	// 方便后面我们的ApplyBlock
	bcReactor := bc.NewBlockchainReactor(state.Copy(), blockExec, blockStore, fastSync)
	bcReactor.SetLogger(logger.With("module", "blockchain"))

	// 创建共识的reactor
	consensusState := cs.NewConsensusState(
		config.Consensus,
		state.Copy(),
		blockExec,
		blockStore,
		mempool,
		evidencePool,
		cs.StateMetrics(csMetrics),
	)
	consensusState.SetLogger(consensusLogger)
	if privValidator != nil {
		consensusState.SetPrivValidator(privValidator)
	}
	consensusReactor := cs.NewConsensusReactor(consensusState, fastSync, cs.ReactorMetrics(csMetrics))
	consensusReactor.SetLogger(consensusLogger)

	eventBus := types.NewEventBus()
	eventBus.SetLogger(logger.With("module", "events"))

	// services which will be publishing and/or subscribing for messages (events)
	// consensusReactor will set it on consensusState and blockExecutor
	consensusReactor.SetEventBus(eventBus)

	// 交易索引的服务
	var txIndexer txindex.TxIndexer
	switch config.TxIndex.Indexer {
	case "kv":
		store, err := dbProvider(&DBContext{"tx_index", config})
		if err != nil {
			return nil, err
		}
		if config.TxIndex.IndexTags != "" {
			txIndexer = kv.NewTxIndex(store, kv.IndexTags(splitAndTrimEmpty(config.TxIndex.IndexTags, ",", " ")))
		} else if config.TxIndex.IndexAllTags {
			txIndexer = kv.NewTxIndex(store, kv.IndexAllTags())
		} else {
			txIndexer = kv.NewTxIndex(store)
		}
	default:
		txIndexer = &null.TxIndex{}
	}

	indexerService := txindex.NewIndexerService(txIndexer, eventBus)
	indexerService.SetLogger(logger.With("module", "txindex"))

	p2pLogger := logger.With("module", "p2p")
	nodeInfo, err := makeNodeInfo(
		config,
		nodeKey.ID(),
		txIndexer,
		genDoc.ChainID,
		p2p.NewProtocolVersion(
			version.P2PProtocol, // global
			state.Version.Consensus.Block,
			state.Version.Consensus.App,
		),
	)
	if err != nil {
		return nil, err
	}

	// Setup Transport.
	var (
		mConnConfig = p2p.MConnConfig(config.P2P)
		transport   = p2p.NewMultiplexTransport(nodeInfo, *nodeKey, mConnConfig)
		connFilters = []p2p.ConnFilterFunc{}
		peerFilters = []p2p.PeerFilterFunc{}
	)

	if !config.P2P.AllowDuplicateIP {
		connFilters = append(connFilters, p2p.ConnDuplicateIPFilter())
	}

	// Filter peers by addr or pubkey with an ABCI query.
	// If the query return code is OK, add peer.
	if config.FilterPeers {
		connFilters = append(
			connFilters,
			// ABCI query for address filtering.
			func(_ p2p.ConnSet, c net.Conn, _ []net.IP) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/addr/%s", c.RemoteAddr().String()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)

		peerFilters = append(
			peerFilters,
			// ABCI query for ID filtering.
			func(_ p2p.IPeerSet, p p2p.Peer) error {
				res, err := proxyApp.Query().QuerySync(abci.RequestQuery{
					Path: fmt.Sprintf("/p2p/filter/id/%s", p.ID()),
				})
				if err != nil {
					return err
				}
				if res.IsErr() {
					return fmt.Errorf("Error querying abci app: %v", res)
				}

				return nil
			},
		)
	}

	p2p.MultiplexTransportConnFilters(connFilters...)(transport)

	// Setup Switch.
	sw := p2p.NewSwitch(
		config.P2P,
		transport,
		p2p.WithMetrics(p2pMetrics),
		p2p.SwitchPeerFilters(peerFilters...),
	)
	sw.SetLogger(p2pLogger)
	// 把各种组件都添加进去，之前的模块都在这集合起来了
	sw.AddReactor("MEMPOOL", mempoolReactor)
	sw.AddReactor("BLOCKCHAIN", bcReactor)
	sw.AddReactor("CONSENSUS", consensusReactor)
	sw.AddReactor("EVIDENCE", evidenceReactor)
	sw.SetNodeInfo(nodeInfo)
	sw.SetNodeKey(nodeKey)

	p2pLogger.Info("P2P Node ID", "ID", nodeKey.ID(), "file", config.NodeKeyFile())

	// 地址信息，方便节点连接其他节点，发现其他节点
	addrBook := pex.NewAddrBook(config.P2P.AddrBookFile(), config.P2P.AddrBookStrict)

	// Add ourselves to addrbook to prevent dialing ourselves
	addrBook.AddOurAddress(nodeInfo.NetAddress())

	addrBook.SetLogger(p2pLogger.With("book", config.P2P.AddrBookFile()))
	if config.P2P.PexReactor {
		// TODO persistent peers ? so we can have their DNS addrs saved
		pexReactor := pex.NewPEXReactor(addrBook,
			&pex.PEXReactorConfig{
				Seeds:    splitAndTrimEmpty(config.P2P.Seeds, ",", " "),
				SeedMode: config.P2P.SeedMode,
			})
		pexReactor.SetLogger(logger.With("module", "pex"))
		sw.AddReactor("PEX", pexReactor)
	}

	sw.SetAddrBook(addrBook)

	// run the profile server
	profileHost := config.ProfListenAddress
	if profileHost != "" {
		go func() {
			logger.Error("Profile server", "err", http.ListenAndServe(profileHost, nil))
		}()
	}

	node := &Node{
		config:        config,
		genesisDoc:    genDoc,
		privValidator: privValidator,

		transport: transport,
		sw:        sw,
		addrBook:  addrBook,
		nodeInfo:  nodeInfo,
		nodeKey:   nodeKey,

		stateDB:          stateDB,
		blockStore:       blockStore,
		bcReactor:        bcReactor,
		mempoolReactor:   mempoolReactor,
		consensusState:   consensusState,
		consensusReactor: consensusReactor,
		evidencePool:     evidencePool,
		proxyApp:         proxyApp,
		txIndexer:        txIndexer,
		indexerService:   indexerService,
		eventBus:         eventBus,
	}
	node.BaseService = *cmn.NewBaseService(logger, "Node", node)
	return node, nil
}
```

结合我们之前的分析，我们可以知道在新建一个peer的时候的主要的流程 1. 创建一个blockstore的实例，并且有个Reactor，方便注册到switch里面 2. 创建或者打开了state实例，方便更新维护本节点的状态信息 3. 创建mempool，方便处理各种交易，往ABCI以及共识传递信息，并且写入区块链 4. 创建ABCI，并且重放所有的区块 5. 创建证据池，证据Reactor 6. 创建共识 7. 创建P2P还有switch，开启地址簿功能 8. 成功

接下来就得开始运行了

```go
func (n *Node) OnStart() error {
	now := tmtime.Now()
	genTime := n.genesisDoc.GenesisTime
	if genTime.After(now) {
		n.Logger.Info("Genesis time is in the future. Sleeping until then...", "genTime", genTime)
		time.Sleep(genTime.Sub(now))
	}

	err := n.eventBus.Start()
	if err != nil {
		return err
	}

	// Add private IDs to addrbook to block those peers being added
	n.addrBook.AddPrivateIDs(splitAndTrimEmpty(n.config.P2P.PrivatePeerIDs, ",", " "))

	// Start the RPC server before the P2P server
	// so we can eg. receive txs for the first block
	if n.config.RPC.ListenAddress != "" {
		listeners, err := n.startRPC()
		if err != nil {
			return err
		}
		n.rpcListeners = listeners
	}

	if n.config.Instrumentation.Prometheus &&
		n.config.Instrumentation.PrometheusListenAddr != "" {
		n.prometheusSrv = n.startPrometheusServer(n.config.Instrumentation.PrometheusListenAddr)
	}

	// Start the transport.
	addr, err := p2p.NewNetAddressStringWithOptionalID(n.config.P2P.ListenAddress)
	if err != nil {
		return err
	}
	if err := n.transport.Listen(*addr); err != nil {
		return err
	}

	n.isListening = true

	// Start the switch (the P2P server).
	err = n.sw.Start()
	if err != nil {
		return err
	}

	// Always connect to persistent peers
	if n.config.P2P.PersistentPeers != "" {
		err = n.sw.DialPeersAsync(n.addrBook, splitAndTrimEmpty(n.config.P2P.PersistentPeers, ",", " "), true)
		if err != nil {
			return err
		}
	}

	// start tx indexer
	return n.indexerService.Start()
}
```

首先是开启RPC服务，是为了和我们这些客户端进行通信的，然后开启P2P的地址监听，最后打开所有的服务。 程序运行起来了。 结束的函数也差不多，就不粘贴了。 我们现在可以大体说一下Tendermint的工作流程了，创建一系列服务，然后注册到switch上，由switch统一管理。 看完后想着去画一个Tendermint的结构图，找时间试一试看看能不能画出来。