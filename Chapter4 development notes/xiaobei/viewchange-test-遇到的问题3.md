* 出现的问题：在由`view1`变换的`view2`的时候，`signer3`在处理`func (instance *pbftCore) processNewView2(nv *types.NewView) events.Event`时报错：
```
2017/11/21 22:04:31 Replica 2 accepting new-view to view 2
2017/11/21 22:04:31 Replica 2 stopping a running new view timer
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x8dff27]
```
