---
type: Note
---
# Route

## 1.666 ETF route

- 数据源: Tushare ProMax。
- 核心计算: score=slope\*r^2 ^3 
- 运行时间:每日 22:00 Asia/Shanghai。
- 输出:Prefect Markdown artifact.
- 通知:成功结果或失败diagnostics 通过 ServerChan 推送。

## 2. 新增 US stock route

- Universe：38个Core，12 个 Satellite，SPY、 QQQ 个 Benchmark.
- Authoritative data: Alpaca SIP, 1Day + adjustment=all.
- Secondary verification: yfinance, 验有 52 个 symbol 最近 5 个 session.
- 数据门槛:必须完整取得最后 30个market session,不允许使用stale data。
- 计算公式:与当前ETFroute完全一致，即 score=slope\*r^2 ^3 。
- 输出:
  - Overall Core Top 10 Best Hold,SPY, QQQ benchmark score.
  - Technology/Growth、 Diversifier、 Satellite 分组排名.
  - 相对上一交易日的 rank change。
  - 完整 snapshot、 report 和 ranking state。
- 调度:工作日 18:00 America/New_York。休市日: 生成skipped artifact，不读取行情、不写ranking、不发送成功通知。
- 通知:成功发送concise summary;失败发送非敏感 diagnostics。
