---
name: sector-momentum-scanner
description: A股主流板块ETF动量扫描 — 捕捉板块轮动信号，识别启动/加速/衰竭阶段，辅助判断持仓板块机会流失与新机会浮现。
---

# 板块动量扫描仪

## 触发场景

- 嵌入 fund-estimate 日报末尾（周一至周五 14:30）
- 用户主动询问"今天有哪些板块在动"
- 独立 cron 每日 9:00 推送早盘预览（可选）

**不要用于**：选基推荐、买卖决策的唯一依据。

---

## 监控板块列表

| 板块 | ETF代码 | 前缀 | 备注 |
|------|---------|------|------|
| 半导体 | 159813 | sz | |
| 机器人 | 159770 | sz | |
| 人工智能 | 159819 | sz | |
| 通信设备 | 563020 | sh | |
| 有色金属 | 512400 | sh | |
| 有色金属 | 560860 | sh | 双重覆盖 |
| 新能源 | 516160 | sh | |
| 卫星通信 | 159206 | sz | |
| 红利低波 | 515100 | sh | |
| 电网设备 | 560660 | sh | |
| 沪深300 | 000300 | sh | 基准 |

---

## 数据获取

**新浪财经 ETF K线接口**：
```bash
curl -s "https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData?symbol={prefix}{code}&scale=240&ma=5&datalen=90"
```

返回字段：`day`, `open`, `high`, `low`, `close`, `volume`, `ma_price5`

---

## 核心指标计算

### RSI(14) — Wilder平滑

```python
def calc_rsi(prices, period=14):
    gains = [max(prices[i]-prices[i-1], 0) for i in range(1, len(prices))]
    losses = [max(prices[i-1]-prices[i], 0) for i in range(1, len(prices))]
    avg_gain = sum(gains[:period]) / period
    avg_loss = sum(losses[:period]) / period
    for i in range(period, len(gains)):
        avg_gain = (avg_gain * (period - 1) + gains[i]) / period
        avg_loss = (avg_loss * (period - 1) + losses[i]) / period
    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    return round(100 - (100 / (1 + rs)), 2)
```

### MACD(12,26,9)

```python
def calc_ema(prices, n):
    k = 2 / (n + 1)
    ema = [None] * (n - 1)
    ema.append(sum(prices[:n]) / n)
    for i in range(n, len(prices)):
        ema.append(prices[i] * k + ema[-1] * (1 - k))
    return ema

ema12 = calc_ema(closes, 12)
ema26 = calc_ema(closes, 26)
macd_line = [e12 - e26 for e12, e26 in zip(ema12, ema26) if e12 is not None and e26 is not None]
signal_line = calc_ema([m for m in macd_line if m is not None], 9)
macd_hist = [m - s for m, s in zip(macd_line, signal_line) if m is not None and s is not None]
```

### 历史分位

```python
sorted_closes = sorted(window)
percentile = sum(1 for p in sorted_closes if p < current) / len(sorted_closes) * 100
```

### 赔率

```python
upside = (high_90 - current) / current * 100
downside = (current - low_90) / current * 100
reward_ratio = upside / downside
```

---

## 信号判断规则

### 板块异动分级

| 级别 | 触发条件 | 含义 |
|------|---------|------|
| 🔴 **启动信号** | MACD金叉（柱由负转正）+ RSI > 50 + 成交量放大（>均量1.2倍） | 板块开始第一波上攻 |
| 🟡 **趋势确认** | 启动信号 + 20日新高 | 趋势确立，可关注 |
| 🟢 **加速信号** | RSI从55上穿70 + 赔率>1.5 + 均线多头排列 | 板块进入主升浪 |
| ⚪ **高位整固** | RSI > 70 持续3天以上 | 强势但注意回调风险 |
| 🔵 **机会流失** | 持仓板块RSI < 40 + MACD死叉 + 均线空头 | 原有趋势告破 |
| 🟤 **衰竭信号** | RSI从70+快速回落至50以下 + 成交量萎缩 | 趋势可能反转 |

### 通用阈值

| 指标 | 低位 | 中性 | 高位 |
|------|------|------|------|
| RSI(14) | < 45 | 45–55 | > 55 |
| 历史分位 | < 40% | 40–70% | > 70% |
| 赔率 | < 1:1 | 1–2:1 | > 2:1 |

---

## 输出格式

```markdown
### 🚀 板块动量扫描

**扫描时间**：YYYY-MM-DD HH:MM | 覆盖11只主流ETF

| 板块 | 收盘 | RSI | 历史分位 | 赔率 | MACD | 20日新高 | 信号 |
|------|------|-----|---------|------|------|---------|------|
| 半导体 | 1.618 | 78.6 | 89% | 0.8:1 | ★金叉 | ▲ | 🟢加速 |
| ... | | | | | | | |

**今日异动**：[列出触发🔴启动信号的板块]
**持仓提醒**：[列出⚠️机会流失的持仓相关板块]
**重点关注**：[给出2–3个板块的操作参考]
```

---

## K线延迟验证

⚠️ 获取K线后立即检查 `klines[-1]["day"]` 与今日差距。
- 差距 ≤ 1天：数据实时，信号可靠
- 差距 2–3天：数据参考用，不作为启动信号触发
- 差距 > 3天：跳过扫描，标注"数据延迟"

---

## 高风险动作约束

- ✅ Agent 可执行：计算指标、输出信号、生成参考建议
- ⚠️ 必须人工确认：任何买卖操作

---

## 参考文档

| 文件 | 内容 |
|------|------|
| `references/semiconductor-rally-2026.md` | 2026年4月半导体行情复盘：MACD金叉（4/8）比20日新高（4/14）提前6天，验证信号体系优先级 |
| `references/etf-technical-analysis.md` | ETF技术分析框架（历史分位/赔率/RSI/均线系统/K线结构）— 可与 fund-estimate 共用 |
