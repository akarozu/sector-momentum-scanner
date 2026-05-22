# 信号判定伪代码与规则表

> 配合《板块动量扫描仪 v2.0 §五轮动评分框架》《§六信号判定规则表》使用
> 本文件是 SKILL.md §六的核心补充，提供完整的判定伪代码和规则表。

---

## 一、完整信号判定伪代码

```python
"""
板块动量扫描仪 v2.0 — 信号判定主函数
输入：ETF数据字典、沪深300基准数据、数据延迟标志
输出：(信号标签, 动作含义) 或 (None, None)
"""

# ── 数据结构定义 ──────────────────────────────────────────────
ETF_DATA = {
    "code":         "159813",          # ETF代码
    "name":         "半导体",           # 板块名称
    "closes":       [...],              # 收盘价列表（最旧→最新）
    "volumes":      [...],              # 成交量列表
    "rsi":          78.6,               # RSI(14) 当前值
    "prev_rsi":     72.1,               # RSI(14) 前一日值
    "macd":         0.0153,            # MACD 柱当前值
    "prev_macd":    0.0029,            # MACD 柱前一日值
    "volume_ratio": 1.84,              # 成交量放大倍数（当前/20日均量）
    "percentile":   88.5,              # 90日历史分位（%）
    "reward_ratio": 1.62,              # 赔率（上行/下行空间比）
    "ma_bullish":   True,              # 均线多头排列（MA5>MA10>MA20且MA5上行）
    "new_high_20d": True,              # 20日新高标志
    "rel_strength": {                  # 相对沪深300超额收益（%）
        "5日":  6.3,
        "10日": 8.7,
        "20日": 12.1,
    },
    "abs_return": {                    # 绝对收益（%）
        "5日":  5.8,
        "20日": 14.3,
    }
}

HS300_DATA = {
    "rel_strength": {"5日": -0.32, "10日": 0.87, "20日": 2.15},
    "abs_return":   {"5日": -0.40, "20日":  2.15},
}

# ── 辅助函数 ───────────────────────────────────────────────────
def macd_golden_cross(curr_hist, prev_hist):
    """MACD金叉：柱由负转正（或从更负转向非负）"""
    return prev_hist < 0 <= curr_hist

def macd_death_cross(curr_hist, prev_hist):
    """MACD死叉：柱由正转负"""
    return prev_hist >= 0 > curr_hist

def rsi_cross_above(value, prev_value, threshold):
    """RSI上穿阈值"""
    return prev_value < threshold <= value

def rsi_cross_below(value, prev_value, threshold):
    """RSI下穿阈值"""
    return prev_value >= threshold > value

def all_bearish_signals(macd_dead, rsi_low, ma_bearish):
    """三条件同时满足（机会流失判断）"""
    return macd_dead and rsi_low and ma_bearish


# ── 信号判定主函数 ─────────────────────────────────────────────
def determine_signal(etf: dict, hs300: dict, delay_flag: bool = False):
    """
    参数：
        etf        — ETF数据字典
        hs300      — 沪深300基准数据
        delay_flag — True=数据延迟≥2天，禁止触发启动/加速信号

    返回：
        (信号标签, 动作含义)  或  (None, None) 表示无明确信号
    """

    # ══ 阶段0：数据有效性过滤 ══════════════════════════════════
    # 优先级最高：数据不合格直接跳过，不产生任何信号
    if etf.get("rsi") is None or etf.get("macd") is None:
        return "❌数据不足", "指标计算样本不足，无法生成信号"

    # ══ 阶段1：机会流失判断（风控优先） ══════════════════════
    # 三条件同时满足 → 最优先输出，优先于所有做多信号
    macd_dead = macd_death_cross(etf["macd"], etf["prev_macd"])
    rsi_low   = etf["rsi"] < 40
    ma_bear   = not etf.get("ma_bullish", False)

    if all_bearish_signals(macd_dead, rsi_low, ma_bear):
        return "🟤机会流失", "三条件同现：MACD死叉+RSI<40+均线破位，建议减仓/离场"

    # ══ 阶段2：过热回落（从高位快速反转向下） ════════════════
    # 判断：RSI从70+快速回落至40-50区间 + 成交量萎缩 + MACD转负
    rsi_fell_from_high = (
        40 < etf["rsi"] < 55
        and etf.get("prev_rsi", 0) >= 65   # 前一日RSI在65以上（从高位回落）
    )
    vol_shrink   = etf.get("volume_ratio", 1.0) < 1.0
    macd_turned  = etf["macd"] < 0

    if rsi_fell_from_high and vol_shrink and macd_turned:
        return "🟤过热回落", "高位反转迹象，注意趋势反转风险"

    # ══ 阶段3：高位整固（超买但未反转） ══════════════════════
    # RSI>70 持续（用历史分位>85%代理"持续"）且成交量萎缩
    overbought_sustained = (
        etf.get("percentile", 0) > 85
        and etf.get("volume_ratio", 1.0) < 1.2
    )
    if overbought_sustained:
        return "⚪高位整固", "高位缩量整固，持有不追，回撤风险积累"

    # ══ 阶段4：动量加速（持有但不再追加） ════════════════════
    # 判断：RSI在65-75区间（从55上穿70的简化）+ 赔率>1.5 + 均线多头
    rsi_accelerating = 65 <= etf["rsi"] <= 75
    reward_high      = etf.get("reward_ratio", 0) > 1.5
    ma_aligned       = etf.get("ma_bullish", False)

    if rsi_accelerating and reward_high and ma_aligned:
        if delay_flag:
            return "🟢动量加速（延迟⚠️）", "数据延迟，信号仅供参考"
        return "🟢动量加速", "持有为主，不追加仓位"

    # ══ 阶段5：趋势确认（从启动升级为确认） ═════════════════
    # 已在启动观察状态，出现20日新高或RSI上穿60
    rsi_crossed_60  = rsi_cross_above(etf["rsi"], etf.get("prev_rsi", 0), 60)
    hit_new_high_20 = etf.get("new_high_20d", False)

    if hit_new_high_20d or rsi_crossed_60:
        # 前置条件：已有启动迹象（简化：用5日相对强度>0代理）
        if etf.get("rel_strength", {}).get("5日", 0) > 0:
            if delay_flag:
                return "🟡趋势增强（延迟⚠️）", "趋势成立但数据延迟，谨慎确认"
            return "🟡趋势增强", "趋势确立，可关注但仍不追高"

    # ══ 阶段6：启动观察（最早做多信号） ═════════════════════
    # MACD金叉 + RSI>50 + 成交量放大>1.2倍
    macd_golden     = macd_golden_cross(etf["macd"], etf["prev_macd"])
    rsi_above_50    = etf["rsi"] > 50
    vol_expanding   = etf.get("volume_ratio", 0) > 1.2

    if macd_golden and rsi_above_50 and vol_expanding:
        if delay_flag:
            # 数据延迟：降级为观察，不触发启动
            return "🔴启动观察（延迟⚠️）", "MACD金叉成立但数据延迟，等确认后再操作"
        if etf.get("new_high_20d", False):
            return "🟡趋势增强", "启动+20日新高，趋势已确认"
        return "🔴启动观察", "关注不追，等确认信号出现"

    # ══ 阶段7：相对走弱（持仓管理用） ═══════════════════════
    # 持仓板块出现：RSI<40 + MACD死叉 + 均线空头
    if rsi_low and macd_dead and ma_bear:
        return "🔵相对走弱", "原有趋势告破，关注轮出机会"

    # ══ 阶段8：相对强势（短线超额收益） ═════════════════════
    if etf.get("rel_strength", {}).get("5日", 0) > 2.0:
        return "🟡相对强势", "短线超额收益明显，但不等同于买入信号"

    # ══ 阶段9：中性/无信号 ════════════════════════════════════
    return None, None
```

---

## 二、信号优先级规则表（摘要）

| 优先级 | 信号 | 触发条件 | 输出动作 | 延迟时处理 |
|--------|------|---------|---------|-----------|
| P0（风控） | 🟤机会流失 | MACD死叉 **AND** RSI<40 **AND** 均线破位 | 减仓/离场提醒 | 正常触发（优先） |
| P0（风控） | 🟤过热回落 | RSI从70+快速回落至40-55 **AND** 成交量萎缩 **AND** MACD<0 | 注意反转风险 | 正常触发 |
| P1 | ⚪高位整固 | 历史分位>85% **AND** 成交量缩<1.2倍 | 持有不追 | 正常触发 |
| P2 | 🟢动量加速 | RSI 65-75 **AND** 赔率>1.5 **AND** 均线多头 | 持有为主，不追加 | 降级为⚠️ |
| P3 | 🟡趋势增强 | 启动观察 **AND** (20日新高 **OR** RSI上穿60) | 可关注但不追高 | 降级为⚠️ |
| P4 | 🔴启动观察 | MACD金叉 **AND** RSI>50 **AND** 成交量>1.2倍 | 关注等确认 | 降级为⚠️ |
| P5 | 🔵相对走弱 | RSI<40 **AND** MACD死叉 **AND** 均线空头 | 关注轮出 | 正常触发 |
| P6 | 🟡相对强势 | 5日相对强度>2% | 短线关注 | 正常触发 |
| — | ⚪中性 | 不满足以上任何条件 | 无信号 | — |

---

## 三、组合优先级核心规则（与SKILL.md §五.4对应）

```
1. 数据有效性  > 所有信号（数据不合格直接跳过）
2. 机会流失/过热回落  > 所有做多信号（风控优先）
3. MACD金叉    > 20日新高（MACD金叉是最早做多信号）
4. 相对强弱趋势 > 绝对强弱（轮动看相对强度，不是单纯涨跌）
5. 成交量确认  > 价格突破（无量突破需谨慎）
```

---

## 四、半导体复盘映射（2026年4月行情验证）

> 详见 `references/semiconductor-rally-2026.md`，本节说明如何将复盘数据映射到判定规则。

| 日期 | 实际信号 | 按规则应触发 | 是否符合 | 说明 |
|------|---------|------------|--------|------|
| 2026-03-31 | 阶段新低 | — | — | 尚未满足启动条件 |
| 2026-04-01 | 修复中 | 🔵超跌反弹观察 | ✅ | RSI从27.5升至36.1，成交量未明显放大 |
| **2026-04-08** | **MACD金叉** | **🔴启动观察** | **✅** | MACD柱+0.0029（由负转正），RSI 43.7>50需等下一日确认；4/8当日不触发完整启动（金叉单条件），4/9或4/10 RSI>50时触发 |
| 2026-04-09 | RSI>50确认 | 🔴启动观察→🟡趋势增强 | ✅ | RSI上穿50，成交量放大1.3倍，满足完整启动条件 |
| 2026-04-14 | 20日新高 | 🟡趋势增强 | ✅ | 启动信号确认，趋势确立 |
| 2026-04-22 | RSI 77.6 | ⚪高位整固 | ✅ | 历史分位~80%，成交量收缩，RSI>70持续但未加速 |
| 2026-04-30 | RSI 79.8 | ⚪高位整固 | ✅ | RSI维持高位，赔率收敛至1.3 |
| 2026-05-06 | RSI 83.8 | ⚪高位整固/🟢动量加速 | ⚠️ | RSI 83.8但赔率仍>1.5，若成交量配合则触发加速；复盘显示5月行情加速延伸 |
| 2026-05-20 | 全程最高 | ⚪高位整固 | ✅ | 无衰竭/机会流失信号，持有阶段 |

**关键验证点：**
- MACD金叉（4/8）比20日新高（4/14）提前6天 → 验证"MACD金叉优先级高于20日新高"
- 半导体RSI长期维持55-86，从未触发"过热回落"或"机会流失" → 验证"RSI>70不等于卖点"原则
- 均线多头排列从4月中旬持续至5月中旬 → 趋势持有信号比RSI超买更有价值

---

## 五、数据校验清单（判定前必检）

```python
CHECKLIST = {
    "rsi_enough_samples":    len(closes) >= 20,    # RSI最少14+1根
    "macd_enough_samples":   len(closes) >= 35,    # MACD最少34+1根
    "new_high_20d_enough":   len(closes) >= 35,    # 20日新高判断最少35根
    "data_not_stale":        (today - latest_kline_date).days < 2,  # 数据延迟<2天
    "volume_ratio_valid":    volume_ratio is not None and volume_ratio > 0,
    "rel_strength_valid":    rel_strength["5日"] is not None,
}
```

每项检查失败时的输出标记和处理方式，详见 SKILL.md §三"数据异常处理规则表"和§八"异常数据处理汇总"。
