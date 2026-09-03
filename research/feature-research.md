---
type: Note
---
# Feature Research


1. Factor Definition
- 因子名称、频率、表达式
        - 数据输入、历史窗口、来源与迁移信息
1. Parameters
        - 研究区间、开发集/OOS 划分
        - 股票池及上市天数限制
        - 预测目标与周期
        - 采样/调仓频率
        - Winsorize、排序、缺失值和中性化设定
        - PIT 数据可用性假设
2. IC Analysis
        - IC、RankIC、ICIR、t 值
        - 置信区间和方向命中率
        - 不同预测周期的衰减表现
        - 开发集与 OOS 分开展示
3. Factor Distribution
        - 因子值分布直方图
        - 覆盖率与缺失比例
4. Signal Buckets
        - 十分组平均未来收益
        - Top/Bottom 5% 和 10% 尾部表现
        - 正负收益命中率
5. Robustness
        - 按时间段、流动性、波动率、价格和 VWAP 分桶
        - 各分桶 IC、RankIC、方向命中率
6. Stability
        - 日内 Rank 自相关
        - Top/Bottom 成分流失率
        - 相邻交易日 Rank 稳定性
7. Factor Correlation
        - 与其他因子的 Rank 相关性
        - 用于识别重复因子和互补信号
