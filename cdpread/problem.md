# 阅读中发现的问题

> 版本303


### 问题1：怎么进行计算下推

比如有个sql select count(*) from t where t.a = "hello";

个人理解的单机的大概流程为，扫描kv里的所有数据，然后逐个判断每个数据是否符合t.a = "hello"，如果符合就返回
该数据，不符合则跳过，这一步是在kv上操作。

最后再计算所有数据的count，这一步是在db上操作。

这样肯定会有个问题，如果count的数据特别大，db就给捅死了。

那怎么进行分布式计算呢？

因为我只需要进行count计算并不需要数据的内容，kv是不是可以返回该节点计算的count，而不是返回数据。

并且扫描数据，每一次扫描肯定会有一次开销，是不是可以在kv先进行计算呢？

是不是可以理解，kv层也可以进行简单计算，该计算是由db层下推过去的。

---

### 问题2：计划

ast -> 逻辑计划 -> 物理计划 -> 执行

逻辑计划是干啥的

物理计划是干啥的

是一种数据库的通用执行方式吗？

是不是可以理解hbase也有，pg也有，kudu也有？

---

### 问题3：schema判断

为什么在buildInsert的时候要有这样的错误判断

```go
ts, ok := insert.Table.TableRefs.Left.(*ast.TableSource)
if !ok {
   return nil, infoschema.ErrTableNotExists.GenWithStackByArgs()
}
```

印象中前面已经进行schema的操作了

不明白的是这种返回错误意思.

---

### 问题4：GetSessionVars

里面有很多这样的代码

```go
e.ctx.GetSessionVars()
```

这是干嘛的呢?获得这个链接中的一些全局变量吗?

---

### 问题5：什么叫分组接口

kv.go中有2类接口，分别负责查询和插入

```go
type Retriever interface{}
type Mutator interface{}
```

然后又有一个新的接口，叫分组接口

```go
// RetrieverMutator is the interface that groups Retriever and Mutator interfaces.
type RetrieverMutator interface {
   Retriever
   Mutator
}
```

这样设计的目的是啥呢?不可能是包一下方便引用把？

---

### 问题6：WriteStmtBufs

这个地方是不是叫写入buf，为写入提供缓存功能，加速写入的意思？

```go
// WriteStmtBufs can be used by insert/replace/delete/update statement.
// TODO: use a common memory pool to replace this.
type WriteStmtBufs struct {
   // RowValBuf is used by tablecodec.EncodeRow, to reduce runtime.growslice.
   RowValBuf []byte
   // BufStore stores temp KVs for a row when executing insert statement.
   // We could reuse a BufStore for multiple rows of a session to reduce memory allocations.
   BufStore *kv.BufferStore
   // AddRowValues use to store temp insert rows value, to reduce memory allocations when importing data.
   AddRowValues []types.Datum
   // IndexValsBuf is used by index.FetchValues
   IndexValsBuf []types.Datum
   // IndexKeyBuf is used by index.GenIndexKey
   IndexKeyBuf []byte
}
```

这里有个todo，意思用个内存池可以加速写入,什么是内存池呢？

---

### 问题7：位运算

这里是为了性能,所以使用的位运算吗?

```go
func (p *preprocessor) Leave(in ast.Node) (out ast.Node, ok bool) {
   switch x := in.(type) {
   case *ast.CreateTableStmt:
      p.flag &= ^inCreateOrDropTable
      p.checkAutoIncrement(x)
      p.checkContainDotColumn(x)
```

这个 &= 没懂是啥意思

---

### 问题8：写入kv部分

最后跟到代码是将值组装成kv

并且调用一个set方法实现将数据put到kv存储中

最后会调用到 memdb_buffer 中的set方法

看了下,这个最终的实现是用的goleveldb的实现,虽然leveldb rocksdb底层都是skiplist

这里是就是直接用了leveldb的memtable的插入方式,为什么没用rocksdb的呢?还是说就是一样的.

kv层在Mutator接口里定义了set

下面有4个set的实现,想知道都是啥意思

```go
func (st *TxnState) Set(k kv.Key, v []byte) error {...}
```

```go
func (lmb *lazyMemBuffer) Set(key Key, value []byte) error {...}
```

```go
func (m *memDbBuffer) Set(k Key, v []byte) error {...}
```

```go
func (t *mockTxn) Set(k Key, v []byte) error {...}
```

```go
func (txn *tikvTxn) Set(k kv.Key, v []byte) error {...}
```

---

### 问题9：执行方法的含义

读代码到adapter.go 中的 Exec 方法中,有一块具体执行sql语句的代码

```go
// Special handle for "select for update statement" in pessimistic transaction.
if isPessimistic && a.isSelectForUpdate {
   return a.handlePessimisticSelectForUpdate(ctx, e) 
}
// If the executor doesn't return any result to the client, we execute it without delay.
if e.Schema().Len() == 0 {
   if isPessimistic {
      return nil, a.handlePessimisticDML(ctx, e) 
   }
   return a.handleNoDelayExecutor(ctx, e) 
} else if proj, ok := e.(*ProjectionExec); ok && proj.calculateNoDelay {
   // Currently this is only for the "DO" statement. Take "DO 1, @a=2;" as an example:
   // the Projection has two expressions and two columns in the schema, but we should
   // not return the result of the two expressions.
   return a.handleNoDelayExecutor(ctx, e) 
}
```

通过上下文,以及字面意思一共有如下这些具体的执行方法,但是不明白他们的具体意思?

```go
handlePessimisticSelectForUpdate()
handlePessimisticDML()
handleNoDelayExecutor()
```


