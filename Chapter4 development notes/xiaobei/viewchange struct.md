1、当当前执行区块的`seqNo`为`checkpoint`周期（即`pbftcore.K`）的整数倍的时候，就初始化`checkpoint`的信息。

2、当接受到一个`block`发送`preprepare`消息的时候，都将会将给该`block`分配的`seqNo+1`。`seqNo`默认是从0开始的。如果发生了`viewchange`且当前的`nv.Xset`里有给`block`重新分配的序号，那么`seqNo`就等于`nv.Xset`里存的`seqNo`的最大值；如果`nv.Xset`里为空，那么`seqNo`就等于当前结点的低水位。
# 数据清空
1、何时清空`blockstore`?
* 当节点接受到的`checkpoint`对应`block`的`seqNo`大于本节点的高水位并且大于高水位的`checkpoint`的个数大于`f+1`的时候，那么该节点刚执行完的`block.seqNo`对应的`checkpoint`将永远也达不到`stable`状态（`stable`状态是指收到`2f+1`条针对该`block.seqNo`的`checkpoint`信息），因为别的节点比本节点快太多，本节点错过了接受其他节点发送该`block.seqNo`对应的`checkpoint`的时机。在这种状况下，本节点将会清空`blockStore`、`outstandingBlocks`，从数据库里也删除`block`，重新定义高低水位。当新的区块来到时，如果是非主节点就会调用`recvPrePrepare()`,在其中针对`block`的`seqNo`在新的高低水位中进行判断；如果该节点是主节点，它就会调用`sendPrePrepare`，此时分配的`seqNo`就不在高低水位之间，就会触发`viewchange`，在`viewchange`的过程中会将`pbftcore.seqNo`更新为当前的低水位或者是`seqNo`就等于`nv.Xset`里存的`seqNo`的最大值。
* 在调用函数`movewatermarks()`的时候，如果`certstore`存储`block`的`seqNo`小于低水位就把`block`从数据库里删除，从`blockstore`里删除。
* `viewchange`结束后`blockstore`清空。`blockStore`里存储的始终是当前`view`下共识的区块，`pset`、`qset`里始终存储的是当前`view`下的`request`。所以如果再次发生`viewchange`，`blockStore`和`pset`、`qset`里的区块能正常进行对比，不会报错。

2、何时清空`pset`、`qset`、`certstore`?

在调用函数`movewatermarks()`的时候，如果`pset`、`qset`、`certstore`存储的`seqNo`小于低水位，就清空。

3、`pset`、`qset`能不能每当一个`block`记录到区块链上，就清空一次`pset`、`qset`？能不能不用`pset`、`qset`了？

不能。如果一个`block`处于`committed`状态后，就把它从`pset`、`qset`里删除。假如再此时停止了挖矿，再重新启动节点的时候，`pset`、`qset`为空，那么在`restoreState()`的时候就不能从`pset`、`qset`里面获得最新的关于`view`和`seqNo`的信息，结果导致`seqNo`、`view`又从0开始。在`fabric`里面，除了第一次启动节点时会出现`pset`、`qset`为空的情况，在接下来的启动中不会出现`pset`、`qset`为空的情况，因为在进行`movewatermarks(n)`的时候，低水位的值计算是小于n的，也就是说`pset`、`qset`里面只会删除一部分关于`request`的信息，不会全删完。

4、在`fabric`里面：假设一种情况，有一个`RequestBatch1`进行共识，里面的请求都存储在了`pset`、`qset`里，一轮共识结束后，`pset`、`qset`里面并没有把`RequestBatch1`里的请求进行删除。然后又来了一个`RequestBatch2`,`RequestBatch1`里的请求也都存储在了`pset`、`qset`里面,进行共识的过程中发生了`viewchange`，此时会对`pset`、`qset`里面的请求重新进行共识，其中包括`RequestBatch1`里面的请求，对`RequestBatch1`里面的请求再重新进行共识有无必要？

有必要。`RequestBatch1`里面的请求共识达到`committed`状态，只能是说明在该节点上`RequestBatch1`里的请求得到了其他节点的认可，但是其他节点上`RequestBatch1`能否达到`committed`状态还不一定。当节点在一个`checkpoint`上收到`2f+1`条来自其他节点的`checkpoint`的信息的时候，说明`2f+1`个其他节点都完成了该`checkpoint`所在之前请求的共识，此时便可移动高低水位，把`checkpoint`之前的请求从`pset`、`qset`等里删除。

在`geth-pbtf`里面，当发生`viewchange`的时候，我们就没必要对`RequestBatch1`（这里相当于是`Block1`）重新进行共识了，因为一轮共识结束之后，我们就把该区块放在了区块链上了。