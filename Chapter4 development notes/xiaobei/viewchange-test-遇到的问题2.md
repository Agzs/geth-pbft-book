* 出现的问题：还是在`func (instance *pbftCore) processNewView() events.Event`里报的错：
```
2017/11/14 21:30:03 Replica 2 missing assigned, non-checkpointed request batch ?�����t ��4�����Q��-���y��<r
2017/11/14 21:30:03 Replica 2 requesting to fetch batch ?�����t ��4�����Q��-���y��<r

```
* 出现错误的原因：报错的地方为：
```
			if _, ok := instance.blockStore[d]; !ok { //=> reqBatchStore -> blockStore --Agzs
				logger.Warningf("Replica %d missing assigned, non-checkpointed block %x",
					instance.id, d)
				//fmt.Println("ok is", ok)
				if _, ok := instance.missingReqBatches[d]; !ok {
					logger.Warningf("Replica %v requesting to fetch block %x",
						instance.id, d)
					newReqBatchMissing = true
					instance.missingReqBatches[d] = true
				}
			}
```
出现该错误的原因是因为`Xset`里存储的`block`在`blockStore`里没有存储，而`Xset`里存储的`block`是从`pset`、`qset`里面读取出来的。经查看共识消息的传播过程，发现`pset`和`qset`存储的`blockhash`的信息都是从`certstore`里读取的，而不是从`blockstore`。在第一次进行`viewchange`的时候之所以能够正确进行，是因为初始状态时`pset`和`qset`里都没有存储任何信息：
```
2017/11/15 20:31:30 Replica 1 restored state: view: 0, seqNo: 0, pset: 0, qset: 0, blockStore: 0, chkpts: 1 h: 0
```
当进行`pbft`共识的时候，在函数`func (instance *pbftCore) recvPrePrepare(preprep *types.PrePrepare) error`里的`block`进行判断：
```
if _, ok := instance.blockStore[string(preprep.BlockHash)]; !ok && string(preprep.BlockHash) != "" {
		digest := string(preprep.Block.Hash().Bytes())
		......
		instance.blockStore[digest] = preprep.Block
		......
	}
```
如果区块不在`blockstore`里就把它存到`blockstore`里。这样，每进行一次共识，都会将`block`存储到`blockstore`里，同时关于`block`的信息也都存储到了`pset`、`qset`里。当关闭`geth-pbtf`，重新启动时，`pset`、`qset`里存储的区块信息仍在，但是`blockstore`里存储的区块信息清空了：

第一次启动`geth-pbft`：
```
2017/11/15 20:31:30 Replica 1 restored state: view: 0, seqNo: 0, pset: 0, qset: 0, blockStore: 0, chkpts: 1 h: 0
```
第二次启动`geth-pbft`:
```
2017/11/15 21:14:03 Replica 1 restored state: view: 1, seqNo: 10, pset: 5, qset: 10, blockStore: 1, chkpts: 1 h: 0
```
由上述信息可见，区块都已经挖到第10个了，在`blockstore`里却只存储了一个区块。~因此推断出现上述错误的原因是`blockStore`存储问题。~

接收到区块的存储是在`func (instance *pbftCore) recvPrePrepare(preprep *types.PrePrepare) error`里，在存储后增加了一个输出，输出当前在`blockstore`里存储的区块的`digest`：
```
                for key := range instance.blockStore { ////--xiaobei 11.16
			logger.Infof("blockstore digest is %x", key)
		}
```
经输出结果发现，在每一个`view`下，`blockstore`里区块的存储是没有任何问题的：
```
2017/11/16 09:48:00 Replica 2 received pre-prepare from replica 0 for view=0/seqNo=1
INFO [11-16|09:48:00] Replica storing block in outstanding block store Replica(PeerID)=2 hash=fe77dc…554253
2017/11/16 09:48:00 blockstore digest is fe77dc4b817bd2be04a373d231261e05a5bc737847b1242e7c9d919b49554253
```
当挖到第6个块时：
```
2017/11/16 09:49:30 Replica 2 received pre-prepare from replica 1 for view=1/seqNo=6
INFO [11-16|09:49:30] Replica storing block in outstanding block store Replica(PeerID)=2 hash=fd549e…6ff0f3
......
2017/11/16 09:49:30 blockstore digest is fd549e8a0fe954e268f41dc263f0e7295ed71687dd8b8c6358aff976576ff0f3
2017/11/16 09:49:30 blockstore digest is 944db88485614f2417678ca5ce79769975fc4a28a4271033ab99e37d4f47972a
2017/11/16 09:49:30 blockstore digest is a24e71c8a348b9e9f410f10952ee4643de0fe5444ff8842403566bf5aec6eaf7
2017/11/16 09:49:30 blockstore digest is e36111de4b9e0ed3b4c7fa24994332151d2a9db9a046f4b429c9cb123954793e
2017/11/16 09:49:30 blockstore digest is 480fdabed83d8bc39fee8d9bc22b09c519213ff61f05187085f778a466e77116

```
>此时`blockstore`的长度只有5是因为期间发生了`viewchange`,在`view0`时存储了一个区块，当`viewchange`成功之后，就会清空`blockstore`里的信息。

由此可见，`blockstore`里区块的存储不存在问题，~问题出在重新启动`geth-pbtf`时`blockstore`的恢复重建上~。

在状态重建函数`func (instance *pbftCore) restoreState()`增加输出语句进行调试，发现：
```
2017/11/16 10:57:07 reqBlockesPacked's length is 6
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
2017/11/16 10:57:07 restore blockstore digest is 1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347
```
恢复出的`blockstore`的`digest`都是相同的，但是长度是对的，这也是恢复出的`blockStore`长度为1的原因。

进一步进行调试，发现问题出在`reqBlockesPacked, err := instance.helper.ReadStateSet("reqBlock.")`这里，`reqBlockesPacked`的类型为`var reqBlockesPacked map[string][]byte`，经输出发现，它的`value`值始终是一样的：
```
2017/11/16 11:20:35 reqBlockesPacked's length is 6
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
2017/11/16 11:20:35 reqBlockesPacked value is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d
```
进一步排查`block`在数据库中的存储，发现在存储区块的函数`func (instance *pbftCore) persistRequestBlock(digest string)`里，从`blockstore`里读取出的不同的区块，经过语句`reqBlockPacked, err := json.Marshal(reqBock)`后，`reqBlockPacked`结果是一样的。
```
2017/11/17 11:55:03 reqBock is Block(#115): Size: 606.00 B {
MinerHash: 61d6fc1b8458212a1369ef0a51698d5451c99cdf452f197665ab03d42fcbfb01
Header(948b1526747fde350d05fa793ed1eca36f447a31e6d185a7edcc763ec2d82ab7):
[
......
}
2017/11/17 11:55:03 reqBlockPacked is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d

...
2017/11/17 11:55:17 reqBock is Block(#116): Size: 606.00 B {
MinerHash: 5cb19e017ea42021a81a7340007a46882a7531fcda0176507fb590fd6a8b9195
Header(cbd320d43c9466d7a167a87e62393774c7248de32f7fe49acd50eb8744eccf9d):
[
......
}
2017/11/17 11:55:17 reqBlockPacked is 7b2252656365697665644174223a22303030312d30312d30315430303a30303a30305a222c22526563656976656446726f6d223a6e756c6c7d

```
经查找资料发现，`reqBock`的结构问题使`block`序列化成`JSON`格式失败，按照下面的方法解决完`block`的存储问题后，发现最开始提示的问题还是没有解决，`pset`、`qset`里存储的区块的`digest`和`blockstore`里存储的不一样。
```
2017/11/20 19:49:56 pset--block digest is efbfbdefbfbdefbfbd432aefbfbd546eefbfbdefbfbdefbfbdc2a96c28efbfbdefbfbd23efbfbd53efbfbd1355efbfbd725b7202efbfbdefbfbd06efbfbd
...
2017/11/20 19:49:56 pset--block digest is efbfbd1f3cefbfbd21694d6d6aefbfbdefbfbdefbfbdd0830cefbfbdefbfbdefbfbd6a6b0b35efbfbddaab3cefbfbd5fefbfbdefbfbdefbfbdefbfbd
2017/11/20 19:49:56 pset--block digest is 0f0623efbfbdefbfbd5928216defbfbd761c137fefbfbd64efbfbd53efbfbd68efbfbd4cefbfbd04112305efbfbdefbfbd080236
2017/11/20 19:49:56 qset--block digest is 265e2c3aefbfbdefbfbd3f0132efbfbddb87efbfbdefbfbdefbfbd6a4479efbfbd191a0aefbfbd68efbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbd34
2017/11/20 19:49:56 qset--block digest is 60193d16efbfbd3a19efbfbd36efbfbdefbfbdefbfbdefbfbdefbfbd280befbfbd6befbfbd06efbfbdefbfbd4cefbfbd5a1311efbfbdefbfbdefbfbd6276
2017/11/20 19:49:56 qset--block digest is efbfbd1f3cefbfbd21694d6d6aefbfbdefbfbdefbfbdd0830cefbfbdefbfbdefbfbd6a6b0b35efbfbddaab3cefbfbd5fefbfbdefbfbdefbfbdefbfbd
2017/11/20 19:49:56 qset--block digest is efbfbd22efbfbdefbfbd1a7812efbfbdefbfbd790a2e49efbfbdefbfbd7f286008426a02efbfbdefbfbdefbfbd30efbfbd2205efbfbdefbfbdefbfbd
2017/11/20 19:49:56 qset--block digest is efbfbdefbfbdefbfbd432aefbfbd546eefbfbdefbfbdefbfbdc2a96c28efbfbdefbfbd23efbfbd53efbfbd1355efbfbd725b7202efbfbdefbfbd06efbfbd
...
2017/11/20 19:49:56 reqBlockesPacked's length is 10
2017/11/20 19:49:56 restore blockstore digest is 0f0623b5f55928216db7761c137fd764b453ed68e44ca104112305a790080236
2017/11/20 19:49:56 restore blockstore digest is 265e2c3ae9c33f0132d9db878d92e66a4479cc191a0a80689bacbba4adf2af34
2017/11/20 19:49:56 restore blockstore digest is 2ed126b8eb16f23153fcd8c843d7ddaec6a25d5d4e6969bcea03b5dfdfea5782
2017/11/20 19:49:56 restore blockstore digest is b722bfd81a7812f1e5790a2e49b69b7f286008426a02f29b993092220594acb6
2017/11/20 19:49:56 restore blockstore digest is c69b3a68493d0b9bf2c64490051084db253ba00ce5a7bd70600a97fb202d1693
...

```
经过`debug`发现，问题出在`raw, err := json.Marshal(&types.PQset{Set: set})`这条语句上，`set`里的`viewchange`里的`BlockHash`不能正确转换成`json`格式，转换后的`BlockHash`与转换前的`BlockHash`对应不上。
* 解决问题的方法：
1. 首先改一下输出格式，把`%s`换成`%x`。
2. 解决`blockstore`存储问题: 经查阅资料后发现，当把结构体会转化为JSON对象的时候，只有结构体里边以大写字母开头的可被导出的字段才会被转化输出。~在语句`reqBlockPacked, err := json.Marshal(reqBock)`里，`reqBock`的类型为`Block`类型，将`Block`内的成员变量的首字母改为大写：~
```
type Block struct { ////change for json.Marshal()--xiaobei 11.17
	// header       *Header
	// uncles       []*Header
	// transactions Transactions

	// // caches
	// hash atomic.Value
	// size atomic.Value

	// // Td is used by package core to store the total difficulty
	// // of the chain up to and including the block.
	// td *big.Int

	Hheader       *Header
	Uuncles       []*Header
	Ttransactions Transactions

	// caches
	Hhash atomic.Value
	Ssize atomic.Value

	// Td is used by package core to store the total difficulty
	// of the chain up to and including the block.
	Ttd *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}
```
在`types`下新建文件`gen_block_json.go`，在该文件里定义的两个函数能够实现将`Block`转换成能够进行`json`编码的`Block`，并在`func (instance *pbftCore) persistRequestBlock(digest string)`和`func (instance *pbftCore) restoreState() `里面调用这两个函数。
```
//=> overwrite persistRequestBlock. --Agzs
func (instance *pbftCore) persistRequestBlock(digest string) {
	...
	reqBlockPacked, err := reqBock.MarshalJSON() ////--xiaobei 11.22
	...
}
func (instance *pbftCore) restoreState() {
	...
	if err == nil {
		for k, v := range reqBlockesPacked {
			reqBlock := &types.Block{} //=>RequestBatch -> BLock. --Agzs
			//err = json.Unmarshal(v, reqBlock)
			err = reqBlock.UnmarshalJSON(v) ////--xiaobei 11.22
		        ...
		}
	} else {
		....	}


}
```
经修改后，`block`能够正确存储进`blockstore`里，每次重新启动后，`blockstore`的长度也是正确的：
```
2017/11/20 19:49:56 Replica 0 restored state: view: 1, seqNo: 9, pset: 8, qset: 9, blockStore: 10, chkpts: 1 h: 0
```
3. 解决`pset`、`qset`中`digest`存储错误的问题：将`viewchange`结构体里的`BlockHash`由`string`类型变为`common.hash`的类型，此次更改还会涉及到其他地方的关于格式的修改，在这不做赘述，详情参见代码 。

解决完以上问题之后，最初的报错最终解决。
