# TiDB 源码阅读

> 版本303

目前的阅读情况

- 阅读中发现的问题 [点击跳转](./cdpread/problem.md)
- DML流程 [点击跳转](./cdpread/dml.md)

---

windows 中需要注意的地方

util/signal/signal_windows.go文件中 需要手动引下面这2个包 才能启动

```
"context"
"github.com/pingcap/tidb/util/logutil"
```
