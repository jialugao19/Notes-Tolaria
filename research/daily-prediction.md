---
type: Note
_width: wide
---
# NN-Overnight-Ashare-Prediction

## Baseline

日期：0913 - 0920

- 数据: A 股全市场 1min bar, 信号日期范围为 2021-01-04 至 2025-12-31. 排除北交所, ST 区间, 停牌区间, 上市 60 天内股票和退市整理期.
  - Train: 2021-01-04 至 2023-12-29, 726 个有效训练日期.
  - Validation: 2024 全年, 排除 9 月和 10 月, 共 205 个 prediction dates.
  - Test: 2025 全年, 124 个 evaluable rows. 
- 样本与标签: 每个样本是一个 stock-day. Signal 截止 D 13:58, D 14:00 使用 1min VWAP 买入. 无有效成交时使用该 bar 的 Close.
  - Planned exit 为下一交易日 10:00 的 1min VWAP.
  - 若不可卖, 寻找最早连续 10 个 sellable trading-minute bars, 使用窗口 sum(Amount) / sum(Vol) 卖出.
  - 一字涨停可卖并使用 up_limit; 跌停需要实际炸板. 搜索可跨交易日, 午休不中断连续性.
  - Realized return 为 (exit_price *exit_adj_factor) / (entry_price* entry_adj_factor) - 1.
  - 若退市前始终无法退出, terminal value 为 0, realized return 为 -1.
  - 训练 target y_cs 是每个 signal date 内对 raw adjusted realized return 做 cross-sectional z-score 后的值.
- 输入序列:
  - Minute branch: 120 个 scheduled minute bars, 对齐 MinuteIndex=59..178, 最后一个 token 为 D 13:58.
  - Daily branch: 20 个交易日, 对齐 D-20..D-1.
  - Minute input shape: (120, 71).
  - Daily input shape: (20, 82).
  - 午休不生成 placeholder. Sequence 不使用 padding 或 mask. 任一 required factor 缺失时删除整个 stock-day.
- 标准化:
  - Stock factors: 每个 timestamp 在当时 eligible universe 内做 cross-sectional population z-score.
  - Market factors: 使用 train period 的 population mean 和 std.
  - Target: 每个 signal date 做 cross-sectional population z-score.
  - 不进行 winsorization, clipping, rank transform 或 per-stock time-series normalization.
  - Population std 为 0 时, 对应 normalized channel 设为 0.
- 特征: 共 4 组, 153 个 channels. Minute 和 daily 分别输入, 不拼成单个 153D tensor.
  - Minute stock, 50 个: close_log_1bar_continue, close_log_5bar_continue, close_log_20bar_continue, close_log_60bar_continue, close_log_120bar_continue,
    - close_from_session_open, close_from_bar_open_log, close_to_session_vwap, close_to_session_high, close_to_session_low, bar_clv, close_to_high_5bar, close_to_low_5bar,
    - close_to_high_20bar, close_to_low_20bar, close_to_high_60bar, close_to_low_60bar, log_range_5bar, log_range_20bar, log_range_60bar, log_range_120bar, return_std_10bar,
    - return_std_30bar, rv_5bar, rv_20bar, rv_60bar, rv_120bar, total_trend_5bar, total_trend_20bar, total_trend_60bar, total_trend_120bar, return_shell_5_20bar,
    - return_shell_20_60bar, return_shell_60_120bar, rv_shell_5_20bar, rv_shell_20_60bar, rv_shell_60_120bar, log_vol_mean_1bar_reset, log_amount_mean_1bar_reset,
    - log_vol_mean_5bar_reset, log_amount_mean_5bar_reset, vol_1bar_to_mean_5bar, vol_sum_5bar_to_mean_30bar, vol_pct_5bar, amount_pct_5bar, vol_pct_20bar, amount_pct_20bar,
    - vol_pct_60bar, amount_pct_60bar, amount_return_corr_30bar.
  - Minute market, 21 个: eligible_equal_return_1bar, eligible_equal_return_5bar, eligible_equal_return_20bar, eligible_equal_return_60bar, eligible_equal_return_120bar,
    - eligible_equal_rv_20bar, eligible_equal_rv_60bar, csi300_return_10bar, csi300_return_60bar, csi300_return_120bar, csi500_return_10bar, csi500_return_60bar,
    - csi500_return_120bar, csi1000_return_10bar, csi1000_return_60bar, csi1000_return_120bar, csi300_rv_60bar, csi500_rv_60bar, csi1000_rv_60bar, stock_return_60bar_std,
    - positive_return_1bar_equal.
  - Daily stock, 52 个: close_log_1day, close_log_5day, close_log_20day, close_log_60day, log_range_5day, log_range_20day, log_range_60day, close_to_high_5day,
    - close_to_low_5day, close_to_high_20day, close_to_low_20day, close_to_high_60day, close_to_low_60day, rv_5day, rv_20day, rv_60day, total_trend_5day, total_trend_20day,
    - total_trend_60day, close_pct_20day, close_pct_60day, close_to_ma_5day, close_to_ma_20day, close_to_ma_60day, ema10_to_ma5_minus_1, sharpe_20day, sharpe_60day,
    - vol_60_minus_20day, sharpe_60_minus_20day, vol_log_mean_5day, amount_log_mean_5day, vol_log_mean_20day, amount_log_mean_20day, vol_log_mean_60day, amount_log_mean_60day,
    - vol_5_to_20day, amount_5_to_20day, vol_20_to_60day, amount_20_to_60day, return_to_vol_corr_20day, return_to_amount_corr_20day, range_to_vol_corr_20day,
    - range_to_amount_corr_20day, return_to_vol_corr_60day, return_to_amount_corr_60day, range_to_vol_corr_60day, range_to_amount_corr_60day, return_skew_20day,
    - return_kurt_20day, return_skew_60day, return_kurt_60day, signed_amount_slope_mean_21day.
  - Daily market, 30 个: eligible_equal_return_1day, eligible_equal_return_5day, eligible_equal_return_20day, eligible_equal_return_60day, eligible_equal_rv_20day,
    - eligible_equal_rv_60day, csi300_return_1day, csi300_return_5day, csi300_return_20day, csi300_return_60day, csi500_return_1day, csi500_return_5day, csi500_return_20day,
    - csi500_return_60day, csi1000_return_1day, csi1000_return_5day, csi1000_return_20day, csi1000_return_60day, csi300_rv_20day, csi300_rv_60day, csi500_rv_20day,
    - csi500_rv_60day, csi1000_rv_20day, csi1000_rv_60day, stock_return_1day_std, stock_return_5day_std, stock_return_20day_std, positive_return_1day_equal,
    - positive_return_5day_equal, positive_return_20day_equal.
- 模型: DualGRU, minute/daily late fusion.
  - Minute encoder: 1-layer unidirectional GRU, input size 71, hidden size 64.
  - Daily encoder: 1-layer unidirectional GRU, input size 82, hidden size 64.
  - 两个 final hidden 分别经过 LayerNorm, concatenate 为 128D.
  - Prediction head: Linear(128,64) -> GELU -> Dropout(0.1) -> Linear(64,1).
  - 无 bidirectional, attention, residual GRU 或多层 encoder.
  - 总参数量: 63,297.
- Loss: date-balanced SmoothL1, beta=1.0.
  - 每个 signal date 的 sample weight 为 1 / N_D.
  - 每个日期对完整 objective 的总权重相同.
  - 不使用 pairwise, listwise 或 differentiable RankIC loss.
- Batch sampler:
  - Batch size 为 512 stock-days.
  - 每个 epoch 对全部 441,574 个 trainable samples 无放回访问一次.
  - 先按 monthly shard 保持 mmap locality, batch 不跨 shard, shard order 和 shard 内 block/row order 每个 epoch deterministic shuffle.
  - 不使用 replacement, oversampling 或 WeightedRandomSampler.
  - 不 drop tail. 当前实际为 881 batches/epoch, batch size 范围约 483 至 512.
- 最新已确认训练方案, 尚未实现:
  - Seed 42.
  - AdamW, learning rate 3e-4, weight decay 1e-4.
  - 固定 learning rate, 不使用 scheduler.
  - FP32, 单张 GPU, gradient accumulation 为 1.
  - Global gradient clipping 为 1.0.
  - 最多 2 epochs, 即最多 883,148 次 sample visits 和 1,762 optimizer steps.
  - 每个 epoch 做 10 次完整 validation, 即约每 0.1 epoch 一次.
  - 第 1 epoch 的 validation steps 为 89, 177, 265, 353, 441, 529, 617, 705, 793, 881.
  - 至少训练满 1 epoch 后才能 early stop.
  - Early-stopping patience 为 5 次 validation.
  - DataLoader 为 4 workers, prefetch_factor=2, pinned memory, persistent workers, deterministic algorithms.
- Model selection:
  - Primary: validation daily mean RankIC, 越高越好.
  - Secondary: validation cohort Q10-Q1, 越高越好.
  - 每次 validation 都形成候选 checkpoint.
  - 最终重新加载 best checkpoint, 完整生成 validation predictions 和 daily metrics.
  - Test 2025 不参与 checkpoint selection.
- checkpoint 方案:
  - 每次 0.1-epoch validation 后保存一次, 最多约 20 个 checkpoints.
  - 保存 model, optimizer, global step, epoch index, next batch index, best metrics, early-stop state, history, Python/NumPy/PyTorch CPU/CUDA RNG state, Git/config/data
- 实验跟踪与调度:
  - MLflow canonical endpoint: [http://host.docker.internal:5000](http://host.docker.internal:5000).
  - MLflow experiment: astock-overnight, 当前尚未创建正式 run.
  - MLflow backend: PostgreSQL + MinIO, health check 已通过.
  - 正式训练必须通过 Prefect deployment experiment-runner/m-nas.
  - Prefect API: [http://host.docker.internal:4201/api](http://host.docker.internal:4201/api).
  - 宿主机 UI: [http://127.0.0.1:4201](http://127.0.0.1:4201).
  - Prefect deployment 当前为 READY, GPU 0/1/2 均有 limit=1 的 concurrency control.
  - 正式 experiment ID: astock-overnight-core-v1.
  - 完成后向 MLflow 上传 config, manifest, history, validation outputs 和全部 checkpoints.
  - 项目内计划新增 experiment-registry.yaml, 不登记到 /home/maomao/exp-nn/0-select&report.


##  Baseline 2: 13:50 cutoff

日期：1015-1017

| Signal | Minute sequence | Daily sequence | Trainable samples | Best checkpoint | Best Validation RankIC | Experiment | Evaluation |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 13:58 | 59..178 (120 bars) | D-20..D-1 (20 trade dates) | 441,574 | step 617 | 0.147947 | 20260918T082705Z | 20260918T141822Z |
| 13:50 | 51..170 (120 bars) | D-20..D-1 (20 trade dates) | 451,876 | step 630 | 0.133224 | 20261015T162946Z | 20261015T164233Z |


- Train: 2021-2023；Validation: 2024 且排除 9-10 月；Test: 2025 one-shot。
- 13:58 minute sequence 为 59..178；13:50 为 51..170。两者均为 120 bars。Daily sequence 均为 D-20..D-1。
- Stock/target 使用 cross-sectional population z-score；market factors 只用 Train statistics；不 winsorize。
- Label 固定为 adjusted realized return：D 14:00 entry 到 next trade date 10:00，无法卖出时沿用既定 delayed-exit state machine。
- Model 均为 63,297-parameter dual-GRU late fusion，training config 与 seed 42 完全一致，各自按 Validation RankIC 选择 checkpoint。

结果分析：发现了 1 个实际的 look-ahead bias, 以及 1 个未实际触发的结构性风险.

- 实际问题: Validation ledger 纳入了 purged_boundary positions. 2024-08-30 的收益跨入排除的 9-10 月, 2024-12-31 的收益跨入 2025 Test. 
- 结构性风险: 旧代码曾使用 finite realized_return 过滤 entry candidates, 这依赖未来 outcome availability. Frozen ledgers 中没有 candidate 因此被删除, 所以未产生额外实证偏差. 现
  已改为先固定 entry set, 再对 return completeness fail fast.


### 实验 1：样本留存率过低问题的解决

- 数据：A 股 1 分钟及日线数据。Signal 为 2021-01-04 至 2024-12-31; Train 截至 2023-12-29, 共 726 个交易日; Validation 为 2024-01 至 08 月及 11 至 12 月, 共 205 个交易日。2024-09 至 10 月排除, 没有运行 2025 signal Test。
- 输入：每个 stock-day 输入截至 13:50 的 120 根 1 分钟序列；以及 D-20 至 D-1 的 20 日序列。每个时间点把个股特征与共享市场特征拼接。
  - baseline：分钟个股 50、分钟市场 21、日线个股 52、日线市场 30, 合计 153 个。
    - 分钟组包含价格位置、趋势、波动和成交量等; 日线组还包含相关性、偏度、峰度和流动性冲击等。
    - 市场组包含等权市场与 CSI 指数的收益、波动和市场宽度。
  - drop_factor 删除分钟个股的 bar_clv, 合计 152 个。
  - drop_family 删除日线个股的 signed_amount_slope_mean_21day, 也为 152 个。
- 标签：计划在 D 日 14:00 入场、下一交易日 10:00 出场; 无法按计划卖出时按 sellable 规则延迟。原始标签是复权执行价收益 exit_adjusted_price / entry_adjusted_price - 1, 训练目标再按信号日做截面 z-score。
- 标准化：个股特征按每个 timestamp 的合格股票截面做 population z-score; 市场特征的 population 均值和标准差只用 Train 日期拟合，再应用到 Validation; 目标按每个 signal date截面标准化。
- 模型：DualGRU，用两个独立的 GRU 看两种时间尺度: 一个读取股票过去 120 根 1 分钟 bar 的特征，另一个读取过去 20 个交易日的特征，hidden size 各 64，各接 LayerNorm。模型将它们拼接，经过一个小型 MLP，即 Linear(128,64) → GELU → Dropout(0.1) → Linear(64,1)，输出该股票的单个预测分数，用于比较同日股票的相对表现。本次 baseline 模型约 6.33 万参数。
- loss function：SmoothL1，beta=1。
- 训练：seed 42, AdamW, LR 3e-4, weight decay 1e-4, batch size 512, 日期均衡加权的 SmoothL1 loss (beta=1), gradient clip 1.0。单张 RTX 3080, FP32, 每个 batch 执行一次 optimizer step; 没有 AMP、梯度累积或学习率 scheduler。最多 2 epoch; 至少训练 1 epoch 后, Validation RankIC 连续 5 次未改善则 early stop。
- Sampler：先随机排列月份, 再在各月内随机排列 batch blocks 和样本; 每个 epoch 无放回覆盖 Train 样本。
- Checkpoint：每个 epoch 验证 10 次, 并在每个验证边界保存 checkpoint。Checkpoint 保存模型、optimizer、精确到下一 batch 的训练进度、选择指标及 Python/NumPy/PyTorch CPU/CUDA RNG; 因为没有 scheduler 或 AMP, 也没有相应 state。身份校验绑定配置、dataset content hash 和 Git commit。
- **结果：**





## 交易策略研究问题

日期：1014 - 

- **selection breadth**
- **weighting**
- **liquidity and capacity**
- **transaction-cost frontier**
- **entry/exit realization and alpha decay**
- **risk and exposure decomposition**
