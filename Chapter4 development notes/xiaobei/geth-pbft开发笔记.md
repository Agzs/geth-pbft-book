# viewchange.go
1、在`func (instance *pbftCore) sendNewView() events.Event`里更改消息的传递格式，并把`pbftMessage`放入了`commChan`管道里。
```
instance.commChan <- &types.PbftMessage{Payload: &types.PbftMessage_NewView{NewView: nv}}
```
# pbft-core.go
1、`outstandingRequest`改为了`outstandingBlock`。

2、在`sendPrePrepare`里将`n`的初始化放在了`preprep`初始化的上面，将`preprep.SequenceNumber`初始化为n。

3、在`func (instance *pbftCore) recvMsg(msg *types.PbftMessage, senderID uint64) (interface{}, error)`里将获取`block`的语句注释掉了，因为`block`会在`prePrepare`消息里传递过来。

4、在`func (instance *pbftCore) recvPrePrepare(preprep *types.PrePrepare) error`里更改将`block`存储到`blockStore`的部分。



# handler.go
1、给`protocolManager`增加成员变量`pbftmanager`。

2、在`func (pm *ProtocolManager) handleMsg(p *peer) error`里增加对`ConsensusMsg`的处理。
# backend.go
1、在`func CreateConsensusEngine(ctx *node.ServiceContext, config *Config, chainConfig *params.ChainConfig, db ethdb.Database, pm *ProtocolManager) consensus.Engine`里将`PBFT`的`manager`赋值给`protocolManager`的成员变量`pbftmanager`。

