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


项目目标是用 TDX 行情生成全市场特征, 驱动三只 ETF 的 V2 shadow 预测, 不执行交易.

- 已完成: 固定历史选源, 重建截至 9 月 22 日的 75 个交易日归档, 完成历史回放并部署修复版本.
- 核心问题: 预测通道仍用 dated-history 查询当天成交, 当日数据为空; realtime 有可用节点, 但尚未完成与历史口径的一致性验收.
- 代码缺陷: 全空归档仍可能被标记 complete, 存在污染后续缓存的风险.
- 当前判断: 已部署, 尚未证明能够正常持续预测.
- 建议下一步: 修复归档完整性门禁, 验证“同节点实时采集 + 历史查询”, 通过全市场对账和盘中验收后接入预测. 保留现有 checkpoint
