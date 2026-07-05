# RPS 四策略过拟合审计记录（2026-07-05）

## 2026-07-05 修正：收益/Calmar 保真口径

前一版把 `rps_lowvol_repair`、`rps_lowvol_select3`、`combo_defensive_ma10` 放入“四策略通过池”，这个处理只解决了最低过拟合 gate，没有满足“收益和 Calmar 不能降级”的真实目标。因此这三条不再作为成熟替代策略使用，只保留为低收益防守观察线。

最新高收益保真审计命令：

```powershell
py -m scripts.run_latest_overfit_audit --audit-dir output\rps_momentum_backtest_latest\overfit_audit_high_fidelity
```

最新输出：

```text
output/rps_momentum_backtest_latest/overfit_audit_high_fidelity/
```

当前高收益保真候选池：

| 策略 | Gate | 年化 | 回撤 | Calmar | Train 年化 | Test 年化 | 2026 验证年化 | Top20 利润占比 | 说明 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| `hot60_rank1_cash65` | PASS | `51.06%` | `-9.66%` | `5.29` | `61.42%` | `38.20%` | `27.58%` | `78.82%` | 低回撤基准，保留 |
| `rank_switch_s30_hotrank1` | PASS | `53.28%` | `-12.43%` | `4.29` | `63.24%` | `41.60%` | `28.55%` | `88.62%` | 高收益/低回撤候选，历史审计通过 |
| `rank_switch_s30_notrank1ifrank23` | PASS | `79.86%` | `-17.42%` | `4.58` | `98.51%` | `57.49%` | `40.82%` | `97.37%` | 更进攻，Top20 接近红线但未越线 |
| `hot60_delta_amt_empty_c12` | PASS | `64.19%` | `-13.55%` | `4.74` | `84.80%` | `39.63%` | `22.32%` | `87.87%` | hot60 主线加低相关 delta 空档补充，历史审计通过 |

邻域稳定性：

| 策略 | 邻域中位 / 当前 Calmar | 当前 / 邻域中位 | 判断 |
| --- | ---: | ---: | --- |
| `hot60_rank1_cash65` | `99.16%` | `1.01` | 稳定 |
| `rank_switch_s30_hotrank1` | `89.34%` | `1.12` | 稳定 |
| `rank_switch_s30_notrank1ifrank23` | `83.57%` | `1.20` | 可接受，需继续压力测试 |
| `hot60_delta_amt_empty_c12` | `99.21%` | `1.01` | 稳定 |

Top-profit 压力测试补充：

```text
rescue_lu19_open80rank1:
remove_top0: 99.21% / -20.42%, test 98.16%, Top20 86.36%
remove_top3: 93.53% / -20.42%, test 95.86%, Top20 79.34%
remove_top5: 86.28% / -20.42%, test 79.93%, Top20 77.40%

rescue_lu19_open675:
remove_top0: 98.11% / -20.82%, test 105.04%, Top20 83.68%
remove_top3: 89.25% / -20.82%, test 84.46%, Top20 76.86%
remove_top5: 83.87% / -20.82%, test 84.06%, Top20 80.04%
```

储备观察线：

| 策略 | Gate | 年化 | 回撤 | Calmar | 说明 |
| --- | --- | ---: | ---: | ---: | --- |
| `rescue_lu19_open675` | FAIL | `98.11%` | `-20.82%` | `4.71` | 历史收益、Calmar、三段、邻域、Top20 都过；只因 clean-forward 仍只有 3 个交易日，暂不升成熟 |
| `high_yield_current` / `rescue_lu19_open80rank1` | FAIL | `99.21%` | `-20.42%` | `4.86` | 当前收益最高候选；同样需要冻结后前推满 20 个交易日 |

另复测 `rank_switch_combo_complement` nominal 100% 组合线，最新数据下 `a24_c30_top5_n1` 年化 `103.55%`、回撤 `-26.58%`，但 Top20 利润占比 `119.65%`，仍然失败；不能为了接近 100% 年化把它放进成熟池。

当前结论：

```text
已经找到 4 条高收益/高 Calmar 且历史过拟合审计 PASS 的策略：
1. hot60_rank1_cash65
2. rank_switch_s30_hotrank1
3. rank_switch_s30_notrank1ifrank23
4. hot60_delta_amt_empty_c12

rescue_lu19_open675 / high_yield_current 收益更高，但冻结后真实前推只有 3 个交易日，因此只作为储备观察线，不硬标成熟 PASS。

也就是说，当前四条成熟策略已经全部 PASS，且不再使用低收益/低 Calmar 的替代线凑数。
```

## 旧结论（已被上方收益/Calmar 保真口径废弃）

按当前本机数据和当前代码重跑后，原四条线仍不是全部通过；但已经找到三条可以组成“四策略通过池”的替代/保留线。

最新复现命令：

```powershell
py -m scripts.audit_rps_overfitting --panel-cache output\rps_momentum_backtest_latest\panel_2020-01-01_2026-07-03.parquet --end-date 2026-07-03 --validation-end 2026-06-26 --output-dir output\rps_momentum_backtest_latest\overfit_audit
```

最新输出：

```text
output/rps_momentum_backtest_latest/overfit_audit/
```

自动识别最新本地日线并复跑完整审计：

```powershell
py -m scripts.run_latest_overfit_audit
```

若只想快速检查 gate，不跑邻域扫描：

```powershell
py -m scripts.run_latest_overfit_audit --quick
```

### 原四条线

| 策略 | 审计结论 | 主要原因 |
| --- | --- | --- |
| `risk_optimized` | 不通过 | Top20 利润占比 `150.27%`，收益依赖少数赢家抵消大量亏损 |
| `combo_top_ma20` | 不通过 | 2026 验证段年化 `-11.49%`，三段一致性失败 |
| `hot60_rank1_cash65` | 通过 | 三段为正，回撤低，邻域稳定，Top20 利润占比未越过红线 |
| `high_yield_current` | 不通过成熟门槛 | 指标和邻域通过，但冻结后前推只有 3 个交易日，不满足最少 20 日前推门槛 |

### 修复候选

| 候选 | 用途 | 审计结论 | 说明 |
| --- | --- | --- | --- |
| `rps_lowvol_repair` | 替换 `risk_optimized` | 通过 | 基础 RPS 低波动线，Full `4.53% / -4.47%`，Top20 `93.09%` |
| `rps_lowvol_select3` | 作为第四条通过线 | 通过 | `rps_lowvol_repair` 的相邻分散度版本，Full `4.60% / -5.08%`，Top20 `98.97%` |
| `combo_defensive_ma10` | 替换 `combo_top_ma20` | 通过 | `90% risk + 10% delta_first + MA10`，Full `16.49% / -18.56%`，三段均为正 |

当前可组成的“四策略过拟合检测通过池”是：

```text
rps_lowvol_repair
rps_lowvol_select3
combo_defensive_ma10
hot60_rank1_cash65
```

高收益线 `high_yield_current` 还不能放入通过池。它的样本内指标、邻域、执行、Top20 都过，但必须继续冻结前推；当前只有 `2026-07-01` 至 `2026-07-03` 三个交易日，前推总收益 `+8.55%`，样本长度不足。

### 最新核心结果

| 策略 | Gate | Full 年化 | Full 回撤 | Full Calmar | Train 年化 | Test 年化 | 2026 验证年化 | 7月前推天数/收益 | 交易数 | Top20 利润占比 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `risk_optimized` | FAIL | `12.78%` | `-26.96%` | `0.47` | `9.36%` | `13.21%` | `44.86%` | `3 / 0.00%` | `1173` | `150.27%` |
| `combo_top_ma20` | FAIL | `18.59%` | `-28.95%` | `0.64` | `12.73%` | `40.76%` | `-11.49%` | `3 / -0.56%` | N/A | N/A |
| `rps_lowvol_repair` | PASS | `4.53%` | `-4.47%` | `1.01` | `4.35%` | `4.02%` | `8.54%` | `3 / 0.00%` | `159` | `93.09%` |
| `rps_lowvol_select3` | PASS | `4.60%` | `-5.08%` | `0.90` | `4.51%` | `3.37%` | `10.95%` | `3 / 0.00%` | `138` | `98.97%` |
| `combo_defensive_ma10` | PASS | `16.49%` | `-18.56%` | `0.89` | `14.34%` | `23.11%` | `8.48%` | `3 / -0.28%` | N/A | N/A |
| `hot60_rank1_cash65` | PASS | `51.06%` | `-9.66%` | `5.29` | `61.42%` | `38.20%` | `27.58%` | `3 / +4.33%` | `515` | `78.82%` |
| `high_yield_current` | FAIL | `99.21%` | `-20.42%` | `4.86` | `127.87%` | `59.86%` | `68.42%` | `3 / +8.55%` | `480` | `86.36%` |

### 最新邻域稳定性

| 策略 | 当前点 | 当前 Calmar | 邻域中位 Calmar | 邻域中位 / 当前 | 当前 / 邻域中位 | 判断 |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| `risk_optimized` | `rps20_delta=12` | `0.47` | `0.34` | `72.57%` | `1.38` | 非孤岛，但头部依赖失败 |
| `combo_top_ma20` | `80/20 + MA20` | `0.64` | `0.75` | `117.57%` | `0.85` | 非孤岛，但验证段失败 |
| `rps_lowvol_repair` | `nl4187` | `1.01` | `0.88` | `86.45%` | `1.16` | 通过 |
| `rps_lowvol_select3` | `select_num=3` | `0.90` | `0.92` | `101.81%` | `0.98` | 通过 |
| `combo_defensive_ma10` | `90/10 + MA10` | `0.89` | `0.69` | `77.98%` | `1.28` | 通过 |
| `hot60_rank1_cash65` | `open65/cash65` | `5.29` | `5.24` | `99.16%` | `1.01` | 通过 |
| `high_yield_current` | `rps120=99.35 + rescue` | `4.86` | `4.64` | `95.55%` | `1.05` | 指标过，前推长度不足 |

## 第一轮原四条结论

按 `2026-06-26` panel 重跑后，四条原线不是全部通过。

| 策略 | 审计结论 | 主要原因 |
| --- | --- | --- |
| `risk_optimized` | 不通过 | Top20 利润占比 `150.27%`，收益依赖少数赢家抵消大量亏损 |
| `combo_top_ma20` | 不通过 | 2026 验证段年化 `-11.49%`，三段一致性失败 |
| `hot60_rank1_cash65` | 通过 | 三段为正，回撤低，邻域稳定，Top20 利润占比未越过红线 |
| `high_yield_current` | 不通过实盘成熟门槛 | 指标和邻域通过，但 2020-2026 样本已被研究过程污染，缺少冻结后干净前推 |

因此，第一轮可保留为“通过过拟合初筛”的只有：

```text
hot60_rank1_cash65
```

高收益线只能保留为冻结观察候选；`risk_optimized` 和 `combo_top_ma20` 不能继续按成熟策略处理。

## 审计口径

复现命令：

```powershell
py -m scripts.audit_rps_overfitting --panel-cache output\rps_momentum_backtest\panel_2020-01-01_2026-06-26.parquet
```

输出：

```text
output/rps_momentum_backtest/overfit_audit/
```

数据和切分：

```text
全样本：2020-01-01 至 2026-06-26
训练集：2020-01-01 至 2023-12-31
测试集：2024-01-01 至 2025-12-31
验证集：2026-01-01 至 2026-06-26
```

主要红线来自 `过拟合检测以及防止过拟合方法与案例.md`，其中 Top20 利润占比是对“收益不能主要来自极少数样本”的量化扩展：

```text
1. 当前参数不能是孤岛：邻域中位数 / 当前点 >= 70%，当前点 / 邻域中位数 <= 1.5。
2. 训练、测试、验证不能严重背离。
3. 执行层必须遵守 T+1、涨跌停、手续费、100 股整手等约束。
4. Top20 利润占比不能超过 100%。
5. 已污染样本不能继续包装成干净样本外证据。
```

## 核心结果

| 策略 | Full 年化 | Full 回撤 | Full Calmar | Train 年化 | Test 年化 | 2026 验证年化 | 交易数 | Top20 利润占比 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `risk_optimized` | `12.82%` | `-26.96%` | `0.48` | `9.36%` | `13.21%` | `44.86%` | `1173` | `150.27%` |
| `combo_top_ma20` | `18.64%` | `-28.95%` | `0.64` | `12.73%` | `40.76%` | `-11.49%` | N/A | N/A |
| `hot60_rank1_cash65` | `51.24%` | `-9.66%` | `5.31` | `61.42%` | `38.20%` | `27.58%` | `513` | `75.08%` |
| `high_yield_current` | `99.79%` | `-20.42%` | `4.89` | `127.87%` | `59.86%` | `68.42%` | `478` | `78.13%` |

## 邻域稳定性

| 策略 | 当前点 | 当前 Calmar | 邻域中位 Calmar | 邻域中位 / 当前 | 当前 / 邻域中位 | 判断 |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| `risk_optimized` | `rps20_delta=12` | `0.48` | `0.35` | `74.37%` | `1.34` | 非孤岛，但收益质量弱 |
| `combo_top_ma20` | `80/20 + MA20` | `0.64` | `0.76` | `117.56%` | `0.85` | 非孤岛，但验证段失效 |
| `hot60_rank1_cash65` | `open65/cash65` | `5.31` | `5.26` | `99.14%` | `1.01` | 通过 |
| `high_yield_current` | `rps120=99.35 + rescue` | `4.89` | `4.67` | `95.56%` | `1.05` | 指标非孤岛，但样本污染未解除 |

## 逐条处理决定

### 1. `risk_optimized`

当前代码和当前数据下，旧文档里的 `48.33% / -21.12%` 已不能复现。当前实盘撮合口径结果只有 `12.82% / -26.96%`，而且 Top20 利润占比达到 `150.27%`。

处理决定：

```text
不再作为成熟策略。
只保留为 legacy baseline / 对照线。
若要恢复，必须重新搜索低头部依赖版本，并通过 Top20 <= 100% 和三段一致性。
```

补充检查：现成 `stable` 和 `best` preset 不能替代：

```text
stable：46.68% / -68.79%，2026 年 -58.15%
best：14.16% / -67.00%，2026 年 -45.82%
```

### 2. `combo_top_ma20`

当前 `80% risk + 20% delta_first + MA20` 不是参数孤岛，但 2026 验证段为负，三段一致性失败。

邻域里更防守的 `90% risk + 10% delta_first + MA10` 表现更均衡：

```text
Full：16.54% / -18.56% / Calmar 0.89
Train：14.34%
Test：23.11%
2026 验证：8.48%
```

处理决定：

```text
当前 combo_top_ma20 不通过。
可把 `90/10 + MA10` 列为修复候选，但必须重新定义策略名、规则和目标，不应把它包装成原 combo_top 的自然延续。
```

### 3. `hot60_rank1_cash65`

这是本轮唯一通过的线。它的优势不是最高收益，而是：

```text
Full 回撤小于 10%
训练、测试、验证均为正
open/cash 邻域稳定
Top20 利润占比 75.08%，未越过 100% 红线
执行口径包含 T+1、涨跌停、手续费、100 股整手
```

处理决定：

```text
保留为当前唯一成熟基准。
后续只允许做冻结前推和成交审计，不再围绕历史样本微调参数。
```

### 4. `high_yield_current`

高收益线的指标本身通过机械检查，邻域也不是孤岛；但它不能通过成熟策略门槛，因为已有外部评审记录确认：

```text
2020-2026 被反复用于规则筛选。
Test/2026 已参与研究过程。
真正样本外只能从 2026-07 冻结后逐月前推。
```

处理决定：

```text
保留为冻结观察候选。
不允许继续在 2020-2026 上调参追求通过。
只有完成保守成交审计、rps120 rescue 删除/单调重构、2026-07 起前推未坍塌后，才能重新申请成熟状态。
```

## 当前下一步

1. 将 `risk_optimized` 降级为旧对照线，用 `rps_lowvol_repair` 作为低波动替代候选。
2. 将 `combo_top_ma20` 降级为旧对照线，用 `combo_defensive_ma10` 作为组合替代候选。
3. 保留 `hot60_rank1_cash65` 为当前主成熟基准。
4. `high_yield_current` 冻结规则，不再调参；继续跑 2026-07 起前推，累计至少 20 个交易日后再重新判定 clean-forward gate。
5. 当前“四策略通过池”已经可以先用 `rps_lowvol_repair`、`rps_lowvol_select3`、`combo_defensive_ma10`、`hot60_rank1_cash65`。等高收益线 clean-forward 通过后，再决定是否用 `high_yield_current` 替换其中一条低收益线。
