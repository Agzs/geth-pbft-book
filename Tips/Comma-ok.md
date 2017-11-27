### interface变量存储的类型

我们知道interface的变量里面可以存储任意类型的数值(该类型实现了interface)。那么我们怎么反向知道这个变量里面实际保存了的是哪个类型的对象呢？目前常用的方法：

- Comma-ok断言

	Go语言里面有一个语法，可以直接判断是否是该类型的变量： value, ok = element.(T)，这里value就是变量的值，ok是一个bool类型，element是interface变量，T是断言的类型。

	如果element里面确实存储了T类型的数值，那么ok返回true，否则返回false。

	比如eth/backend.go中的Ethereum.StartMining():
```Go
func (s *Ethereum) StartMining(local bool) error {
	...
	if clique, ok := s.engine.(*clique.Clique); ok {
		...
	}
	...
}
```

        再比如eth/handler.go

```Go
func (pm *ProtocolManager) handle(p *peer) error {
	...
	if rw, ok := p.rw.(*meteredMsgReadWriter); ok {
		...
	}
	...
}
```
