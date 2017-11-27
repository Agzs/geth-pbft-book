> 触发`viewchange`的方法：修改配置文件`config.yaml`里`request`的时间为0.00003s，这个参数代表的是一个请求（`block`或者是收到的`preprepare`消息等）从接收到一直到处于`committed`状态所需要的时间。如果超过这个时间就会触发`viewchange`.

# 一、
* 出现的问题：当触发viewchange操作的时候，各个节点会曝出类似于如下错误：
```
2017/11/07 19:50:58 Replica 0 found incorrect signature in view-change message: %!s(<nil>)
```
* 出现问题的原因：在`func (instance *pbftCore) sendViewChange() events.Event`里，形成`vc.Signature`的时候有用到了`vc`：
```
vc.Signature, _ = signFn(accounts.Account{Address: signer}, sigHash(nil, vc).Bytes())
```
此时，`sigHash`里的`vc`里的参数`vc.Signature`是没有被赋值的。而在`func (instance *pbftCore) recvViewChange(vc *types.ViewChange) events.Event`里验证签名的时候用的`vc`，里面的`vc.Signature`是已经被赋值的，所以导致签名验证失败。
* 解决问题的方法：在`pbft.go`里的`func sigHash(header *types.Header, viewChange *types.ViewChange) (hash common.Hash)`把`viewChange.Signature`注释掉。
# 二、
* 出现错误：
在signer1：
```
2017/11/07 20:41:28 Replica 0 ignoring processNewView as it could not find view 1 in its newViewStore
```
在signer2：
```
2017/11/07 20:41:28 Replica 1 received view change quorum, processing new view
2017/11/07 20:41:28 Replica 1 appending checkpoint from replica 0 with seqNo=0, h=0, and checkpoint digest XXX GENESIS
2017/11/07 20:41:28 Replica 1 appending checkpoint from replica 1 with seqNo=0, h=0, and checkpoint digest XXX GENESIS
2017/11/07 20:41:28 Replica 1 appending checkpoint from replica 2 with seqNo=0, h=0, and checkpoint digest XXX GENESIS
2017/11/07 20:41:28 Stopping timer
panic: runtime error: index out of range
```
* 出现错误的原因：经打印排查发现，问题出现在函数`func ConvertMapToStruct(msgList map[uint64]string) []*types.XSet`里面，在该函数里只定义了数组`xset`，但是没有给他分配空间。
* 解决问题的方法：增加两条语句
```
l := len(msgList)
xset := make([]*types.XSet, l)
```
问题解决。
# 三、
* 出现的问题：`signer1`发送`viewchange`消息后，其他节点包括`signer1`一直循环输出如下语句(以`signer1`为例)：
```
2017/11/09 09:27:39 Replica 0 already has a view change message for view 1 from replica 0
```
不能进行接下来的操作。
* 出现问题的原因：出现这种状态的原因是因为只有`signer1`发送了`viewchange`消息，其他节点都没有发送，这样会造成永远无法收到一个`quorum`的`viewchange`消息,也就无法进行接下来的操作。
* 解决问题的方法：在`config.yaml`里把`nullrequest`的时间由0变为5s。`nullrequest`对应的时间是`nullRequestTimeout`。当`block`处于`committed`状态或者是触发`stateUpdatedEvent`的时候会通过调用`func (instance *pbftCore) executeOutstanding()`层层调用进而调用`func (instance *pbftCore) startTimerIfOutstandingBlocks()`来触发`nullRequestEvent`。该事件触发后，如果该节点是主节点，会重新发送`preprepare`消息，如果是非主节点，会发送`viewchange`消息。由于在这里循环输出如上语句，所以无论是主节点还是非主节点都无法进行接下来的操作，于是在`func (instance *pbftCore) recvViewChange(vc *types.ViewChange) events.Event`的`if`条件语句里增加了如下语句来触发`nullRequestEvent`:
```
if _, ok := instance.viewChangeStore[vcidx{vc.View, vc.ReplicaId}]; ok {
		logger.Warningf("Replica %d already has a view change message for view %d from replica %d", instance.id, vc.View, vc.ReplicaId)
		/////////
		////--xiaobei 11.9
		if !flag && instance.nullRequestTimeout > 0 {
			timeout := instance.nullRequestTimeout
			if instance.primary(instance.view) != instance.id {
				// we're waiting for the primary to deliver a null request - give it a bit more time
				timeout += instance.requestTimeout
			}
			instance.nullRequestTimer.Reset(timeout, nullRequestEvent{})
			flag = true
		}
		/////////
		return nil
	}
```
`flag`的作用是使该事件的开启只开启一次。加入后该段语句后，`viewchange`就可以不止主节点触发，防止进入无限循环之中。
# 四、
* 出现的问题：各个节点不能正确验证`new view`里的`xset`,以replica0为例:
```
2017/11/09 17:27:35 Replica 0 failed to verify new-view Xset: computed map[19:>��>�E�#�_Tg�)�bIvh��<V}!S 7:�}f/7� ��ʜ��䤿������V��޼0 18:�#l�7����Q?���d/��V?1Rp��5�� 4:��oz�<a�>L��d�v*����-/��)���f�� 11:&^ɶ��t)�w4�S����F�-e�mѩ 15:��D;�+��i���Ǥ�xF`Iٚ���z�%���� 16:U�c�V߲�Փ$&���F۽
                                                                      �c��9۴�=� 6:ѻ��GB�(׍$���ݶ����9/))���/6MV 2:/'�K�9ަ�GG�TH�E:�]������, 12:�i܀0���0C\E�����|�C܏��[6��W 5:_��E�^�uR�
                                    �q�q��r���.�� 14:]�pft=�������Θ��Tb+�8z;�q�S�� 17:��k<*��
          �j��AH��#�+ 3:��A	�Y�;Ls�.v����Qb�yAq�М 9:h������N�iƩCY��	�|�7k�촚��  13:?	�1���Ѳ����1/��mؿ;V��� 20:��'�<���Q!��R�?rwr�� m��n�Q�Wݓ 8:�C����m�������*5s�>#�(��cC 10:�Qt&�'����u��|��p��~8|���Pn 1:4]�$%©bRRGXU�B�A����A��W�a��], received [0xc42131f7e0 0xc42131f800 0xc42131f820 0xc42131f840 0xc42131f880 0xc42131f8a0 0xc42131f8e0 0xc42131f900 0xc42131f920 0xc42131f960 0xc42131f9a0 0xc42131f9c0 0xc42131f9e0 0xc42131fa20 0xc42131fa40 0xc42131fa60 0xc42131fa80 0xc42131faa0 0xc42131fac0 0xc42131fb00]
```
* 出现错误的原因：在`func (instance *pbftCore) processNewView() events.Event`里将`msgList`和`nv.Xset`进行了比较，而`nv.Xset`是进行结构体转换后的`msgList`，他俩不可能相等，自然报错。
* 解决问题的方法：将`msgList`先进行格式变化，再与`nv.Xset`进行比较，`msglist := ConvertMapToStruct(msgList)`,问题解决。
# 五、
* 出现的问题：当`signer1`触发`viewchange`之后，`signer2`成为了主节点，但是`signer2`不能进行挖矿，报错：
```
panic:assignment to entry in nil map
```
* 出现错误的原因：当`viewchange`完成之后，会触发`viewchangedEvent`，在`func (instance *pbftCore) ProcessEvent(e events.Event) events.Event`里对该事件进行处理的时候，误将`instance.blockStore`置为了`nil`。
* 解决问题的方法：在对`viewchangedEvent`进行处理的时候，重新给`instance.blockStore`分配空间，而不是置为空：
```
instance.blockStore = make(map[string]*types.Block) ////--xiaobei 11.9
```
# 六、
* 出现的问题：解决完上述问题之后，开启四个节点能够正确进行`viewchange`，`viewchange`完毕后也能正确进行挖矿、同步。但是当我把所有的节点都`exit`后再重新开启四个节点进行挖矿的时候，就会不断出现类似于以下的问题：
```
Replica 1 failed to verify new-view Xset: computed [0xc42179c300 0xc42179c320 0xc42179c340 0xc42179c360 0xc42179c380 0xc42179c3e0 0xc42179c420 0xc42179c440 0xc42179c460 0xc42179c4a0 0xc42179c4c0 0xc42179c500 0xc42179c520 0xc42179c560 0xc42179c580 0xc42179c5a0 0xc42179c5c0 0xc42179c5e0 0xc42179c600 0xc42179c620], received [0xc421785ee0 0xc421785f00 0xc421785f20 0xc421785f40 0xc421785f80 0xc421785fc0 0xc421792000 0xc421792020 0xc421792060 0xc4217920e0 0xc421792100 0xc421792120 0xc421792140 0xc421792180 0xc4217921a0 0xc4217921c0 0xc4217921e0 0xc421792200 0xc421792220 0xc421792260]
```
进而不断的触发`viewchange`。
* 出现错误的原因：~看了一下`Xset`没有在数据库中进行存储，所以猜想是这个原因导致每次重启之后报错。~
当收到一个`quorum`的`viewchange`消息，处理`NewView`的时候：
1. 如果节点是主节点：触发`func (instance *pbftCore) sendNewView() events.Event`
2. 如果节点是非主节点：触发`func (instance *pbftCore) recvNewView(nv *types.NewView) events.Event`
两个函数都会最终触发`func (instance *pbftCore) processNewView() events.Event`。奇怪的是在同一个节点当中，无论是主节点还是非主节点，在调用`msgList := instance.assignSequenceNumbers(vset, cp.SequenceNumber) `生成`msgList`的时候，`processNewView()`与`sendNewView()/recvNewView(nv *types.NewView)`里调用`func (instance *pbftCore) assignSequenceNumbers(vset []*types.ViewChange, h uint64) (msgList map[uint64]string)`传递的参数是相同的，但是生成的`msgList`却不相同：

signer2:(非主节点)
```
2017/11/13 17:29:39 Replica 1 received new-view 3
2017/11/13 17:29:39 ---recvNewView() msgList:{nv.vset:[0xc43462cd80 0xc43462ce10 0xc43462cea0]}
2017/11/13 17:29:39 receive new-view, v:3, X:[0xc422099460 0xc422099480 0xc4220994a0 0xc4220994c0 0xc422099500 0xc422099520 0xc422099560 0xc422099580 0xc4220995a0 0xc4220995e0 0xc422099600 0xc422099620 0xc422099640 0xc422099680 0xc4220996a0 0xc4220996c0 0xc4220996e0 0xc422099700 0xc422099720 0xc422099760]
.....
2017/11/13 17:29:39 ---processNewView() msgList:{nv.vset:[0xc43462cd80 0xc43462ce10 0xc43462cea0],cp.sequenceNumber:0}
2017/11/13 17:29:39 Replica 1 failed to verify new-view Xset: computed [0xc4220ae020 0xc4220ae060 0xc4220ae080 0xc4220ae0a0 0xc4220ae0c0 0xc4220ae0e0 0xc4220ae120 0xc4220ae140 0xc4220ae160 0xc4220ae180 0xc4220ae1a0 0xc4220ae1e0 0xc4220ae200 0xc4220ae220 0xc4220ae240 0xc4220ae260 0xc4220ae280 0xc4220ae2a0 0xc4220ae2c0 0xc4220ae2e0], received [0xc422099460 0xc422099480 0xc4220994a0 0xc4220994c0 0xc422099500 0xc422099520 0xc422099560 0xc422099580 0xc4220995a0 0xc4220995e0 0xc422099600 0xc422099620 0xc422099640 0xc422099680 0xc4220996a0 0xc4220996c0 0xc4220996e0 0xc422099700 0xc422099720 0xc422099760]

```
signer4：（主节点）
```
2017/11/13 16:22:41 ---sendNewView() msgList:{nv.vset:[0xc437ec6090 0xc437e98480 0xc437ec6990],cp.sequenceNumber:0}
2017/11/13 16:22:41 Replica 3 is new primary, sending new-view, v:3, X:[0xc437efc5e0 0xc437efc600 0xc437efc620 0xc437efc640 0xc437efc660 0xc437efc680 0xc437efc6a0 0xc437efc6c0 0xc437efc6e0 0xc437efc700 0xc437efc720 0xc437efc740 0xc437efc760 0xc437efc780 0xc437efc7a0 0xc437efc7c0 0xc437efc7e0 0xc437efc800 0xc437efc820 0xc437efc840]
......
2017/11/13 16:22:41 ---processNewView() msgList:{nv.vset:[0xc437ec6090 0xc437e98480 0xc437ec6990],cp.sequenceNumber:0}
2017/11/13 16:22:41 Replica 3 failed to verify new-view Xset: computed [0xc437efc9a0 0xc437efc9c0 0xc437efc9e0 0xc437efca00 0xc437efca20 0xc437efca40 0xc437efca60 0xc437efca80 0xc437efcaa0 0xc437efcac0 0xc437efcae0 0xc437efcb00 0xc437efcb20 0xc437efcb40 0xc437efcb60 0xc437efcb80 0xc437efcba0 0xc437efcbc0 0xc437efcbe0 0xc437efcc00], received [0xc437efc5e0 0xc437efc600 0xc437efc620 0xc437efc640 0xc437efc660 0xc437efc680 0xc437efc6a0 0xc437efc6c0 0xc437efc6e0 0xc437efc700 0xc437efc720 0xc437efc740 0xc437efc760 0xc437efc780 0xc437efc7a0 0xc437efc7c0 0xc437efc7e0 0xc437efc800 0xc437efc820 0xc437efc840]
```
但是对于同一节点来说，发送或接收到的`xset`与处理时的`xset`是相同的。

在`func (instance *pbftCore) sendNewView() events.Event`和`func (instance *pbftCore) processNewView() events.Event`里都加入了两个循环：
```
        msgList := instance.assignSequenceNumbers(nv.Vset, cp.SequenceNumber)
        ......
	msglist := ConvertMapToStruct(msgList) ////--xiaobei 11.9
	......
	for key, value := range msgList { ////--xiaobei 11.14
		logger.Infof("processNewView msgList[%d]=%x", key, value)
	}
	for key2, value2 := range msglist { ////--xiaobei 11.14
		logger.Infof("processNewView xset[%d]={Seq:%d,Hash:%x}", key2, value2.Seq, value2.Hash)
	}
```
分别查看经过`func (instance *pbftCore) assignSequenceNumbers(vset []*types.ViewChange, h uint64) (msgList map[uint64]string) `和`func ConvertMapToStruct(msgList map[uint64]string) []*types.XSet`之后，`msgList`的值是否发生变化，结果发现能够正确进行转换，没有发生变化。接着比较了`sendNewView()`和`processNewView()`里的`xset[]`,发现二者相同下标对应的值却不同，比如：

`sendNewView()`:
```
xset[9]={Seq:6,Hash:efbfbdefbfbd293cefbfbd09efbfbd6d0e3a703fefbfbdefbfbd39efbfbd114aefbfbd28efbfbdefbfbd7218efbfbdefbfbd6defbfbd1defbfbdefbfbdefbfbd}
```
`processNewView()`:
```
xset[0]={Seq:6,Hash:efbfbdefbfbd293cefbfbd09efbfbd6d0e3a703fefbfbdefbfbd39efbfbd114aefbfbd28efbfbdefbfbd7218efbfbdefbfbd6defbfbd1defbfbdefbfbdefbfbd}
```
两个函数里面`seq:6`对应的`xset[]`的下标一个是0，一个是9。在第一次进行`viewchange`的时候能够正确进行是因为当时`xset`里面只有一个值：
```
2017/11/14 15:59:09 sendNewView xset[0]={Seq:1,Hash:db3b8fbe2be02696f8d0cbf33f5b127800961ef6822aa8540cd0fcc967ff273c}
```
* 解决问题的方法：在`processNewView()`里面初始化一个数组`msglist2`，与`msglist`类型相同。当收到的`NewView`里的`Xset`与新生成的`msgList`里的`seq`相同时，就执行`msglist2[i] = msglist[j]`,这样就能保证新生成的`msgList`与`nv.Xset`相同数组下标对应的`seq`相同：
```
        l := len(msgList)
	msglist2 := make([]*types.XSet, l)
	for i := 0; i < l; i++ {
		for j := 0; j < l; j++ {
			if nv.Xset[i].Seq == msglist[j].Seq {
				msglist2[i] = msglist[j]
				break
			}
		}
	}
```