# FLP不可能原理

## 结论
在网络可靠、但允许节点失效（即便只有一个）的最小化异步模型系统中，不存在一个可以解决一致性问题的确定性共识算法
（No completely asynchronous consensus protocol can tolerate even a single unannounced process death）。

## 同步 VS. 异步
1. 同步：节点间时钟误差，网络延迟，节点执行存在上限。可以判断是宕机 还是 网络问题。即消息在一定时间内完成。
2. 异步：节点间时钟误差，网络延迟，节点执行都是任意长的。无法判断是宕机 还是 网络问题。

## 理论
FLP 不可能原理实际上说明对于允许节点失效情况下，纯粹异步系统无法确保共识在有限时间内完成。

## ### 参考
https://yeasy.gitbook.io/blockchain_guide/04_distributed_system/flp