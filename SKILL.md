---
name: sector-momentum-scanner
description: A股主流板块ETF动量扫描 — 捕捉板块轮动信号，识别启动/加速/衰竭/机会流失阶段，辅助判断持仓板块轮动节奏。输出为关注/观察/状态提醒，不构成任何买卖指令。
---

# 板块动量扫描仪 v2.0

## 合规声明与使用边界

**输出性质**：本技能产出的所有信号（启动观察、趋势增强、动量加速、相对走弱等）均为**板块轮动状态的客观描述**，仅供辅助判断板块轮动节奏与持仓健康度，**不构成任何投资建议**，不能作为买卖决策的依据。

**使用者责任**：任何人依据本技能输出进行实际投资操作或交易操作，后果自负。

**不适用的场景**：
- 选基推荐
- 择时决策
- 单一指标驱动决策
- 作为唯一的风控依据

---

## 触发场景

| 场景 | 触发条件 | 输出侧重点 |
|------|---------|-----------|
| 日报嵌入 | 周一至周五 14:30，fund-estimate 日报末尾 | 持仓关联、相对强弱、排名变化 |
| 用户主动询问 | 用户问"今天有哪些板块在动" | 全量排名、今日信号、重点关注 |
| 持仓检查 | 用户问"我的 XX 板块怎么样了" | 单一板块深度信号 + 趋势阶段判定 |
| 早盘预览（可选） | cron 每日 9:00 推送 | 前日信号持续性、隔夜外盘影响 |

---

## 一、输入与输出边界

### 输入边界

| 项目 | 要求 |
|------|------|
| 数据源 | 新浪财经 ETF K 线接口（见 §数据获取） |
| 时间范围 | 最近 90 个交易日日线（scale=240, datalen=90） |
| 基准对比 | 沪深300（000300）作为绝对基准 |
| 最小样本 | RSI 计算至少 20 根；MACD 计算至少 35 根；20 日新高判断至少 35 根 |
| 交易日判断 | 以获取到的最新 K 线日期与今日之差判断；非交易日不生成新信号 |

### 输出边界

| 项目 | 说明 |
|------|------|
| 信号类型 | 关注/观察/状态提醒，不输出任何买卖指令 |
| 信号粒度 | 按板块输出，不针对具体 ETF 发出差异化信号 |
| 重复板块 | 同一板块有多个 ETF 跟踪时（如有色金属 512400 + 560860），取 RSI 较高的那只为代表，输出时两者均列出但信号归并 |
| 字段类型 | 所有数值字段统一保留 2 位小数；相对强度单位为百分比（%）；排名为正整数 |

---

## 二、ETF 监控列表

| 板块 | ETF代码 | 前缀 | 双重覆盖 | 备注 |
|------|---------|------|---------|------|
| 半导体 | 159813 | sz | — | |
| 机器人 | 159770 | sz | — | |
| 人工智能 | 159819 | sz | — | |
| 通信设备 | 563020 | sh | — | |
| 有色金属 | 512400 | sh | 是 | 双重覆盖 |
| 有色金属 | 560860 | sh | 是 | 双重覆盖 |
| 新能源 | 516160 | sh | — | |
| 卫星通信 | 159206 | sz | — | |
| 红利低波 | 515100 | sh | — | |
| 电网设备 | 560660 | sh | — | |
| 沪深300 | 000300 | sh | — | **基准，不可缺失** |

> **双重覆盖去重规则**：同一板块多只 ETF 时，先按"数据延迟优先排除"原则过滤，再取 RSI 较高的作为代表信号输出。结果中同时列出所有跟踪该板块的 ETF 代码。

---

## 三、数据获取与校验

### 接口

```
GET https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData?symbol={prefix}{code}&scale=240&ma=5&datalen=90
```

返回字段（均为字符串，需 `float()` 转换）：`day`, `open`, `high`, `low`, `close`, `volume`, `ma_price5`

### 数据异常处理规则（优先级顺序）

| # | 异常情况 | 判断标准 | 处理方式 |
|---|---------|---------|---------|
| 1 | 接口请求失败 | HTTP 非 200 或空响应或超时（>10s） | 该板块标记「❌数据获取失败」，不生成任何信号，不参与排名 |
| 2 | K 线根数不足 MACD | < 35 根 | 不计算 MACD；其他指标正常输出；不触发 MACD 相关信号 |
| 3 | K 线根数不足 RSI | < 20 根 | 不计算 RSI；其他指标正常输出 |
| 4 | K 线根数不足 20 日新高 | < 35 根 | 不判断 20 日新高 |
| 5 | **数据严重延迟** | 最新 K 线日期距今 ≥ 2 个自然日 | 标记「⚠️数据延迟」；**禁止触发 🔴启动观察、🟡趋势增强、🟢动量加速**；其他信号正常输出但加注延迟说明；若无明确历史信号则不生成新标签，仅提示数据延迟 |
| 6 | 单只 ETF 数据异常 | 上述任意一条 | 只影响该 ETF；其他 ETF 正常参与排名 |

### 交易日与非交易日处理

| 情况 | 行为 |
|------|------|
| 9:00–14:30 交易日 | 实时扫描，数据延迟判断以当前时间对比最新 K 线日期 |
| 非交易日（周末/节假日） | 若最新 K 线日期与昨日（或最近交易日）一致，视为正常；若一致且时间为非交易时段，不生成新信号但保留昨日信号供参考 |
| 盘中（9:30–11:30, 13:00–15:00）| 数据为最新，信号正常输出 |
| 盘后（15:00–次日 9:00）| 数据为今日收盘，可生成当日最终信号 |

---

## 四、基础指标计算

> 所有收盘价数组为 `closes = [close_0, close_1, ..., close_N]`，其中 `close_0` 为最旧数据，`close_N` 为最新数据。

### RSI(14) — Wilder 平滑法

```python
def calc_rsi(closes, period=14):
    if len(closes) < period + 1:
        return None
    gains = [max(closes[i] - closes[i-1], 0) for i in range(1, len(closes))]
    losses = [max(closes[i-1] - closes[i], 0) for i in range(1, len(closes))]
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
def calc_ema(data, n):
    k = 2 / (n + 1)
    ema = [None] * (n - 1)
    ema.append(sum(data[:n]) / n)
    for i in range(n, len(data)):
        ema.append(data[i] * k + ema[-1] * (1 - k))
    return ema

def calc_macd(closes):
    if len(closes) < 35:
        return None, None, None
    ema12 = calc_ema(closes, 12)
    ema26 = calc_ema(closes, 26)
    macd_line = [e12 - e26 for e12, e26 in zip(ema12, ema26) if e12 is not None and e26 is not None]
    signal_line = calc_ema([m for m in macd_line if m is not None], 9)
    macd_hist = [m - s for m, s in zip(macd_line, signal_line) if m is not None and s is not None]
    # 最新值
    curr_macd = macd_line[-1] if macd_line else None
    curr_signal = signal_line[-1] if signal_line else None
    curr_hist = macd_hist[-1] if macd_hist else None
    prev_hist = macd_hist[-2] if len(macd_hist) >= 2 else None
    return curr_macd, curr_signal, curr_hist, prev_hist

def macd_crossover(curr_hist, prev_hist):
    """MACD 金叉：柱由负转正"""
    if curr_hist is None or prev_hist is None:
        return False
    return prev_hist < 0 and curr_hist >= 0

def macd_deathcross(curr_hist, prev_hist):
    """MACD 死叉：柱由正转负"""
    if curr_hist is None or prev_hist is None:
        return False
    return prev_hist >= 0 and curr_hist < 0
```

### 历史分位（90 日窗口）

```python
def calc_percentile(closes, window=90):
    if len(closes) < 2:
        return None
    window_closes = closes[-window:] if len(closes) >= window else closes
    current = closes[-1]
    percentile = sum(1 for p in window_closes if p < current) / len(window_closes) * 100
    return round(percentile, 2)
```

### 赔率（90 日窗口）

```python
def calc_reward_ratio(closes, window=90):
    if len(closes) < 2:
        return None
    window_closes = closes[-window:] if len(closes) >= window else closes
    current = closes[-1]
    high_90 = max(window_closes)
    low_90 = min(window_closes)
    if low_90 == 0:
        return None
    upside = (high_90 - current) / current * 100
    downside = (current - low_90) / current * 100
    if downside == 0:
        return None
    return round(upside / downside, 2)
```

### 成交量放大倍数

```python
def calc_volume_ratio(volumes):
    """最新成交量 / 20 日均量"""
    if len(volumes) < 20:
        return None
    avg_volume = sum(volumes[-20:]) / 20
    current_volume = volumes[-1]
    return round(current_volume / avg_volume, 2) if avg_volume > 0 else None
```

### 均线多头排列

```python
def calc_ma_bullish(closes):
    """MA5 > MA10 > MA20 且三线均在上行（MA5 斜率 > 0）"""
    if len(closes) < 21:
        return False
    ma5 = sum(closes[-5:]) / 5
    ma10 = sum(closes[-10:]) / 10
    ma20 = sum(closes[-20:]) / 20
    slope_ma5 = closes[-1] - closes[-2]  # 简化斜率
    return ma5 > ma10 > ma20 and slope_ma5 > 0
```

### 4.6 相对强弱扩散度（成分股广度）

衡量板块内成分股跑赢基准的比例，避免单一龙头拉动被误判为板块级轮动。

```python
def calc_sector_breadth(etf_closes, hs300_closes, lookback=20):
    """
    板块相对强弱扩散度：板块近期涨幅中，跑赢沪深300的交易日占比。
    etf_closes / hs300_closes: 收盘价列表（最旧→最新）
    lookback: 回顾窗口（默认20日）
    返回：(扩散度百分比, 是否有效)
    """
    if len(etf_closes) < lookback + 1 or len(hs300_closes) < lookback + 1:
        return None, False

    wins = 0
    total = 0
    for i in range(-lookback, 0):
        etf_ret = etf_closes[i] / etf_closes[i - 1] - 1
        hs_ret = hs300_closes[i] / hs300_closes[i - 1] - 1
        if etf_ret > hs_ret:
            wins += 1
        total += 1

    return round(wins / total * 100, 2), True
```

**评分应用（输出字段，非综合得分维度）**：
- 扩散度 > 70%：成分股广泛跑赢 → 板块级轮动信号可靠性高，输出标注「✅扩散度高」
- 扩散度 40–70%：部分成分股领先 → 谨慎解读，输出标注「⚠️扩散度中等」
- 扩散度 < 40%：仅少数龙头强势 → 警惕假轮动，输出标注「🔺扩散度低」

> 注：ETF本身无法获得成分股明细，用ETF价格与沪深300的日度胜负比作为代理指标，与赔率/分位共同构成量能验证的辅助维度。

### 4.7 轮动强度排名变化（3/5/10/20 日）

记录不同周期相对强弱排名的变化速度与稳定性，区分一日脉冲与持续爬升。

```python
def calc_rank_velocity(rank_5d, rank_10d, rank_20d):
    """
    rank_5d / rank_10d / rank_20d: 分别为近5日/10日/20日相对强度排名（正整数，1=最强）
    返回 dict:
        - speed_score: 排名加速得分（0-10），综合3个周期的变化
        - stability: 排名稳定性（标准差，越小越稳定）
        - trend: 'accelerating' / 'stable' / 'decelerating'
    """
    import statistics
    ranks = [rank_5d, rank_10d, rank_20d]
    if any(r is None or r <= 0 for r in ranks):
        return None

    stdev = round(statistics.stdev(ranks), 2) if len(ranks) > 1 else 0
    # 加速：5日排名显著优于20日
    improvement = rank_20d - rank_5d  # 正数=排名上升
    speed_score = round(min(max(improvement * 1.5, 0), 10), 2)

    if improvement >= 3:
        trend = 'accelerating'
    elif improvement <= -3:
        trend = 'decelerating'
    else:
        trend = 'stable'

    return {
        'speed_score': speed_score,
        'stability': stdev,
        'trend': trend,
        'improvement': improvement
    }
```

**输出标注**：
- `↑加速爬升`：5日排名比20日排名提升≥3位，且speed_score≥5
- `→排名稳定`：各周期排名波动小，trend='stable'
- `↓排名下滑`：5日排名比20日排名下降≥3位

### 4.8 拥挤度/过热状态检测

基于短期涨幅分位、成交额分位、乖离率、连续阳线数等形成状态提醒，避免将高位加速误判为健康轮动。

```python
def calc_crowding_score(closes, volumes, period=10):
    """
    综合四个维度计算拥挤度/过热得分（0-100，越高越拥挤）
    仅用于状态标注，不参与综合得分计算。
    """
    if len(closes) < period + 1 or len(volumes) < period + 1:
        return None

    # ① 短期涨幅分位（近10日涨幅在近90日中的分位）
    short_return = (closes[-1] / closes[-period] - 1) * 100
    # ② 成交额分位（近10日均量在近90日中的分位）
    avg_vol_10 = sum(volumes[-10:]) / 10
    vol_percentile = sum(1 for v in volumes[-90:] if v < avg_vol_10) / min(len(volumes), 90) * 100
    # ③ 乖离率（现价偏离20日均线百分比）
    ma20 = sum(closes[-20:]) / 20
    deviation = abs((closes[-1] - ma20) / ma20 * 100)
    # ④ 连续阳线数
    consecutive_green = 0
    for i in range(len(closes) - 1, -1, -1):
        if closes[i] > closes[i - 1]:
            consecutive_green += 1
        else:
            break

    score = 0
    # 短期涨幅分位 > 85% → +30分
    if vol_percentile > 85:
        score += 30
    elif vol_percentile > 70:
        score += 15
    # 成交额分位 > 85% → +25分
    if vol_percentile > 85:
        score += 25
    elif vol_percentile > 70:
        score += 12
    # 乖离率 > 15% → +25分，> 10% → +15分
    if deviation > 15:
        score += 25
    elif deviation > 10:
        score += 15
    # 连续阳线 ≥ 7 → +20分，≥ 5 → +10分
    if consecutive_green >= 7:
        score += 20
    elif consecutive_green >= 5:
        score += 10

    return min(score, 100)
```

**过热状态输出**：
- 拥挤度得分 ≥ 70：标注「🔥拥挤」；综合得分正常输出，但注明「注意追高风险」
- 拥挤度得分 50–69：标注「⚡偏热」
- 拥挤度得分 < 50：正常，无标注

> ⚠️ **拥挤度标注不构成操作建议**：过热状态仅表示该板块短期交易情绪处于历史高位，提示用户关注波动风险，不代表看空或建议卖出。

### 4.9 持续性验证窗口

首次触发启动观察/趋势增强后，设置 1-3 个交易日确认窗口，观察相对强弱和量能状态是否维持。持续性验证仅改变信号置信度，不改变信号类型输出。

```python
def calc_confirmation_window(signal_label, confirm_days=3,
                               rel_strength_series=None,
                               volume_ratio_series=None):
    """
    signal_label: 当前信号标签（'启动观察' / '趋势增强' 等）
    confirm_days: 确认窗口天数（默认3）
    rel_strength_series: 近N日相对强弱列表（用于趋势验证）
    volume_ratio_series: 近N日成交量放大倍数列表

    返回: {
        'confirmed': bool,    # 窗口内是否持续符合条件
        'days Held': int,     # 维持天数
        'confidence': str     # '高' / '中' / '低'
    }
    """
    if signal_label not in ('启动观察', '趋势增强'):
        return {'confirmed': None, 'days_held': 0, 'confidence': '不适用'}

    if rel_strength_series is None or volume_ratio_series is None:
        return {'confirmed': None, 'days_held': 0, 'confidence': '数据不足'}

    held_days = 0
    for i in range(min(confirm_days, len(rel_strength_series))):
        rel_ok = rel_strength_series[i] is not None and rel_strength_series[i] > 0
        vol_ok = volume_ratio_series[i] is not None and volume_ratio_series[i] > 1.0
        if rel_ok and vol_ok:
            held_days += 1

    confirmed = held_days >= confirm_days
    if held_days >= confirm_days:
        confidence = '高'
    elif held_days >= confirm_days - 1:
        confidence = '中'
    else:
        confidence = '低'

    return {
        'confirmed': confirmed,
        'days_held': held_days,
        'confidence': confidence
    }
```

**输出格式**：在信号标签后增加确认状态后缀，例如：
- 「🔴启动观察（确认中，高）」：窗口内条件持续满足，置信度高
- 「🟡趋势增强（确认中，低）」：窗口内条件断续，置信度低，状态描述增加「注意信号稳定性」

> 注：确认窗口不改变信号优先级判定逻辑，仅作为信号置信度标注。窗口内任何一日出现反向信号（如死叉/机会流失），直接输出该反向信号，忽略确认状态。

---

## 五、轮动评分框架（核心）

> 从单 ETF 绝对信号升级为「相对强弱 × 趋势阶段 × 量能验证 × 基准对比」四维评估（**注：本技能无资金流向数据源，量能验证统一采用成交量/赔率/历史分位综合评估**）。

### 5.1 相对强弱计算

以沪深300（000300）为基准，计算各 ETF 的超额收益。

```python
def calc_relative_strength(etf_closes, hs300_closes, periods=[5, 10, 20]):
    results = {}
    for p in periods:
        if len(etf_closes) >= p and len(hs300_closes) >= p:
            etf_return = (etf_closes[-1] / etf_closes[-p] - 1) * 100
            hs300_return = (hs300_closes[-1] / hs300_closes[-p] - 1) * 100
            results[f'{p}日相对强度'] = round(etf_return - hs300_return, 2)
        else:
            results[f'{p}日相对强度'] = None
    return results
```

### 5.2 轮动评分公式

每只 ETF 综合得分为以下四项之和，基准分为 0 分（沪深300）。

```
综合得分 = 相对强弱分(±40) + 趋势强度分(±30) + 量能验证分(±20) + 基准偏离分(±10)

> **注**：拥挤度/过热得分（§4.8）不参与综合得分计算，单独输出为状态标注，用于提示高位风险。
```

| 维度 | 指标 | 满分 | 评分规则 |
|------|------|------|---------|
| **相对强弱分** | 5日/10日/20日超额收益 vs 沪深300 | ±40 | 5日超额每+1%得+4分，上限20；10日超额每+1%得+2分，上限12；20日超额每+1%得+1分，上限8；负值按同等规则扣分，下限-40 |
| **趋势强度分** | RSI(14)所处区间 + 趋势方向 | ±30 | RSI>70得+8，RSI 60-70得+5，RSI 50-60得+2，RSI<50得0；均线多头排列+5；MACD金叉+5；MACD柱持续扩大（近3日）+4；RSI从低位上穿50+3 |
| **量能验证分** | 成交量放大 + 量能指标 | ±20 | 成交量放大>2倍得+8，1.5-2倍得+5，1.2-1.5倍得+2；赔率>1.5得+4，1-1.5得+2，<1得0；历史分位>70%得+3（强势区），<30%得+2（低位区） |
| **基准偏离分** | 20日绝对涨幅 vs 沪深300绝对涨幅 | ±10 | 20日绝对涨幅超过沪深300 5%以上得+5；超过10%得+10；低于沪深300按同等规则扣分 |

### 5.3 趋势阶段标签

| 阶段 | 定义 | 动作含义 |
|------|------|---------|
| **启动** | MACD金叉 + RSI>50 + 成交量放大>1.2倍 | MACD金叉+RSI>50+成交量放大，等待价格确认 |
| **确认** | 启动后出现20日新高，或RSI上穿60 | 趋势确认，观察持续性 |
| **加速** | RSI从55上穿70 + 赔率>1.5 + 均线多头 | 高位风险标记，趋势状态延续 |
| **整固** | RSI>70持续3天以上，或高位缩量 | 高位风险标记，RSI>70持续，回撤风险上升 |
| **衰竭** | RSI从70+快速回落至50以下 + 成交量萎缩 | 趋势失效风险上升，需标记为风险状态 |
| **机会流失** | 均线破位 + MACD死叉 + RSI<40（同日满足） | 趋势失效提示，应标记为风险状态 |

> ⚠️ **RSI > 70 不是高位反转依据/操作依据**：RSI 超买只是高位标记，趋势 ETF 可长期维持 RSI 70+。真正的高位反转风险信号是衰竭或机会流失三条件同时满足。

### 5.4 组合优先级规则

信号判定按以下优先级组合判断（从上到下优先级递减）：

1. **数据有效性** > 所有信号（数据不合格直接跳过）
2. **MACD 金叉** > 20日新高（MACD 金叉是最早强势/上行动能信号）
3. **相对强弱趋势** > 绝对强弱（轮动看相对强度，不是单纯涨跌）
4. **成交量确认** > 价格突破（无量突破需谨慎）
5. **衰竭/机会流失** 信号优先于所有强势/上行动能信号（风控优先）

### 5.5 排名计算

```
板块排名 = 按综合得分降序排列
5日强度排名 = 按5日相对强度降序排列
排名变化 = 近5日排名 vs 近20日排名（差值，正数表示轮动加速）
```

---

## 六、信号判定规则表（伪代码）

> 详见 `references/signal-rules-pseudocode.md`

### 核心判定伪代码

```python
def determine_signal(etf_data, hs300_data, delay_flag=False):
    """
    etf_data = {
        'closes': [...], 'volumes': [...], 'rsi': float,
        'macd_hist': float, 'prev_macd_hist': float,
        'volume_ratio': float, 'percentile': float,
        'reward_ratio': float, 'ma_bullish': bool,
        'new_high_20d': bool,
        'relative_strength': {5日: float, 10日: float, 20日: float},
        'sector_breadth': float,       # 相对强弱扩散度（%）
        'rank_velocity': dict,         # 排名变化 dict（speed_score/stability/trend）
        'crowding_score': float,       # 拥挤度/过热得分（0-100）
        'confirm_status': dict         # 确认窗口 dict（confirmed/days_held/confidence）
    }
    delay_flag: True 表示数据延迟 ≥ 2 天
    """

    # === 阶段一：前置过滤 ===
    if etf_data['rsi'] is None or etf_data['macd_hist'] is None:
        return "❌数据不足", None

    # === 阶段二：机会流失判断（风控优先） ===
    macd_dead = macd_deathcross(etf_data['macd_hist'], etf_data['prev_macd_hist'])
    rsi_below_40 = etf_data['rsi'] < 40
    ma_bearish = not etf_data['ma_bullish']

    if macd_dead and rsi_below_40 and ma_bearish:
        return "🟤机会流失", "趋势失效提示，应标记为风险状态"

    # === 阶段三：过热回落 ===
    # RSI 从 70+ 快速回落至 50 以下，且成交量萎缩
    prev_rsi_above_70 = (etf_data['rsi'] < 50 and
                         etf_data['rsi'] > 40)  # 简化：当前 RSI 在 40-50 区间（从高位回落）
    volume_shrinking = etf_data['volume_ratio'] < 1.0
    if prev_rsi_above_70 and volume_shrinking and etf_data['macd_hist'] < 0:
        return "🟤过热回落风险", "注意趋势反转"

    # === 阶段四：高位整固 ===
    # RSI > 70 持续 3 天以上（简化：用历史分位 > 85% 代理）
    if etf_data['percentile'] is not None and etf_data['percentile'] > 85:
        if etf_data['volume_ratio'] < 1.2:
            return "⚪高位整固", "高位风险标记，RSI>70持续，回撤风险上升"

    # === 阶段五：强信号拦截（数据延迟时禁止触发） ===
    # 延迟 ≥ 2 天时，禁止触发 🟢动量加速 / 🟡趋势增强 / 🔴启动观察
    if delay_flag:
        # 仅保留历史状态参考，不生成新标签
        return None, "⚠️数据延迟，最新信号仅供参考"

    # === 阶段六：动量加速 ===
    # RSI 在 65-75 区间 + 赔率 > 1.5 + 均线多头 + 拥挤度 < 70（避免高位过热区间加速）
    rsi_accelerating = 65 <= etf_data['rsi'] <= 75
    reward_high = etf_data['reward_ratio'] is not None and etf_data['reward_ratio'] > 1.5
    crowding_ok = etf_data.get('crowding_score', 100) < 70  # 过热区间禁止触发加速
    if rsi_accelerating and reward_high and etf_data['ma_bullish'] and crowding_ok:
        return "🟢动量加速", "高位风险标记，趋势状态延续"
    # 过热区间（crowding_score >= 70）出现加速特征 → 输出⚠️过热风险，不输出动量加速
    if rsi_accelerating and reward_high and etf_data['ma_bullish'] and not crowding_ok:
        return "⚠️过热风险", "高位过热状态，注意波动风险"

    # === 阶段七：趋势增强 / 启动观察 ===
    # 启动观察 + 20日新高
    macd_golden = macd_crossover(etf_data['macd_hist'], etf_data['prev_macd_hist'])
    rsi_above_50 = etf_data['rsi'] > 50
    volume_expanding = etf_data['volume_ratio'] is not None and etf_data['volume_ratio'] > 1.2

    if macd_golden and rsi_above_50 and volume_expanding:
        if etf_data['new_high_20d']:
            return "🟡趋势增强", "趋势确立，关注延续性"
        else:
            return "🔴启动观察", "MACD金叉+RSI>50，等待价格确认信号"

    # === 阶段八：相对走弱（持仓专用） ===
    # 持仓板块 RSI < 40 + MACD 死叉 + 均线空头
    if rsi_below_40 and macd_dead and ma_bearish:
        return "🔵相对走弱", "原有趋势告破，轮动弱化状态"

    # === 阶段八：其他观察信号 ===
    if etf_data['relative_strength']['5日'] is not None:
        if etf_data['relative_strength']['5日'] > 2:
            return "🟡相对强势", "短线强势，但不构成操作依据"

    # === 阶段九：扩散度低预警（辅助参考） ===
    # 扩散度 < 40% 且出现强势信号 → 输出提示，不改变信号
    breadh = etf_data.get('sector_breadth')
    if breadh is not None and breadh < 40:
        # 仅作为状态标注追加，不阻断任何信号
        pass  # 在输出端标注🔺扩散度低，由主流程处理

    # === 阶段十：排名加速验证（辅助参考） ===
    # 加速爬升信号配合启动/趋势信号 → 提升置信度
    rv = etf_data.get('rank_velocity')
    if rv is not None and rv.get('trend') == 'accelerating':
        # 仅作为置信度提升标注，不独立输出信号
        pass

    return None, None  # 无明确信号
```

---

## 七、样例输出

```
### 🚀 板块动量扫描

**扫描时间**：2026-05-22 14:35 | 覆盖：11/11 只ETF | 数据状态：✅ 实时
**基准**：沪深300（000300）同期涨跌幅：5日 -0.32% / 10日 +0.87% / 20日 +2.15%

### 综合评分排名

| 排名 | 板块 | 综合得分 | 5日相对 | 10日相对 | 20日相对 | RSI | 20日新高 | 拥挤度 | 信号 |
|------|------|---------|---------|---------|---------|-----|---------|------|------|
| 1 | 半导体 | +52 | +6.3% | +8.7% | +12.1% | 70.6 | ▲ | 🔥78 | 🟢动量加速 |
| 2 | 机器人 | +41 | +4.1% | +6.2% | +9.8% | 71.2 | ▲ | 🔥65 | 🟢动量加速 |
| 3 | 人工智能 | +38 | +3.8% | +5.1% | +7.6% | 68.4 | — | ⚡62 | ⚪高位整固 |
| 4 | 通信设备 | +29 | +2.1% | +3.4% | +5.2% | 62.3 | ▲ | ⚡52 | 🟡趋势增强 |
| 5 | 卫星通信 | +18 | +1.5% | +2.8% | +4.1% | 58.7 | — | — | ⚠️数据延迟 |
| 6 | 有色金属 | +7 | -0.3% | +0.5% | +2.1% | 52.3 | — | 🔺38 | 🔵相对走弱 |
| 7 | 红利低波 | +2 | -0.5% | +0.2% | +1.8% | 48.7 | — | 🔺22 | ⚪中性 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

### 动量排名

**今日综合 Top 3**（按综合得分）
1. 半导体（+52）：5日超额+6.3%，RSI 70.6，动量加速
2. 机器人（+41）：5日超额+4.1%，RSI 71.2，均线多头
3. 人工智能（+38）：5日超额+3.8%，RSI 68.4，高位整固

**近5日强度 Top 3**（按5日相对强度）
1. 半导体：+6.3% 超额
2. 机器人：+4.1% 超额
3. 人工智能：+3.8% 超额

**排名变化**（5日排名 vs 20日排名，含加速/稳定/下滑标注）
- 半导体：5日第1 vs 20日第3 → ↑加速爬升
- 通信设备：5日第4 vs 20日第7 → ↑加速爬升

### 持仓关联

> 持仓数据来源：记忆（018490/010990/004432/006131/016129）

| 持仓板块 | 跟踪ETF | RSI | 5日/10日/20日相对强度 | 综合排名 | 状态 |
|---------|--------|-----|---------------------|---------|------|
| 有色金属 | 560860 / 512400 | 52.3 | -0.3% / +0.5% / +2.1% | 第6 | 🔵相对走弱 |
| 沪深300 | 000300 | — | 基准 | 基准 | 基准 |
| 红利低波 | 515100 | 48.7 | -0.5% / +0.2% / +1.8% | 第7 | ⚪防御价值 |

- **有色金属**：10日相对强度转负（+0.5%），20日超额收敛至+2.1%，综合排名下滑至第6。MACD 柱收窄，注意是否形成死叉。
- **红利低波**：RSI 48.7，防御属性有效，5日相对-0.5%但20日仍+1.8%，呈现防御属性

### 重点关注

- **🔴 启动观察**：无（数据均已延迟或信号不足）
- **🟡 趋势增强**：通信设备（MACD 持续扩张，成交量放大 1.4 倍）
- **🟢 动量加速**：半导体（RSI 70.6，赔率1.5:1，均线多头延续）
- **⚠️ 数据延迟**：卫星通信（最新数据 2026-05-21，延迟1天）

---

*本报告仅供板块轮动节奏观察，不构成任何投资建议。任何人依据本报告进行实际投资操作或交易操作，后果自负。*
```

---

## 八、异常数据处理汇总

| 异常 | 判断标准 | 输出标记 | 信号影响 |
|------|---------|---------|---------|
| 接口失败 | HTTP 非 200 / 空响应 / 超时 10s | `❌数据获取失败` | 不生成任何信号 |
| 数据严重延迟 | 最新 K 线距今 ≥ 2 天 | `⚠️数据延迟` | 禁止🔴启动观察/🟡趋势增强/🟢动量加速 |
| K 线不足 RSI | < 20 根 | `❌数据不足` | 不计算 RSI |
| K 线不足 MACD | < 35 根 | `⚠️指标缺失` | 不计算 MACD |
| K 线不足 20 日新高 | < 35 根 | `⚠️指标缺失` | 不判断 20 日新高 |
| 双重覆盖 ETF 数据不一致 | 同板块两只 ETF 信号矛盾 | 取 RSI 较高者为代表，输出时并列注明 | 较高者参与排名 |
| 扩散度数据不足 | K 线不足 21 根 | `⚠️扩散度数据不足` | 不计算扩散度，仅保留原信号 |
| 排名数据不足 | 10日/20日排名数据不足 | `⚠️排名加速数据不足` | 排名变化输出为空 |

---

## 九、执行流程（不可打乱顺序）

> 以下顺序不可打乱，每步完成前不进入下一步。

1. **确认扫描场景**：日报扫描 / 用户主动询问 / 持仓板块检查 / 早盘预览。
2. **获取监控列表 ETF K 线**：对每只 ETF 拉取最近 90 日日线（scale=240，datalen=90）。
3. **数据有效性检查**：立即验证最新 K 线日期与今日差距；若数据缺失/失败/根数不足，按§八处理。
4. **计算基础指标**：RSI(14)、MACD(12,26,9)、历史分位、赔率、成交量放大倍数、均线状态。
5. **计算相对强弱**：以沪深300为基准，计算各 ETF 的5日/10日/20日超额收益。
6. **计算辅助指标**：计算相对强弱扩散度（§4.6）、轮动强度排名变化（§4.7）、拥挤度得分（§4.8）。
7. **计算综合得分**：按§五评分公式计算每只 ETF 的四维综合得分。
8. **判定信号阶段**：按§六伪代码输出信号，不触发延迟数据的 🔴启动观察 / 🟡趋势增强 / 🟢动量加速信号；拥挤度 < 70 时方触发动量加速。
9. **持续性验证**：对启动观察/趋势增强信号，计算确认窗口（§4.9），输出置信度标注。
10. **输出排名**：综合排名、5日强度排名、排名变化（含加速/稳定/下滑标注）。
11. **持仓关联**：读取记忆中的用户持仓数据，单独标记持仓板块的相对强弱、扩散度、拥挤度与排名变化。
12. **合规声明**：输出末尾声明仅供参考，不构成投资建议。

---

## 十、验收标准

技术经理交付时，新版 SKILL.md 必须满足以下全部条件：

### 10.1 必含内容清单

| # | 内容 | 验收方式 |
|---|------|---------|
| 1 | 合规声明（仅供参考、不构成投资建议） | 文字存在于 SKILL.md 和每次输出末尾 |
| 2 | 输入/输出边界定义 | §一明确列出 |
| 3 | 数据源失败与延迟处理规则 | §三有明确规则表 |
| 4 | 指标最小样本要求 | RSI≥20根，MACD≥35根，20日新高≥35根 |
| 5 | 交易日/非交易日处理说明 | §三明确区分 |
| 6 | 字段类型转换说明 | §三注明 float() 转换 |
| 7 | 重复板块去重规则 | §二明确双重覆盖处理逻辑 |
| 8 | 轮动评分框架（相对强弱+趋势+量能验证+基准四维） | §五完整描述 |
| 9 | 相对沪深300超额收益、近5/20日涨幅排名 | §五§七有具体计算公式和样例 |
| 10 | 成交量放大倍数计算 | §四有公式 |
| 11 | MACD/RSI 阶段标签与趋势阶段定义 | §五.3 表格完整（启动/确认/加速/整固/衰竭/机会流失） |
| 12 | RSI>70 不等于高位反转依据/操作依据的明确说明 | §五.3 表格注释明确 |
| 13 | 组合优先级规则 | §五.4 明确5条优先级 |
| 14 | 信号判定伪代码或规则表 | §六伪代码完整 |
| 15 | 样例输出（至少一组） | §七包含真实格式的完整样例 |
| 16 | 异常数据处理汇总表 | §八表格完整 |
| 17 | 半导体复盘映射说明（验证信号优先级） | 见 `references/semiconductor-rally-2026.md` |
| 18 | 相对强弱扩散度计算（§4.6） | 有完整公式、评分规则、输出标注 |
| 19 | 轮动强度排名变化（§4.7） | 有完整公式、输出标注（加速/稳定/下滑） |
| 20 | 拥挤度/过热检测（§4.8） | 有完整公式、过热状态输出规则、动作含义说明 |
| 21 | 持续性验证窗口（§4.9） | 有确认窗口逻辑、置信度分级、输出格式 |
| 22 | 拥挤度/过热不参与综合得分的说明 | §五.2 注明确认 |
| 23 | 延迟≥2天禁止触发动量加速/趋势增强/启动观察 | §三§六双重覆盖 |

### 10.2 交付检查命令

```bash
# 验证 Skill 文件存在且格式正确
ls -la ~/.hermes/skills/productivity/sector-momentum-scanner/SKILL.md
head -50 ~/.hermes/skills/productivity/sector-momentum-scanner/SKILL.md

# 验证复盘文件存在
ls -la ~/.hermes/skills/productivity/sector-momentum-scanner/references/semiconductor-rally-2026.md

# 验证信号规则伪代码文件存在
ls -la ~/.hermes/skills/productivity/sector-momentum-scanner/references/signal-rules-pseudocode.md
```

---

## 十一、参考文档

| 文件 | 内容 |
|------|------|
| `references/semiconductor-rally-2026.md` | 2026年4月半导体行情复盘：MACD金叉（4/8）比20日新高（4/14）提前6天，验证信号体系优先级 |
| `references/signal-rules-pseudocode.md` | 完整信号判定伪代码与规则表，含所有信号分支逻辑 |
