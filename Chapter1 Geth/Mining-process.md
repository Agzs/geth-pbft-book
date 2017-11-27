## 挖矿详细过程
备注：文中所提到的函数都未带参数或参数不全，与源代码不同，并且挑选了关键函数，只用于说明。">>"代表调用，"=>"代表顺序执行，"==>"代表接口实现。
### 前期注册
#### geth()>>makeFullNode()>>utils.RegisterEthService()
* 在utils.RegisterEthService()中会注册eth.New()函数（该函数在此只是注册，实际被调用在startNode()>>utils.StartNode()>>Node.Start()中的for循环下的constructor()，constructor是个函数类型），eth.New()函数会依次调用几个比较重要的函数：
    * core.NewBlockChain()
    * core.NewTxPool()
    * NewProtocolManager()
    * miner.New()，该函数实例化miner对象，与挖矿联系紧密
	
* miner.New()先实例化miner，其中成员变量worker由newWoker()实例化，newWorker()会先后启动两个goroutine，分别是worker.update()和worker.wait()：
    * worker.update() 通过不同的events执行不同的操作，这些events都是通过worker.mux.Subscribe(...Event...)订阅的
    * worker.wait() 从worker.recv中获取blockh和work，进行区块入链和广播区块等操作，最后会调用worker.commitNewWork
	     * 分支一：worker.chain.InsertChain() => goroutine调用worker.mux.Post(core.NewMinedBlockEvent{Block:block})
		 * 分支二：worker.state.commitTo() => worker.chain.writeBlock() => goroutine多次调用worker.mux.Post()，所传递的参数不同

* miner.New()调用miner.Register(NewCpuAgent(eth.BlockChain(),engine)):
    * eth.BlockChain()
    * NewCpuAgent()实例化一个CpuAgent
    * Register():
	     * agent.Start()==>CpuAgent.Start()>>goroutine调用CpuAgent.update()>>goroutine调用CpuAgent.mine()>>CpuAgent.engine.Seal()>>ethash.mine()
	     * miner.worker.register(agent)>>agent.setReturnCh(worker.recv)

* miner.New()启动goroutine调用miner.update():
    * 订阅events，通过miner.mux.Subscribe(startEvent,DoneEvent,FailedEvent)实现
    * for循环遍历events.Chan()，即读取readC，通过switch分支进行操作：
	     * miner.stop()>>miner.worker.stop()
		 * miner.start()依次调用miner.worker.start()和miner.worker.commitNewWork()

### 后期运行
#### geth调用完makeFullNode()，接着会调用startNode()>>utils.StartNode()>>ethereum.StartMining()
* ethereum.StartMining()启动goroutine调用ethereum.miner.Start():
    * miner.worker.start()>>agent.Start()==>CpuAgent.Start()>>goroutine调用CpuAgent.update()>>goroutine调用CpuAgent.mine()>>CpuAgent.engine.Seal()==>ethash.mine()
    * miner.worker.commitNewWork():
	     * worker.engine.Prepare()=>
		 * work.commitTransactions()>>env.commitTransaction() => 启动goroutine，根据条件分别mux.Post()>>sub.deliver(event)
		 * worker.engine.Finalize()>>types.NewBlock()
         * worker.push(work)，其中work含有刚刚构造好的block，候选block作为work被push到给worker，实际上发送到agent的workCh中
		 
	* 前面已启动goroutine的CpuAgent.update()会读取workCh中的work，然后goroutine调用CpuAgent.mine()>>CpuAgent.engine.Seal()==>ethash.mine()
	* ethash.mine()会将添加nonce的block返回，保存在agent.resultch，即worker.wait()函数用到的worker.recv中
	* 前面已启动goroutine的Post()会发送NewMinedBlockEvent事件

