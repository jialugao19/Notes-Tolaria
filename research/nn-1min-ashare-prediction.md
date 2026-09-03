---
type: Note
---
# NN-1min-Ashare-Prediction

## Data

### Batch Sampler =

- **实验目的：加入截面信息。**
- **方法：1 个 batch 只包含 1 个交易日。**
- 数据：A 股分钟行情，范围 **2021-01-04 至 2024-12-31**。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集；标准化仅使用训练集拟合。
- 输入与标签：每个样本包含连续 60 根 1 分钟 bar、14 个特征；标签为 log\_close[t+10] - log\_close[t+1]。
- 模型：GruMlpRegressor，两层 GRU、hidden size 384，后接 [512, 512] MLP，输出单个收益预测，约 181 万参数。
- 训练：seed 7，AdamW，学习率 1e-3，batch size 8192，FP16 AMP，单张 GPU，梯度累积为 1。
- 核心改动：每个 batch 只包含一个交易日；交易日按有效序列数加权、有放回抽取；batch 内样本不重复，跨 batch 允许重复。
- 训练预算：总抽样行数等于一轮训练集的有效序列数，即“一个等价 epoch”。
- Checkpoint：每 10,000 step 保存，并保存精确终点 checkpoint；包含模型、优化器、scheduler、AMP scaler 和采样恢复位置。
- **结果：最佳 checkpoint：iter\_30000；测试集 pooled Pearson IC：0.103878；测试集 pooled Rank IC：0.064245**
- 日期：0810

###


### **label 分布和 features 分布研究 =**

- **label 分布尖峰厚尾、零点堆积、正偏**
  - 约 15-19% 股票在 10 分钟内收盘价没有变化
  - **零样本尖峰：基本是真正的零收益，不是小收益被显示精度或histogram分箱抹成了零。raw label 是精确零。**
    - 全程为 0
    - 波动后回到原价格
  - 零点处不仅有尖峰，两侧还有凹陷和宽肩，可能是由于价格最小跳动单位，一旦价格发生变化，就会变动超过 1 个 tick
  - 厚尾：峰度 23-51
  - train-val-test 存在明显尺度漂移，train 标准差为 1.000，val 标准差为 1.479（2024年2月初），test 标准差为1.189（2024年9月底至10月）
  - P.S.：样本的处理，只保留 60-bar输入窗口有效；63个features全部有限；label有限
  - **影响：**
    - **厚尾对 MSE 的影响很大，3% 的样本对 train MSE 有超过 50% 的贡献。**
    - **零点堆积虽然有 18%，但对 loss 几乎没有影响，对样本数和 batch 组成有影响。**
    - **MSE 实际上在拟合高波动和极端样本，而这部分的 regime 也是最容易变化的。**
  - **建议：尝试Huber loss**

### **label 尺度改变 =**

- **实验目的：预测 10min、15min、30min 这三个 label 尺度，对比 IC、胜率的变化**
- **方法：重构 label，共用 features**
- 数据：A 股分钟行情，范围 **2021-01-04 至 2024-12-31**。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集。
- 输入与标签：每个样本包含连续 60 根 1 分钟 bar、47 个特征；标签为 **log\_close[t+horizon+1] - log\_close[t+1]**；zscore 标准化，仅使用训练集
  - 特征：基础特征 14 + 相对强弱/流动性 9 + 波动/区间状态 12 + 趋势/收益状态 12，即63 个特征 - 市场环境 16（指数收益 12 + 指数 RV 3 + 市场 dispersion 1）
- 模型：GruMlpRegressor，两层 GRU、hidden size 384，后接 [512, 512] MLP，输出单个收益预测，约 186 万参数。
- 训练：seed 7，AdamW，学习率 1e-3，batch size 8192，FP16 AMP，单张 GPU，梯度累积为 1。学习率调度为 200 step linear warmup，随后 cosine decay，最低学习率 1e-6。
- Batch sampler：全局随机排列，并按该排列无放回取样
- 训练预算：约 727 个交易日 × 每天约 4,700 只股票 × 每只股票每天约 160 个有效窗口 ≈ 5.5 亿个训练序列。batch size 为 8,192，保留最后 4,233 个样本的尾 batch，共有 67,615 个 optimizer step。
- Checkpoint：每 10,000 step 保存，并保存精确终点 checkpoint；包含模型、优化器、scheduler、AMP scaler 和采样恢复位置。
- 日期：0814-0816
- **结果**
  - Horizon Pooled IC Pooled RankIC Top10 胜率 Top-Bottom spread 10min 0.0767 0.0504 41.87% 8.80 bps 15min 0.0912 0.0675 55.63% 9.68 bps 30min 0.0904 0.0708 42.36% 12.10 bps
    - 特征统计量只由训练集有限值计算，三个 horizon 完全共享。标签按 horizon 独立计算，并分别使用训练集监督端点做 z-score.


### 开盘数据处理

- **实验目的：测试跨日期的 rolling 怎么计算比较好，是按照 close-open 拼接这一段比较好，还是删除比较好，还是接受这里有跳空比较好**
- **方法：跳空** g\_d=\\log(Open\_d)-\\log(Close\_{d-1})
- 数据:A股分钟行情，范围2021-01-04至2024-12-31。训练截至2023-12-29，验证截至2024-02-29，其后为测试集。
- 输入与标签: 每个样本包含全市场股票，连续60根1分钟bar, 47个特征;标签为log\_close[t+11]- log\_close[t+1];zscore 标准化，仅使用训练集
- 模型: GruMlpRegressor,两层 GRU,hidden size 384,后接[512,512]MLP,输出单益预测，约186万参数。
- 训练: seed 7,AdamW,学习率1e-3,batch size 8192,FP16 AMP,单张GPU,梯度累积为1。学习率调度为200 step linear warmup,随后cosine decay,最低学习率1e-6
- Batch sampler:全局随机排列，并按该排列无放回取样
- 训练预算:约727个交易日x每天约4,700只股票x每只股票每天约160个有效窗口。batch size为8,192，保留最后零散个样本的尾batch，共有？个 optimizer step
- Checkpoint:每10,000 step保存，并保存精确终点checkpoint;包含模型、优化器、 scheduler、AMP scaler复位置。
- 日期：0815-

## Feature

### **增加 features 数量 =**

- **实验目的：增加截面信息和更长周期 features**
- **方法：增加截面 features**
- 新增特征：在原有 14 个特征基础上追加 49 个特征，总计 63 个。
  - 市场特征：成分股权重构建 CSI300、CSI500、CSI1000 指数分钟收益代理，分别计算 10/60/120/240 分钟收益率和 60 分钟 realized volatility；另计算全市场个股 60 分钟收益率的横截面标准差。
  - 相对特征：个股 60 分钟收益率在同一分钟全市场横截面中的 percentile rank。
  - 流动性 surprise：当前 volume/amount 相对该股票相同日内分钟、过去最多 60 个交易日历史值的分位数，当前日不进入历史窗口，至少需要 20 个历史观测；以及当前 volume/amount 相对前 5/20/60 个连续交易 bar 的分位数。
  - 多尺度状态：新增 realized volatility、range、trend、return 四类状态特征，horizon 为 5/20/60/120/240/480 分钟。保留原有 14 个特征，其中包括 1/2/3/5/10 分钟收益率等短周期特征。
- 数据：A 股分钟行情，范围 **2021-01-04 至 2024-12-31**。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集；标准化仅使用训练集拟合。
- 输入与标签：每个样本包含连续 60 根 1 分钟 bar、63 个特征；标签为 log\_close[t+11] - log\_close[t+1]。
- 模型：GruMlpRegressor，两层 GRU、hidden size 384，后接 [512, 512] MLP，输出单个收益预测，约 186 万参数。
- 训练：seed 7，AdamW，学习率 1e-3，batch size 8192，FP16 AMP，单张 GPU，梯度累积为 1。学习率调度为 200 step linear warmup，随后 cosine decay，最低学习率 1e-6。
- Batch sampler：随机抽取不放回。
- 训练预算：总抽样行数等于一轮训练集的有效序列数，即“一个等价 epoch”。
- Checkpoint：每 10,000 step 保存，并保存精确终点 checkpoint；包含模型、优化器、scheduler、AMP scaler 和采样恢复位置。
- **结果：最佳 checkpoint：iter\_30000；test pooled Pearson IC：-0.002262；test pooled Rank IC：0.000743。**
- **结果分析 1：泛化问题。**
  - 在第 10,000 step，模型只消费了约 14.8% 的训练行，train IC 已达到 0.35，而 val RankIC 为负。
  - 可能的原因：
    - 5.5 亿样本并不等于 5.5 亿独立样本，同一股票每天约 161 个窗口，相邻输入窗口共享 59/60 根 bar，相邻标签也高度重叠。
    - 模型可能迅速学到了训练期的共同市场和状态模式。每个 batch 几乎覆盖全部 727 个训练日，模型能很快学习训练期的平均关系。尤其是新增特征中包含大量市场收益与市场波动；互相高度相关的多周期 return/trend/range/volatility；长周期状态和流动性状态。
    - 当前 pooled IC 容易被时间和市场共同因子抬高
    - MSE 与最终目标不完全一致。MSE 会更重视大收益和高波动样本。仅做 z-score 并不能消除厚尾，模型容易拟合少数极端状态。
- **结果分析 2：不仅排序关系消失，输出尺度和均值也发生了明显的样本外漂移。**
- **下一步：考虑设置 train–val gap gate，检查灾难性分叉**
  - 条件：10% 样本量训练结束时 train\_val\_rank\_ic\_gap >= 0.15
- **结果分析 3：发现错误**
  - Feature：2024-01-03 起变成 raw scale，因此 validation 有 97.31% 行、test 有 100% 行使用了错误尺度的模型输入。
  - Label：2024-03-12 起变成 raw scale，却仍被当作 z-score 再逆变换。test 有 170,117,430 行、占 96.58% 被错误缩小约 237 倍。
- **修改报告模板、报告生成代码及报告本身：**
  1. 增加 label 分布图，放在 features 分布图之后，单开一小节。
  2. 增加 train global ic 随 checkpoint 变化图
  3. grad norm不要clip p99
- 日期：0811-0812
- **结果：最佳 checkpoint：iter\_10000；test pooled Pearson IC：0.061496；test pooled Rank IC：0.051659；test top 90% long win rate：44.43%；test bottom 10% short win rate：50.34%；test top 90% long return：+4.7405 bps；test bottom 10% short return：+4.2364 bps。**
  - 注：top 90% 指预测值位于第 90 百分位以上的 top 10%；bottom 10% 标的平均收益为 −4.2364 bps，因此做空收益记为正值。


### Feature Ablation =

- **实验目的：分组检验 63 个特征的效果**
- **方法：分组分别训练。**
- 分组：
  1. matched control：仅前 14 个旧特征。
  2. 市场环境 16：指数收益 12 + 指数 RV 3 + 市场 dispersion 1
  3. 相对强弱/流动性 9：stock rank 1 + same-minute liquidity 2 + trailing liquidity 6
  4. 波动/区间状态 12：realized-vol 6 + range 6
  5. 趋势/收益状态 12：trend 6 + return 6
- 数据：A 股分钟行情，范围 **2021-01-04 至 2024-12-31**。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集。
- 输入与标签：每个样本包含连续 60 根 1 分钟 bar、63 个特征；标签为 log\_close[t+11] - log\_close[t+1]；zscore 标准化，仅使用训练集
- 模型：GruMlpRegressor，两层 GRU、hidden size 384，后接 [512, 512] MLP，输出单个收益预测，约 186 万参数。
- 训练：seed 7，AdamW，学习率 1e-3，batch size 8192，FP16 AMP，单张 GPU，梯度累积为 1。学习率调度为 200 step linear warmup，随后 cosine decay，最低学习率 1e-6。
- Batch sampler：全局随机排列，并按该排列无放回取样
- 训练预算：约 727 个交易日 × 每天约 4,700 只股票 × 每只股票每天约 160 个有效窗口 ≈ 5.5 亿个训练序列。batch size 为 8,192，保留最后 4,233 个样本的尾 batch，共有 67,615 个 optimizer step。
- Checkpoint：每 10,000 step 保存，并保存精确终点 checkpoint；包含模型、优化器、scheduler、AMP scaler 和采样恢复位置。
- **结果：**
  - Full63 0.061496 0.051659 0.348526 iter 10000 44.4274% 50.3411% +4.7405 bps +4.2363 bps
  - legacy14 0.089951 0.059541 0.126565 iter 40000 45.5264% 49.6887% +6.9926 bps +6.3839 bps
  - market16 — — 0.341112 iter 6762 门控 — — — —
  - relative/liquidity9 0.098183 0.075886 0.124739 iter 10000 46.9943% 50.2264% +8.7375 bps +7.3752 bps
  - volatility/range12 0.097144 0.064713 0.145214 iter 50000 45.4957% 50.6258% +7.5012 bps +7.0354 bps
  - trend/return12 0.096908 0.067951 0.154549 iter 30000 46.1291% 50.2885% +8.0789 bps +6.9293 bps
    - Train IC 是固定 train-report sample 指标；market16 的值是在门控 checkpoint 上测得，未读取 test。
    - 按 prediction ≥ P90（最高 10%）和 prediction ≤ P10（最低 10%）统计；收益单位为 bps，做空收益已反转符号。

![image.png](attachments/1786679763712-image.png)

- 日期：0812-0814

### 引入市场横截面信息 1 =

- \*\*实验目的：比较三种市场信息注入方式对模型效果的影响。\*\*传统单股票序列、同一交易时点联合包含全市场合格股票的 panel，以及按时间戳读取的工程化市场环境特征。重点检验全市场横截面上下文能否改善样本外预测能力。
- 方法：分组。
  1. baseline: 单股票序列样本。输入为 47 维股票特征与 16 维零值市场特征; 模型中的panel 维度 N=1，因此 leave-one-out mean/std 上下文严格为零。
  2. panel\_mean\_std: 同一日期、同一分钟的全部合格股票组成一个 panel。每只股票先独立经过共享 GRU 编码，再计算排除自身的 panel 编码均值和标准差，并与自身编码拼接。16维市场特征保持为零。
  3. market\_environment\_16: 保持传统单股票序列样本，但在每根 bar 上拼接按时间戳读取的真实 16 维市场环境特征。panel 维度仍为 N=1，因此 leave-one-out mean/std 上下文为零。
- 数据：A 股分钟行情，范围 2021-01-04 至 2024-12-31。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集。
- 输入与标签：每只股票输入连续 60 根 1 分钟 bar，每根 bar 包含 47 个特征。标签定义为：log\_close[t+horizon+1] - log\_close[t+1]
  - 当前 horizon=10。
  - 标准化：pooled z-score，均只使用训练集拟合均值和标准差。
- 特征：
  - 基础特征 14 + 相对强弱/流动性 9 + 波动/区间状态 12 + 趋势/收益状态 12
  - 市场特征16: 指数收益 12 + 指数已实现波动率 3 + 市场横截面 dispersion 1，
- 模型：GruMlpRegressor，两层 GRU、hidden size 384；每只股票的 GRU 编码与leave-one-out mean/std 上下文拼接，后接 [512, 512] MLP，输出单个收益预测。
  - 上下文统计使用FP32计算。
  - 模型共约 225 万参数。
- 训练：seed 7，AdamW，学习率 1e-3，MSE loss，FP16 AMP，单张 GPU，梯度累积为 1。
  - 学习率调度为 200 step linear warmup，随后 cosine decay，最低学习率 1e-6。
- Batch sampler：首先对全部同日期、同分钟 panel 做全局随机排列，并按该排列无放回取样。
  - panel\_mean\_std 每步读取一个完整 panel，模型输入形状为 (1，N\_g，60,63)。
  - baseline 和 market\_environment\_16 使用同一套独立于 panel 顺序的全局行随机排列，并按照相同的 N\_g 序列切分，模型输入形状为 (N\_g,1，60，63)。
  - batch size：每个optimizer step 处理一个变长的 panel-sized股票集合，有效**batch size 为该 step 对应的合格股票数 N\_g，为全局随机排列后第 g 个panel中包含的有效股票数量，随日期、停牌情况和数据有效性变化**，大约 4400-4700。三臂每一步包含相同数量的股票，拥有相同的 step-size 序列和 optimizer step 数，并在一个 epoch 内恰好覆盖全部训练行一次。MSE loss 在每个 step 内对 N\_g 个预测取均值。
- 训练预算：约 727 个交易日 × 每天约 4,700 只股票 × 每只股票每天约 160 个有效窗口 ≈ 5.5 亿个训练序列。
  - optimizer step 数等于训练集中的实际 panel 数，预计约为 727 × 160 ≈ 11.6 万 step
- Checkpoint：每完成约 5% 训练保存一次，并保证保存精确终点 S；每 10% 验证一次并包含终点。
  - 包含模型、优化器、scheduler、AMP scaler 和采样恢复位置、Python/NumPy/PyTorch CPU 与 CUDA RNG 状态，以及配置、代码、数据 artifact、panel fingerprint 和模型身份信息。
- 实验跟踪与调度：
  - 训练通过 MLflow experiment market-sample 记录配置、指标和 provenance，tracking endpoint 为 <[http://127.0.0.1:15000，不可用时失败关闭。](http://127.0.0.1:15000%EF%BC%8C%E4%B8%8D%E5%8F%AF%E7%94%A8%E6%97%B6%E5%A4%B1%E8%B4%A5%E5%85%B3%E9%97%AD%E3%80%82)
  - 正式任务通过现有> Prefectexperiment-runner/m-nas 调度：数据重建阶段不申请 GPU，两组训练分别使用单张 GPU。容器内 Prefect API 为[http://host.docker.internal:4201/api；宿主机](http://host.docker.internal:4201/api%EF%BC%9B%E5%AE%BF%E4%B8%BB%E6%9C%BA) UI 地址为 [http://127.0.0.1:4201](http://127.0.0.1:4201)
- 结果分析：
  - Baseline：测试 Pearson IC 0.0879，Rank IC 0.0781。只使用个股序列，跨股票上下文严格为零。
  - Panel mean/std：测试 Pearson IC 0.1260，Rank IC 0.1028，较 baseline 分别提升约 43.3% 和 31.7%，是唯一在验证集和测试集都明显改善的方案。
  - Market16：测试 Pearson IC 0.0722，Rank IC 0.0602，较 baseline 分别下降约 17.9% 和 22.9%。
  - 原因：
    - 把个股编码与其余股票编码的均值和标准差比较。
    - 这些统计是在 GRU 学到的高维编码上计算的。
    - Market16 表现较差则可能因为它对所有股票提供相同的低维市场摘要，没有相对位置，并且手工指数特征可能存在冗余、噪声、 regime shift。
- Panel mean/std 的优点
  - **第一步：GRU 负责回答「这只股票过去 60 分钟发生了什么？」
    第二步：mean/std 负责回答「它和此刻整个市场相比怎么样？」**

  - GRU 给每只股票分别做时序摘要。过去 60 分钟经过 GRU 提炼之后的状态。
    最终 MLP 实际看到 [z\_i,\\mu\_{-i},\\sigma\_{-i}]，三个都是 384 维，同时知道个股、市场、市场分化。

  - 共享 GRU 参数，每只股票都被映射到同一个表示空间； GRU 的表示空间会主动适应后面的横截面聚合方式。
  - 对比股票之间的 Attention：用关系表达能力换取计算效率和规模扩展能力。
- 日期：0820-0825
- 下一步计划：
1. Latent z-position (e\_i-μ\_-i)/(σ\_-i+ε)
2. 原始特征横截面 rank 每分钟 percentile/rank
- 增加：  横截面排序                       原值
- stock\_ret\_5m\_cs\_rank        state\_return\_5m
  stock\_ret\_20m\_cs\_rank       state\_return\_20m
  stock\_ret\_240m\_cs\_rank      state\_return\_240m
  stock\_rv\_20m\_cs\_rank        state\_realized\_vol\_20m
  stock\_rv\_60m\_cs\_rank        state\_realized\_vol\_60m
  stock\_trend\_60m\_cs\_rank     state\_trend\_60m
  stock\_log\_amount\_cs\_rank    log\_amount

- 现有特征中，相关的是 stock\_ret\_60m\_cs\_rank：在同一 Date + MinuteIndex 下，将每只股票的 60 分钟收益放到当时全市场股票中计算百分位排名。
1. 分层位置 全市场、行业、规模/流动性组内位置
2. Learned set token Deep Sets 聚合全市场分布
- 把固定 mean/std 替换为可学习的集合聚合：
  c = mean\_j φ(e\_j)
  prediction\_i = ρ(e\_i, c)

- 这种结构保持股票排列不变性，同时允许模型自己学习哪些横截面统计有用；是当前 mean/std 的自然推广。Deep Sets ([https://arxiv.org/abs/1703.06114)](<https://arxiv.org/abs/1703.06114)%E7%BB%99%E5%87%BA%E4%BA%86%E8%BF%99%E7%B1%BB>)
1. Induced attention 少量 latent token 与全市场交互
- 16–64 个 inducing/market token，将复杂度降到近似 O(N\_gm)。
- 这类结构能够表达 mean/std 无法描述的多峰分布和股票簇。Set Transformer ([https://proceedings.mlr.press/v97/lee19d.html](https://proceedings.mlr.press/v97/lee19d.html))


### Features 分析 =

- 日期：0825
- minute\_norm, session\_minute\_norm
- OHLCVA
  - signed log return 1m, 5m, 2m, 3m, 10m
  - tanh log hl, tanh log oc
  - log vol, log\_amount
  - log\_vol\_5m\_mean, log\_amount\_5m\_mean
- Volatility
  - Realized volatility 5m, 60m, 120m, 240m, 480m
  - tanh var\_10m
  - High-low range 5m, 20m, 60m, 120m, 240m, 480m
  - OLS total trend 5m, 20m, 60m, 120m, 240m, 480m,
  - Endpoint return 5m, 20m, 60m, 120m, 240m, 480m
- Liquidity
  - cs\_rank
  - stock\_ret\_60m vol, amount
  - vol\_trailing\_percentile 5m, 20m, 60m 5m, 20m, 60m
- Market
  - csi300/csi500/csi1000 log return 10m, 60m, 120m, 240m
  - realized volatility 60m
  - market\_cs\_ret\_60m\_dispersion
- 分析以上 Features 的相关性、可用性
  - V1：基于现有 cache/artifact 的 Feature Audit Report
    V2：基于 10m label 的 richer factor diagnostics
    V3：Multi-horizon factor research
    V4：候选 feature set matched ablation / model validation

  - 实际上执行了 V1+V2
- 结果：保留 5 组、36 个：
  - price\_action（7）：ret\_1m, ret\_2m, ret\_3m, ret\_5m, ret\_10m, hl, oc
  - liquidity\_activity（8）：log\_vol, log\_amount, log\_vol\_5m\_mean, log\_amount\_5m\_mean, vol\_same\_minute\_60d\_pct, vol\_trailing\_5m\_pct, vol\_trailing\_20m\_pct, amount\_trailing\_60m\_pct
  - time\_cross\_section（3）：minute\_norm, session\_minute\_norm, stock\_ret\_60m\_cs\_rank
  - volatility\_range（13）：vol\_10m，6 个 realized-vol shell（5m 核心及 5–20/20–60/60–120/120–240/240–480），以及 6 个 range（5/20/60/120/240/480m）
  - return\_state（5）：5–20/20–60/60–120/120–240/240–480m 非重叠 return shell

作为 new baseline 重跑

- 数据：A 股 1 分钟行情，2021-01-04–2024-12-31；train 截至 2023-12-29，validation 截至 2024-02-29，之后为 test。交易日为 train/val/test = 727/37/205。
  - 输入/标签：每个样本是单只股票、单个日内终点之前的 60 根 1 分钟 bar，窗口不跨交易日；需要跨日历史的特征可以使用过去交易日。每根 bar 为 36 维。标签为 log\_close[t+11] - log\_close[t+1]，即 entry lag=1、horizon=10。
  - 标准化：feature 与 label 都是 pooled z-score；均值、标准差仅由 train split 拟合，ddof=0、float64 累加，输出 float32。
- 特征：36 个；price\_action（7）+ liquidity\_activity（8）+ time\_cross\_section（3）+ time\_cross\_section（3）+ volatility\_range（13）+ return\_state（5）
- 模型：当前实现是两层单向 GRU，hidden size 384，随后 [512,512] MLP 输出 1 个预测；精确参数量 1,833,473，约 183.35 万。
- 训练：seed 7，AdamW，lr 1e-3，MSE，CUDA FP16 autocast + GradScaler，单 GPU，梯度累积 1；200 optimizer-step linear warmup，之后 cosine decay 到 1e-6。
  - batch sampler：当前是对全部有效“股票 × 窗口终点”样本做一个确定性的全局随机排列，一轮内无放回。
  - 预算：暂估 67,615 optimizer steps。
- Checkpoint：每 5% 保存 checkpoint、每 10% 做 validation，并都包含精确终点
  - 包含 model、optimizer、scheduler、AMP scaler 和 sampler 的下一 batch 位置；
  - 当前没有保存 Python/NumPy/PyTorch CPU/CUDA RNG 状态。
  - 配置/代码/data/model 身份分布在 run manifest 与 checkpoint sidecar 中，不是全部嵌入 .pt。因此您给出的 checkpoint 描述目前也不能原样采用。
- MLflow：
  - 服务已启动，[http://127.0.0.1:15000/health](http://127.0.0.1:15000/health) 返回 OK，
  - prediction-NN-2 experiment 可查询；MLflow run。
  - 训练时记录 loss/loss\_mean、lr、AMP scale、梯度范数、批次日期混合度、各阶段耗时，以及 validation IC/Rank IC/MSE、top/bottom decile return、hit rate、CSI300/CSI500 IC 和 checkpoint selection 摘要。
- Prefect：
  - 宿主机 API 健康，deployment experiment-runner/m-nas 已注册且状态 READY，最后轮询正常；
  - gpu-0/1/2 并发限制均 active、limit=1、占用 0。
  - 宿主机 UI：[http://127.0.0.1:4201](http://127.0.0.1:4201%E3%80%82)

再次选择：保留 34 个


## Model

### 引入市场横截面信息 2 =

- **实验目的：进行引入横截面信息的多个不同方向的实验**
  - baseline
  - Latent z-position (e\_i-μ\_-i)/(σ\_-i+ε)
  - 原始特征横截面 rank 每分钟 percentile/rank：return/RV/amount/volume; 1/5/10/20/60
  - Learned set token Deep Sets 聚合全市场分布
    把固定 mean/std 替换为可学习的集合聚合：
    c = mean\_j φ(e\_j)
    prediction\_i = ρ(e\_i, c)
    Deep Sets ([https://arxiv.org/abs/1703.06114](https://arxiv.org/abs/1703.06114))

  - Induced attention 少量 latent token 与全市场交互，16–64 个 inducing/market token
    这类结构能够表达 mean/std 无法描述的多峰分布和股票簇。
    Set Transformer ([https://proceedings.mlr.press/v97/lee19d.html](https://proceedings.mlr.press/v97/lee19d.html))

- 实验结果
- Control 34D：Test IC 0.08646，RankIC 0.06417
  CS Rank 41D：Test IC 0.12329，RankIC 0.09283；7 个显式横截面 rank 有明显增量
  Latent Z 34D Test IC 0.12734，RankIC 0.10245；
  Deep Sets 34D Test IC 0.12103，RankIC 0.09098；
  Induced Attention 34D 预测退化为常数，训练暂停；

- 结果分析
  - 保留 7 个 cs rank 因子，作为baseline
  - latent z \\ deep sets 作为可选项 on-off 保留
- 日期：0827-0902


### De-mean & Deep Sets

- 实验目的：测试 41 因子 + latent z \\ deep sets 是否能改善表现。




## Infrastructure

### 重构 **1min 模型代码 =**

- **baseline-1min-0813**
- MR
- 难点回顾：1. 旧代码存放和何时删除。2. 如何检查代码一致性。3. 不要求代码和产物完全一样的情况下，如何确保业务逻辑的正确性。4. commit & push & merge 的计划
- 日期：0812-0820




## 想法

0810：

- 修改 features，加入更长周期的 features

0811：

- 测试 label（close-close; vwap-vwap）；log close[t+11] - log close[t+1]
- 测试：拼接开盘；去掉开盘

0812

- 在研究报告中加入正确性可视化约束：train/val/test 的标准化对比
- 研究报告中所有位置加入 direction accuracy
- GRU MLP CNN Transformer，基本的 block 只有这些，考虑组合以增加模型参数量

0813

- loss function 很重要，一定要调整
- 涨停幅度 10% 和 20% 样本是否要分开？
- 保管结果很重要，检查保管过程；gitlab 远程提交

0814

- prefect 调度
- push 笔记到远程
- 全局 qmodel-run/qmodel-submit 的 editable 映射确实已失效

0820

- 修改 exp-nn/AGENTS.md 作为项目整体规划。
  - 基本参数模板
  - 实验列表（考虑是否需要中）

0827

- 目前的 features 已经人为把过去 OHLCV path 压缩成 return / RV 等，NN 无法恢复路径。
  - 考虑 state features，同时额外给 NN 一段 **raw/reduced-resolution history**
  - 不合理，弃之

0830

- 重做研究网页，按 data-model-... 划分研究
- skill：连接 github 仓库，必要时可阅读：精简文档；完整研究报告
- evaluation report 修改：胜率；不同期限5/10/20min；改成预测 3-12 且去掉 9/10 两个月
