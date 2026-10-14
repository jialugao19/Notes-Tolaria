---
type: Note
---
# Feature Research

### 单因子报告大纲

总结段落

1. 因子名称和公式
  - 因子名称、表达式
  - 要求的数据、频率（d/min）
2. IC 分析
  - 【数值】IC、RankIC、ICIR、t 值
  - 【图】不同预测周期的衰减表现（min: 5/10/15/20/25/30）
3. 分布
  - 【图】因子值分布直方图
  - 【数值】覆盖率与缺失比例
4. 分组表现
  - 【数值】Top/Bottom 5% 和 10% 尾部表现：方向命中率；实际收益
  - 【表】十分组平均未来实际收益；方向命中率
5. 稳健性
  - 【图】按时间段分桶：IC 和平均未来实际收益（分 2 个纵坐标轴）
  - 【图】按价格分桶：IC 和平均未来实际收益（分 2 个纵坐标轴）
  - 【图】按 VWAP 分桶：IC 和平均未来实际收益（分 2 个纵坐标轴）
  - 【图】按波动率分桶：IC 和平均未来实际收益（分 2 个纵坐标轴）


### 多因子对比报告

1. **Comparison Setup**
  - 对比因子、研究区间、股票池
  - 统一预测目标与 horizon
  - 统一预处理、Dev / OOS 划分
2. **Factor Leaderboard**
  - IC / RankIC / ICIR / t 值
  - OOS 表现
  - Coverage、Turnover、稳定性
  - 与已有因子的最大相关性
3. **Predictive Power**
  - IC / RankIC 横向比较
  - IC 分布与置信区间
  - Dev vs OOS 对比
4. **Horizon & Decay**
  - 不同预测周期下的 RankIC
  - 衰减曲线
  - 有效预测周期
5. **Robustness**
  - 按时间、流动性、波动率、价格、日内时段分组比较
  - 各环境下 IC / RankIC
6. **Stability & Turnover**
  - Rolling IC
  - Rank 自相关
  - Top / Bottom 成分稳定性
  - Signal turnover
7. **Signal Shape**
  - Quantile return
  - Top / Bottom tail 表现
  - 单调性比较
8. **Factor Dependence**
  - 因子 Rank correlation matrix
  - Rolling correlation
  - Factor clustering / family
9. **Incremental Value**
  - Orthogonalized / Residual IC
  - 相对已有因子库的新增预测能力
  - 冗余因子识别
10. **Factor Scorecard**
  - Predictiveness
  - OOS robustness
  - Stability
  - Uniqueness
  - Trading usability
  - 最终状态：Core / Candidate / Redundant / Conditional / Rejected
