# 巡检 Workflow 流程说明

## 1. 流程图

```text
[1] 数据读取
   │  pandas read_csv，多编码探测（utf-8/gbk/latin-1）+ zip 自动解压
   ▼
[2] 字段校验
   │  校验必需字段：hotel, is_canceled, adr, lead_time, market_segment,
   │  distribution_channel, deposit_type, customer_type,
   │  reserved_room_type, assigned_room_type
   │  → 缺任一即阻断核心结论（blocker）
   ▼
[3] 数据清洗（留痕）
   │  children 缺失填 0、country 缺失填 Unknown
   │  company/agent 不填值，转 has_company/has_agent 标志（保留缺失语义）
   │  重复行 report_only 不删除，写入 cleaning_log.csv 留痕
   │  每步动作写入 quality/data_quality.csv
   ▼
[4] 派生字段构造
   │  total_guests = adults + children + babies
   │  total_stays = stays_in_weekend_nights + stays_in_week_nights
   │  is_room_changed = (reserved_room_type != assigned_room_type)
   │  has_agent / has_company / adr_anomaly_flag / adr_bucket
   ▼
[5] 指标计算（分子/分母/样本量标记）
   │  总体取消率、5 维分组取消率、ADR 中位数/P95/P99
   │  房型变更率、停车位需求率、特殊请求率
   ▼
[6] 异常识别
   │  交叉分析（hotel×market_segment 等 6 组）
   │  高取消率组合、高 ADR 异常清单、小样本高波动组合
   │  深度诊断：趋势/集中度/风险放大器
   ▼
[7] 报告输出
   │  hotel_watchdog_report.md（一页式，11 区块）
   │  charts/（5 图表，中文字体自动探测）
   │  risk-tables/、detail-tables/、deep-diagnosis/
   ▼
[8] 人工接管（human-in-the-loop 兜底）
   │  Non Refund 押金类型取消率 ≥90% → 不得直接下"押金策略失败"结论
   │  ADR 极端值、小样本高波动、口径变化、reservation_status 泄漏
   │  → 全部写入 manual_review 事项，待业务团队确认后执行
```

## 2. 关键门控（停止条件）

1. 缺 `hotel`/`is_canceled` → 阻断核心结论
2. `is_canceled` 取值非法（非 0/1） → 停止计算取消率
3. `adr <= 0` 占比 > 5% → 阻断收益判断，保留取消率分析
4. 口径变化 → 阻断历史对比，标注「口径变化待确认」
5. 重复行默认 report_only，未经人工确认不删除

## 3. 人工接管点

- 数据泄漏风险（reservation_status 字段已排除出预警输入，仅用于一致性审计）
- ADR 极端值（>P99）单独输出 `adr_anomaly_list.csv`
- 样本量不足但波动剧烈的组合（n<30 且取消率 ≥50%）
- 口径变化（无法与历史报告对比）
- 业务策略异常（Non Refund 押金类型实测取消率 ≥90% → 「不得直接下押金策略失败结论」）
