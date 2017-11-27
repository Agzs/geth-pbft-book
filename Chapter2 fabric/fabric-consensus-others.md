在`consensus`中还定义了一个跟共识机制关系不大, 但是在工程上比较重要的接口, `StatePersistor`. 该接口可以将一些键值对储存起来, 用于恢复一些由于进程崩溃等导致的错误. 在fabric中, 是通过`db`来实现的, 定义在`consensus/helper/persist/persist.go`. (在`persist.go`重新定义了一个空结构体`Helper`, 在该结构体上实现了接口)

暂且应该用不到处理这类东西, 所以在这里只是提到该接口, 不具体分析.

## `StatePersistor`
> StatePersistor is used to store consensus state which should survive a process crash

```go
type StatePersistor interface {
	StoreState(key string, value []byte) error
	ReadState(key string) ([]byte, error)
	ReadStateSet(prefix string) (map[string][]byte, error)
	DelState(key string)
}
```
```go
// Helper provides an abstraction to access the Persist column family
// in the database.
type Helper struct{}

// StoreState stores a key,value pair
func (h *Helper) StoreState(key string, value []byte) error {
	db := db.GetDBHandle()
	return db.Put(db.PersistCF, []byte("consensus."+key), value)
}

// DelState removes a key,value pair
func (h *Helper) DelState(key string) {
	db := db.GetDBHandle()
	db.Delete(db.PersistCF, []byte("consensus."+key))
}

// ReadState retrieves a value to a key
func (h *Helper) ReadState(key string) ([]byte, error) {
	db := db.GetDBHandle()
	return db.Get(db.PersistCF, []byte("consensus."+key))
}

// ReadStateSet retrieves all key,value pairs where the key starts with prefix
func (h *Helper) ReadStateSet(prefix string) (map[string][]byte, error) {
	db := db.GetDBHandle()
	prefixRaw := []byte("consensus." + prefix)

	ret := make(map[string][]byte)
	it := db.GetIterator(db.PersistCF)
	defer it.Close()
	for it.Seek(prefixRaw); it.ValidForPrefix(prefixRaw); it.Next() {
		key := string(it.Key().Data())
		key = key[len("consensus."):]
		// copy data from the slice!
		ret[key] = append([]byte(nil), it.Value().Data()...)
	}
	return ret, nil
}
```