---
type: Note
---
# NN-1s-Ashare-Prediction

### 0810

- **实验目的：1s 维度只使用 OHLCV 数据，不使用订单簿数据**
- **方法：重做 features**
- 数据：范围 **2021-01-04 至 2024-12-31**。训练截至 2023-12-29，验证截至 2024-02-29，其后为测试集；标准化仅使用训练集拟合。
- 输入：1s 数据生成特征，10s cadence，过去 3600s，共 (360, 42)
- 架构：2 层 GRU、hidden 384、MLP 512×2、1,840,385 参数
- 训练：seed 7、AdamW、lr 2e-4、FP16、batch 128
- loss function：标准 pooled MSE
- 标签：未来约 10 分钟收益，entry/exit 使用 10 秒 VWAP
- Features：42 维 OHLCV-only
