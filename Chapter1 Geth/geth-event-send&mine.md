## 摘要
geth中的事件类型需要向`event`模块进行注册. 每一个外部节点若与geth相链接就会生成一个`peer`实例, `peer`里保存了在网络上与外部节点的通道. `peer`可以向`event`模块订阅相应的事件类型, 内部产生的消息会发送给`event`, `event`保存了订阅该事件类型的`peer`信息, 并想这些`peer`进行广播. `peer`收到消息后会通过网络将消息实际发送给外部节点.

即: 事件产生 >> event >> peer >> 网络 >> 外部节点

特别地, 对于mine过程, 成功挖出块的传递是: ethash.mine() >> ethash.Seal() >> miner.CpuAgent.mine() >> miner.worker >> miner.worker.mux(即:event.TypeMux) >> event.TypeMux.Post() >> peer

## 过程

geth中, 由ethash产生的事件很单一, 只有挖块成功等几个事件类型. geth中生成块的接口定义为`Seal`. 在ethash中, `Seal`会调用内部实现的`func mine()`, 由其进行实际的挖矿工作. 定义如下:
```go
// mine is the actual proof-of-work miner that searches for a nonce starting from
// seed that results in correct final block difficulty.
func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
	// Extract some data from the header
	...
	// Start generating random nonces until we abort or find a good one
	var (
		attempts = int64(0)
		nonce    = seed
	)
	...
	for {
		select {
		case <-abort:
			...
			return

		default:
			// We don't have to update hash rate on every nonce, so update after after 2^X nonces
			attempts++
			...
			// Compute the PoW value of this nonce
			digest, result := hashimotoFull(dataset, hash, nonce)
			if new(big.Int).SetBytes(result).Cmp(target) <= 0 {
				// Correct nonce found, create a new header with it
				...
				// Seal and return a block (if still needed)
				select {
				case found <- block.WithSeal(header):
					logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
				case <-abort:
					logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
				}
				return
			}
			nonce++
		}
	}
}
```

看到, `func mine()`参数中包括一个`found`channel, 当成功获得一个合法的块时, 会将其header通过`found`传出. `found`是由`func Seal()`定义的, `func Seal()`内部接收了这个消息并将header返回. `func Seal()`对外会被`miner/agent`调用:
```go
func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
	if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
		log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
		self.returnCh <- &Result{work, result}
	} else {
		if err != nil {
			log.Warn("Block sealing failed", "err", err)
		}
		self.returnCh <- nil
	}
}
```
`CpuAgent`将收到的结果传送给发送给`returnCh`, 该channel是在`worker`给赋给`CpuAgent`的, 本身是`worker`的成员变量`recv   chan *Result`. `worker`会在`func wait()`接受来自`recv`的结果, 在`func wait()`中, `worker`初步解析结果, 并将结果广播:
```go
// broadcast before waiting for validation
go func(block *types.Block, logs []*types.Log, receipts []*types.Receipt) {
	self.mux.Post(core.NewMinedBlockEvent{Block: block})
	self.mux.Post(core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
	if stat == core.CanonStatTy {
		self.mux.Post(core.ChainHeadEvent{Block: block})
		self.mux.Post(logs)
	}
	if err := core.WriteBlockReceipts(self.chainDb, block.Hash(), block.NumberU64(), receipts); err != nil {
		log.Warn("Failed writing block receipts", "err", err)
	}
}(block, work.state.Logs(), work.receipts)
```
调用的是`func worker.mux.Post()`. `mux`是`worker`的成员变量, 类型为`*event.TypeMux`. 该类型定义在`event/event.go`.
```go
type TypeMux struct {
	mutex   sync.RWMutex
	subm    map[reflect.Type][]*TypeMuxSubscription
	stopped bool
}
...
// Post sends an event to all receivers registered for the given type.
// It returns ErrMuxClosed if the mux has been stopped.
func (mux *TypeMux) Post(ev interface{}) error {
	event := &TypeMuxEvent{
		Time: time.Now(),
		Data: ev,
	}
	rtyp := reflect.TypeOf(ev)
	mux.mutex.RLock()
	if mux.stopped {
		mux.mutex.RUnlock()
		return ErrMuxClosed
	}
	subs := mux.subm[rtyp]
	mux.mutex.RUnlock()
	for _, sub := range subs {
		sub.deliver(event)
	}
	return nil
}
```
在geth的事件处理中, geth会针对各种消息类型生成一个订阅列表, `peer`可以主动的订阅各类消息, 当产生消息后, `event`模块会将此消息发送给所有订阅该消息类型的节点. 一次订阅动作会生成一个`TypeMuxSubscription`结构, 该结构里保存了与订阅者交互的通道(channel).
```go
// TypeMuxSubscription is a subscription established through TypeMux.
type TypeMuxSubscription struct {
	mux     *TypeMux
	created time.Time
	closeMu sync.Mutex
	closing chan struct{}
	closed  bool

	// these two are the same channel. they are stored separately so
	// postC can be set to nil without affecting the return value of
	// Chan.
	postMu sync.RWMutex
	readC  <-chan *TypeMuxEvent
	postC  chan<- *TypeMuxEvent
}
```
接下来`peer`就会接收到相应的消息, 并通过网络传送给节点.