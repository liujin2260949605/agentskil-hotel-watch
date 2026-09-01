---
name: hotel-cancellation-revenue-watchdog
description: 对酒店预订数据进行周期性取消率预警、ADR 异常识别与资源匹配巡检。触发词：酒店巡检、取消率预警、hotel_bookings、ADR 异常、收益巡检、资源匹配巡检。固化脚本一键产出：字段校验、数据质量审计、清洗留痕、派生字段构造、指标计算（分子/分母/样本量标记）、交叉风险分析、风险表与人工复核清单、中文主报告，以及按类别分文件夹的 CSV/JSON/图表交付物。适用于接收 hotel_bookings.csv 或类似酒店预订数据的周期性经营巡检场景。
tools: [file_reader, python, chart_generator, report_writer]
---

# 酒店预订取消率预警与收益巡检智能体

把酒店预订数据转化为可重复执行的经营巡检工作流。核心业务任务是取消率预警；ADR 与资源匹配巡检用于辅助诊断，但不得替代业务判断。

**执行方式：优先调用本 skill 自带的固化脚本 `scripts/hotel_watchdog_runner.py`，不要现场重写分析代码。** 脚本已完整实现本契约；仅当出现脚本确实未覆盖的特殊需求时，才在脚本输出基础上做补充分析，并在 `summary.json` 与报告中注明补充逻辑。

## 快速执行

当用户提供数据文件路径（如 `hotel_bookings.csv` 或 zip 压缩包）并要求巡检时，按以下三步执行：

### 第 1 步：确认 Python 环境

按顺序检测可用解释器（需已安装 pandas、numpy、matplotlib）：

1. 工作区 venv：skill 目录向上三级下的 `.venv/`，即 `<workspace>/.venv/Scripts/python.exe`
2. 用户明确指定的其他 Python 解释器

若无满足依赖的解释器，先一次性准备环境（默认放在工作区 E 盘，避免占用 C 盘）：

```bash
python -m venv <workspace>/.venv
<workspace>/.venv/Scripts/python.exe -m pip install pandas numpy matplotlib
```

### 第 2 步：运行固化脚本

```bash
<venv-python> <skill-dir>/scripts/hotel_watchdog_runner.py \
  --data-file <用户数据文件路径> \
  --output-dir <输出目录> \
  --dimensions all
```

- 输出目录默认：数据文件所在目录下的 `output/`；用户指定时以指定为准
- 常用选项：`--start/--end YYYY-MM-DD`（到店日期窗口，end 含当天）、`--top-n`（风险表展示条数，默认 10）、`--set KEY=VALUE`（阈值覆盖，可多次）、`--deliverables on|off`（完整交付包开关，默认 on）、`--strict`（严格模式，任何质量告警即阻断）、`--config <路径>`（YAML 配置文件）、`--cross-groups <列表>`（指定交叉分组）
- 完整字段映射、必需字段清单、已知数据风险、默认阈值与配置说明见 `references/data_dictionary.md`

### 第 3 步：汇报结果

脚本运行结束后读取 `output/summary.json` 与 `output/hotel_watchdog_report.md`，向用户汇报：

- 顶部 KPI：总体取消率、City Hotel 取消率、Resort Hotel 取消率、ADR 中位数、房型变更率
- 风险信号与重点风险（附证据：分子/分母、样本量、维度取值）
- 必须人工复核的事项（`【待人工审核】` 标记）
- 输出目录导览（按类别分文件夹的 9 类交付物）

## 运行边界

可以做：

- 读取结构化的酒店预订文件，尤其是 `hotel_bookings.csv`。
- 在计算指标之前校验必需字段。
- 记录数据质量问题、清洗动作、阈值与降级分析。
- 计算取消率、ADR、资源匹配指标，并同时输出分子、分母与样本量标记。
- 生成管理层可读的报告（默认中文）与机器可读的 CSV/JSON 交付物。
- 将有风险的发现标记为「风险信号」「建议复核」或 `【待人工审核】`。

不可以做：

- 把本 skill 当作实时监控或在线告警系统。
- 执行处罚、下架、渠道调整、价格调整或资源重新分配等业务动作。
- 将 `reservation_status` 作为取消预警的输入字段。它与取消结果高度耦合，属于数据泄漏风险字段。
- 在没有可靠 booking_id 的情况下默认删除重复行。
- 将相关性或排名当作已被证实的因果结论。
- 报告默认使用中文输出；仅当用户明确要求其他语言时切换。

## 输入

默认接受一张预订事实表。若用户提供多个结构化文件，通过表头字段和行粒度识别出预订事实表。

- 默认文件名：`hotel_bookings.csv`；教学数据集预期规模 119,390 行 × 32 字段；无可靠 `booking_id`；到店日期跨度 2015-07-01 至 2017-08-31
- 可选配置：字段映射、分析日期窗口、阈值覆盖、输出目录、选定维度或交叉分组
- 若未提供配置，从表头推断映射关系，并将推断过程记录到 `summary.json` 与主报告
- 若提供 zip 压缩包，先解压并检查内部文件，不要假设路径
- 必需字段（缺失时阻断或降级）、可选字段、已知数据风险（重复行、company/agent/country/children 缺失率、ADR 异常、泄漏字段）的完整清单与处理规则见 `references/data_dictionary.md`

## 默认阈值

脚本内置默认阈值：`min_sample_size=30`、`min_sample_for_risk=100`、`cancellation_rate_alert=0.40`、`cancellation_rate_critical=0.60`、`non_refund_cancel_alert=0.90`、`room_change_rate_alert=0.15`、`adr_extreme_percentile=99`。每一次阈值覆盖都必须记录来源、原值、新值与生效值（写入 `summary.json`）。

## 工作流程（脚本已固化，运行时按此核对输出）

1. **定位并读取数据**：加载配置，读取预订表，记录文件路径、编码假设、行数、列数与字段映射；分析前确认必需字段齐全。
2. **数据质量校验**：每项检查一行写入 `quality/data_quality.csv`（必需字段、is_canceled 取值、日期可构造性、reservation_status 一致性、重复行统计、缺失率、ADR 分布、total_guests=0 / total_stays=0 / 房型变更计数）。
3. **带审计留痕的清洗**：所有动作写入 `quality/cleaning_log.csv`（列：表、字段、动作、行数、原因、填充值、取值范围）。重复行默认 `report_only`；children 缺失填 0；country 缺失填 `Unknown`；构造 `has_agent` / `has_company`；ADR 转数值并标记异常。清洗后的全量表与日期窗口子表分开保存。
4. **构造派生字段**：`arrival_date`、`total_guests`、`total_stays`、`is_room_changed`、`has_agent`、`has_company`、`adr_anomaly_flag`、`lead_time_bucket`、`adr_bucket`；四个标记字段输出数量与占比。
5. **应用日期窗口**：默认以 `arrival_date` 为窗口字段，`end` 含当天；记录原始范围、选定窗口、保留/排除行数；小样本窗口仅输出描述性表格。
6. **计算核心指标**：所有比率输出分子、分母、百分比与样本量标记；分母为 0 输出 `n/a`。取消率按总体 / hotel / market_segment / distribution_channel / deposit_type / customer_type / arrival_date_month / country；ADR 按 hotel / market_segment / distribution_channel / customer_type（均值、中位数、P95、P99、`adr<=0` 数、`adr>P99` 数）；资源按 hotel / arrival_date_month / market_segment / customer_type（房型变更率、停车位需求率、特殊请求率）。
7. **交叉分析**：默认至少完成 2 组（hotel×market_segment×is_canceled、deposit_type×customer_type×is_canceled、lead_time_bucket×is_canceled、adr_bucket×is_canceled、total_of_special_requests×is_canceled、distribution_channel×deposit_type×is_canceled 中选取），回答：最高风险组合、样本量是否充分、是否达阈值、是否需人工复核、业务含义与局限。
8. **构建风险表**：每维度输出 `维度、维度取值、总单量、取消量、取消率、ADR中位数、ADR_P95、房型变更率、停车位需求率、特殊请求率、风险标记、样本量标记`；判定规则：低于 min_sample_size 仅展示；达 min_sample_size 且取消率≥alert 标记风险信号；≥critical 标记重点风险；Non Refund 且≥non_refund_cancel_alert 标记重点风险并加 `【待人工审核】`。排序依据：风险优先级、取消率、ADR 中位数、订单量。
9. **深度诊断**：数据量充足或用户要求完整交付时执行——趋势对比、取消集中维度、风险组合（高取消+高 ADR、高取消+长提前期）、ADR 异常关联分布、资源压力集中情况。措辞用「与……相关联」，不得声称因果。
10. **生成报告与交付物**：主报告管理层可读、证据可溯源、默认中文，先事实后解读最后复核建议；推荐章节含顶部 KPI、数据质量摘要、取消率诊断、交叉风险、ADR 巡检、资源匹配、深度诊断、人工复核清单、局限性、生成信息。图表写入 `charts/`：KPI 卡片、分组条形图、交叉热力图、ADR 箱线图、资源条形图。

阻断或降级规则（脚本自动执行并在报告中说明）：缺 `hotel` 或 `is_canceled` 阻断核心结论；`is_canceled` 取值非法则停止计算；无取消记录阻断取消率分析；日期无法构造阻断窗口与趋势；`adr<=0` 占比超 5% 阻断收益判断但保留取消率分析；字段口径变化阻断历史对比并标注「口径变化待确认」。

## 输出契约

```text
output/
├── hotel_watchdog_report.md          # 主报告（中文）
├── summary.json
├── clean-data/
│   ├── bookings_clean.csv
│   └── bookings_derived.csv
├── quality/
│   ├── data_quality.csv
│   └── cleaning_log.csv
├── risk-tables/
│   ├── cancellation_by_hotel.csv
│   ├── cancellation_by_market_segment.csv
│   ├── cancellation_by_channel.csv
│   ├── cancellation_by_deposit.csv
│   ├── cancellation_by_customer_type.csv
│   ├── dimension_risk_table.csv
│   └── cross_analysis_results.csv
├── detail-tables/
│   ├── adr_anomaly_list.csv
│   ├── room_resource_check.csv
│   └── risk_combinations.csv
├── deep-diagnosis/
│   ├── trend_comparison.csv
│   ├── cancellation_concentration.csv
│   └── risk_amplifiers.csv
├── charts/
│   ├── kpi_overview.png
│   ├── cancellation_by_dimensions.png
│   ├── cross_risk_heatmap.png
│   ├── adr_distribution_boxplot.png
│   └── resource_matching_bars.png
└── deliverables/
    ├── business_scenario.md          # 业务场景与价值说明
    ├── workflow_diagram.md           # 流程图/流程说明
    ├── tool_list.md                  # 工具清单
    └── management_summary.md         # 一页式管理层摘要
```

`summary.json` 必须包含：输入文件路径、编码假设、行数、列数；字段映射与事实表角色；原始日期范围与实际窗口；生效阈值与覆盖记录；核心 KPI 及分子分母；清洗动作摘要；缺失字段与降级分析；`reservation_status` 泄漏字段排除确认；阻断问题、人工复核事项、生成文件清单。

## 人工复核规则

以下情况必须触发 `【待人工审核】`：

- Non Refund 押金类型取消率异常接近 100%（可能是业务定义或状态口径问题，不得直接下「押金策略失败」结论）
- ADR 极端值可能实质性影响收益判断
- 小样本组出现高比率异常
- 房型变更涉及实际运营策略调整
- 疑似字段口径变化或数据泄漏
- 无 booking_id 且重复行占比超过 20%

人工复核建议必须附带证据表、样本量、时间范围与数据质量上下文。

## 验收清单

交付前逐项确认：

- 主报告（中文）与 `summary.json` 已生成
- 数据质量表与清洗日志已生成
- 重复行未被默认删除
- `reservation_status` 未被用作预警输入
- 派生字段构造正确；`total_guests=0`、`total_stays=0`、ADR 异常均已标记
- 取消率、ADR、资源指标在相关处均展示分子/分母
- 至少完成 2 组交叉分析
- 小样本组未被误标为异常
- 所有业务动作建议均标注 `【待人工审核】`
- 报告结论与生成的 CSV/JSON 证据一致
- 推荐图表（KPI 卡片、热力图、箱线图、条形图）已生成
- 完整交付包（业务场景、流程图、工具清单、管理层摘要）已输出
- 历史对比被阻断时已标注「口径变化待确认」
