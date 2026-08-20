---
type: Note
---
# Live Producer

### 0811

- 当前使用模型来源：exp-nn/day-pure-batch-1min，Seed 7，Checkpoint Iter 30000
- 时间范围：2024-03-01 ～ 2024-12-31（股票级测试集）
- IC：Pearson 0.1039；Rank IC 0.0642
- top 90% Long 胜率：81.46%
- bottom 10% Short 胜率：84.88%


主要问题：

所有信息都长得同样重要，所以真正重要的信息反而不突出。

颜色是异常和方向的资源，不要浪费在正常状态上。

**一个交易员打开页面 1 秒钟，他最想知道什么？现在预测谁？涨还是跌？还能不能信？预测到什么时候？现在实际走势怎么样？**

改为：

| 页面 | 第一问题 | 核心内容 |
| --- | --- | --- |
| **Live Terminal** | 现在发生什么？ | 当前方向、actionability、实时主图、当前窗口表现 |
| **Replay** | 今天预测怎么样？ | 决策记录、240min bar、Actual vs Forecast、Residual |
| **Research** | 模型长期靠谱吗？ | IC、Hit Rate、月度稳定性、Decile、Robustness |
| **System** | 系统正常吗？ | Runtime、Stream、Data Quality、latency、coverage |
