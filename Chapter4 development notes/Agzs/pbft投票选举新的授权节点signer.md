
在`console`界面执行`pbft.Propose("0x....", true)`，实际调用`consensus/pbft`目录下的`api.go`中的`Propose()`函数，该函数如下：
```go
func (api *API) Propose(address common.Address, auth bool) {
	api.pbft.lock.Lock()
	defer api.pbft.lock.Unlock()

	api.pbft.proposals[address] = auth
}

```
该函数将在`pbft`的`proposals`成员变量中添加新的元素`(address, auth)`，该变量主要在`pbft.go`中的`Prepare()`函数中使用
```go
func (c *PBFT) Prepare(chain consensus.ChainReader, header *types.Header) error {
	// If the block isn't a checkpoint, cast a random vote (good enough for now)
	header.Coinbase = common.Address{}
	header.Nonce = types.BlockNonce{}

	number := header.Number.Uint64()

	// Assemble the voting snapshot to check which votes make sense
	snap, err := c.snapshot(chain, number-1, header.ParentHash, nil)
	...

	if number%c.config.Epoch != 0 {
		c.lock.RLock()

		// Gather all the proposals that make sense voting on
		addresses := make([]common.Address, 0, len(c.proposals))
		for address, authorize := range c.proposals {
			if snap.validVote(address, authorize) {
				addresses = append(addresses, address)
			}
		}
		// If there's pending proposals, cast a vote on them
		if len(addresses) > 0 {
			header.Coinbase = addresses[rand.Intn(len(addresses))]
			if c.proposals[header.Coinbase] {
				copy(header.Nonce[:], nonceAuthVote)
			} else {
				copy(header.Nonce[:], nonceDropVote)
			}
		}
                c.lock.RUnlock()
	}
        ...
}
```
`Prepare()`函数会对`proposals`进行处理，如果`proposals`不为空，便将`proposals`的信息添加到`addresses`切片中，然后随机从下标`[0,n)`中选取一个数，将其对应的元素值保存到`header.Coinbase`，然后后期只对`header.coinbase`进行处理。另外，将授权与撤销的标识通过`header.Nonce[:]`进行表示。该函数在`func (self *worker) commitNewWork()`中被调用。

**注：在pow机制中，`header.coinbase`是作为挖矿接收奖励的账户的，而在clique和pbft中该变量被用来保存授权投票地址。**

后期，会通过调用`func (c *PBFT) snapshot(chain consensus.ChainReader, number uint64, hash common.Hash, parents []*types.Header) (*Snapshot, error)`函数间接调用
`func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error)`，在`apply()`函数中对`header.Coinbase`进行处理:
```go
func (s *Snapshot) apply(headers []*types.Header) (*Snapshot, error) {
          ...
          for _, header := range headers {
		...

		// Tally up the new vote from the signer
		var authorize bool
		switch {
		case bytes.Compare(header.Nonce[:], nonceAuthVote) == 0:
			authorize = true
		case bytes.Compare(header.Nonce[:], nonceDropVote) == 0:
			authorize = false
		default:
			return nil, errInvalidVote
		}
		if snap.cast(header.Coinbase, authorize) {
			snap.Votes = append(snap.Votes, &Vote{
				Signer:    signer,
				Block:     number,
				Address:   header.Coinbase,
				Authorize: authorize,
			})
		}
		// If the vote passed, update the list of signers
		if tally := snap.Tally[header.Coinbase]; tally.Votes > len(snap.Signers)/2 {
			if tally.Authorize {
				snap.Signers[header.Coinbase] = struct{}{}
			} else {
				...
			}
			...
                }
	    }
}
```
从该函数可以看出，该函数通过调用`cast()`函数为`head.coinbase`投票
```go
// cast adds a new vote into the tally.
func (s *Snapshot) cast(address common.Address, authorize bool) bool {
	// Ensure the vote is meaningful
	if !s.validVote(address, authorize) {
		return false
	}
	// Cast the vote into an existing or new tally
	if old, ok := s.Tally[address]; ok {
		old.Votes++
		s.Tally[address] = old
	} else {
		s.Tally[address] = Tally{Authorize: authorize, Votes: 1}
	}
	return true
}
```
`cast()`函数会对`head.coinbase`进行验证，并为其投上一票，最终返回`true`给`apply()`函数，并在`apply()`函数中将投票信息保存到`snap.Votes`中，此后验证投票是否超过当前所有`signers`总数的一半，若超过一半，则该`address`成为新的`signer`(授权节点)，可参与挖矿。

需要注意的是，由于投票是当前有效的`signers`每人一票才能作数，所以：

如果在`clique`中的话，由于每个`signer`轮流挖块，所以至少需要新挖出`len(signers)/2`个区块，就能使被投票的`address`成为正式的`signer`(授权节点)；

如果在pbft中，该机制显然不行，因为只有每次发生`viewchange`时，才会切换主节点，然后新的主节点才能投上一票，因此至少需要发生`len(signers)/2`次`viewchange`，才能使被投票的`address`成为正式的`signer`(授权节点)，这显然不现实，该机制需要进一步修改。

=========================================================================

大体思路：由于每个block都是通过pbft共识后，才将其上链，pbft共识的过程就相当于进行了投票，所以将被授权的节点地址保存到`header.coinbase`中，然后进行共识，共识成功后的区块被放到链上，然后主节点广播新的区块；其他节点收到新的区块后，对其解析出`header.coinbase`，之前本来打算在每个节点收到NewBlockMsg标识的消息后，从Block中读取出header然后进行处理，后来阅读代码，发现关于授权投票在`snap.apply()`中进行，而该函数仅在`pbft.snapshot()`中被调用，而`pbft.snapshot()`又在各种验证函数中被调用，所以相当于在验证区块的时候，顺带着就处理了被授权节点的投票，所以我们只需要修改投票计数的规则就好了，将原来的一半以上的投票改成主节点的一票就可以了。

被授权节点处理过程：

* 1) 主节点通过`pbft.propose()`投票
* 2) 主节点将投票信息保存到`header.coinbase`，并为自己处理该投票
* 3) 主节点将`header`放到`block中`，`pbft`共识该区块
* 4) 共识成功后，区块上链，主节点广播新区块
* 5) 其他节点接收到新区块后，进行区块验证，顺便处理下投票
* 6) 所有节点都导入新的区块后，每个节点的`pbft.Signer`数组保存的授权节点都是相同的，即共同添加了新的授权节点。

其他节点也可以进行`pbft.propose()`操作，也可以存储到`header.coinbase`，但是由于无法进行挖矿，所以无法将`header`保存到`block`中。但是当发生`viewchange`时，该节点将会获得挖矿机会，由于`proposals`提案一直保存，所以此时会将投票信息中的授权节点的地址保存到`header.coinbase`中，然后执行后续操作，由于`viewchange`发生的不确定性，所以采用主节点进行投票授权。

可能存在问题：

1、主节点权限过高

2、共识过程只是对block的digest进行了验证，并未涉及内部具体数据的验证


