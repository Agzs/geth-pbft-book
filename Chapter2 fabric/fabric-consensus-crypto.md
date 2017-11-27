# Crypto
## `SecurityUtils`
> SecurityUtils is used to access the sign/verify methods from the crypto package.
```go
type SecurityUtils interface {
	Sign(msg []byte) ([]byte, error)
	Verify(peerID *pb.PeerID, signature []byte, message []byte) error
}
```
`SecurityUtils`同时也是consensus.go中定义的唯一跟消息加密有关的接口.

### Usage
和`NetworkStack`类似, 其也只有作为`Stack`一部分被使用, 没有结构体单独实现这个接口.

### Implement
注释中提到, `Stack`除了`helper.Helper`还有一处实现, 是`helper.ConsensusHandler`. 但分析发现`ConsensusHandler`其实并没有实现`Stack`, 也没有被作为`Stack`来使用. 所以此处以及下文的`Stack`分析, 单指`helper.Helper`处的实现.
> ConsensusHandler handles consensus messages. It also implements the Stack.

在`Helper`中, 可以看到, `Helper`其实也是通过调用成员变量`secHelper`的函数来实现的.
```go
// Sign a message with this validator's signing key
func (h *Helper) Sign(msg []byte) ([]byte, error) {
	if h.secOn {
		return h.secHelper.Sign(msg)
	}
	logger.Debug("Security is disabled")
	return msg, nil
}
```
`secHelper`的类型为`crypto.Peer`, 定义在`core/crypto`内. 可以看到内部是通过调用ECDSA加密算法, 以`key`和`msg`作为参数, 进行加密. 验证过程类似. 两个过程是用的都是`EnrollmentKey`.
```go
func (node *nodeImpl) sign(signKey interface{}, msg []byte) ([]byte, error) {
	return primitives.ECDSASign(signKey, msg)
}

func (node *nodeImpl) signWithEnrollmentKey(msg []byte) ([]byte, error) {
	return primitives.ECDSASign(node.enrollPrivKey, msg)
}

func (node *nodeImpl) ecdsaSignWithEnrollmentKey(msg []byte) (*big.Int, *big.Int, error) {
	return primitives.ECDSASignDirect(node.enrollPrivKey, msg)
}

func (node *nodeImpl) verify(verKey interface{}, msg, signature []byte) (bool, error) {
	return primitives.ECDSAVerify(verKey, msg, signature)
}

func (node *nodeImpl) verifyWithEnrollmentCert(msg, signature []byte) (bool, error) {
	return primitives.ECDSAVerify(node.enrollCert.PublicKey, msg, signature)
}
```

### Summary
`SecurityUtils`接口仅作为`Stack`的一部分使用, `Stack`是由`Helper`实现的, 对于`SecurityUtils`部分, `Helper`通过调用`secHelper`来实现, `secHelper`类型为`crypto.Peer`, 内部采用ECDSA算法.