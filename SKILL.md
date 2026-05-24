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
        'relative_strength': {5日: float, 10日: float, 20日: float}
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
    # RSI 从 55 上穿 70（简化：RSI 在 65-75 区间且赔率 > 1.5）
    rsi_accelerating = 65 <= etf_data['rsi'] <= 75
    reward_high = etf_data['reward_ratio'] is not None and etf_data['reward_ratio'] > 1.5
    if rsi_accelerating and reward_high and etf_data['ma_bullish']:
        return "🟢动量加速", "高位风险标记，趋势状态延续"

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

    return None, None  # 无明确信号
```

---

## 七、样例输出

```
### 🚀 板块动量扫描

**扫描时间**：2026-05-22 14:35 | 覆盖：11/11 只ETF | 数据状态：✅ 实时
**基准**：沪深300（000300）同期涨跌幅：5日 -0.32% / 10日 +0.87% / 20日 +2.15%

### 综合评分排名

| 排名 | 板块 | 综合得分 | 5日相对 | 10日相对 | 20日相对 | RSI | 20日新高 | 信号 |
|------|------|---------|---------|---------|---------|-----|---------|------|
| 1 | 半导体 | +52 | +6.3% | +8.7% | +12.1% | 70.6 | ▲ | 🟢动量加速 |
| 2 | 机器人 | +41 | +4.1% | +6.2% | +9.8% | 71.2 | ▲ | 🟢动量加速 |
| 3 | 人工智能 | +38 | +3.8% | +5.1% | +7.6% | 68.4 | — | ⚪高位整固 |
| 4 | 通信设备 | +29 | +2.1% | +3.4% | +5.2% | 62.3 | ▲ | 🟡趋势增强 |
| 5 | 卫星通信 | +18 | +1.5% | +2.8% | +4.1% | 58.7 | — | ⚠️数据延迟 |
| 6 | 有色金属 | +7 | -0.3% | +0.5% | +2.1% | 52.3 | — | 🔵相对走弱 |
| 7 | 红利低波 | +2 | -0.5% | +0.2% | +1.8% | 48.7 | — | ⚪中性 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

### 动量排名

**今日综合 Top 3**（按综合得分）
1. 半导体（+52）：5日超额+6.3%，RSI 70.6，动量加速
2. 机器人（+41）：5日超额+4.1%，RSI 71.2，均线多头
3. 人工智能（+38）：5日超额+3.8%，RSI 68.4，高位整固

**近5日强度 Top 3**（按5日相对强度）
1. 半导体：+6.3% 超额
2. 机器人：+4.1% 超额
3. 人工智能：+3.8% 超额

**排名上升最快**（5日排名 vs 20日排名）
- 半导体：5日第1 vs 20日第3 → 轮动加速
- 通信设备：5日第4 vs 20日第7 → 轮动启动

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

---

## 九、执行流程（不可打乱顺序）

> 以下顺序不可打乱，每步完成前不进入下一步。

1. **确认扫描场景**：日报扫描 / 用户主动询问 / 持仓板块检查 / 早盘预览。
2. **获取监控列表 ETF K 线**：对每只 ETF 拉取最近 90 日日线（scale=240，datalen=90）。
3. **数据有效性检查**：立即验证最新 K 线日期与今日差距；若数据缺失/失败/根数不足，按§八处理。
4. **计算基础指标**：RSI(14)、MACD(12,26,9)、历史分位、赔率、成交量放大倍数、均线状态。
5. **计算相对强弱**：以沪深300为基准，计算各 ETF 的5日/10日/20日超额收益。
6. **计算综合得分**：按§五评分公式计算每只 ETF 的四维综合得分。
7. **判定信号阶段**：按§六伪代码输出信号，不触发延迟数据的 🔴启动观察 / 🟡趋势增强 / 🟢动量加速信号。
8. **输出排名**：综合排名、5日强度排名、排名变化。
9. **持仓关联**：读取记忆中的用户持仓数据，单独标记持仓板块的相对强弱与排名变化。
10. **合规声明**：输出末尾声明仅供参考，不构成投资建议。

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
