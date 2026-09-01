# agentskil-hotel-watch

一、项目定位
把“每周/每月重复发生的酒店经营巡检工作”抽象为可执行 workflow，并进一步封装为一个可复用的数据分析 skill。
收益分析是重要模块，但不应喧宾夺主；取消率预警才是这份数据最稳定、最可自动化的业务问题。

二、数据集真实情况
项目	数据情况	
数据规模	119,390 行，32 个字段	足够支撑多维分析与智能体设计
酒店类型	City Hotel / Resort Hotel	 适合做酒店类型差异、渠道差异与取消风险对比
核心目标字段	is_canceled	适合作为取消率预警主线
关键经营字段	adr, lead_time, market_segment, distribution_channel, customer_type, deposit_type	适合做取消率、收益与渠道结构联动分析
资源需求字段	reserved_room_type, assigned_room_type, required_car_parking_spaces, total_of_special_requests	适合做资源压力与服务需求巡检
时间字段	arrival_date_year/month/week/day, reservation_status_date	适合做周期性巡检，但不适合宣称严格未来预测

三、关键数据风险与处理规则
让智能体在自动处理数据时不做危险决策。因此本项目必须把以下数据风险写入 workflow 和 skill 停止/人工接管规则。
数据问题	实际情况	必须写入文档的处理规则
重复记录	约 31,994 条完全重复行	不得默认删除。由于缺少 booking_id，重复行可能是团体预订、脱敏后表面重复或真实重复，必须先说明口径。
company 缺失	约 94.31% 缺失	不作为主分析字段；可剔除，或构造 has_company 作为辅助变量。
agent 缺失	约 13.69% 缺失	不直接当连续变量分析；建议构造 has_agent，或填充 Unknown。
country 缺失	488 条缺失	可填充 Unknown，用于国家维度统计时需说明。
children 缺失	4 条缺失	可填 0，但要说明对 total_guests 的影响。
ADR 异常	最小值 -6.38，最大值 5400，0 值约 1,959 条	不得只算均值；需标记 adr<=0 与极端 ADR，建议输出均值、中位数、P95/P99。
缺少 booking_id	无唯一订单 ID	不能把去重作为默认清洗动作，必须作为数据口径风险提示。
reservation_status	与取消结果高度相关	不可作为取消预测/预警输入字段，避免数据泄漏。

四、项目背景
你是某连锁酒店集团的数据分析智能体设计师。运营团队每周都要重复检查取消率、房价异常、渠道结构、客户类型、房型匹配和资源压力。过去这些工作依赖人工筛表、透视表和临时图表，效率低、口径不稳定、异常发现不及时。
你的任务不是简单完成一次分析，而是为酒店运营团队设计一个“酒店预订取消率预警与收益巡检智能体”。该智能体需要在新一周/新一月预订数据进入时，自动完成字段校验、指标计算、异常识别、报告生成，并标记需要人工复核的情况。

五、核心业务问题
1.本期取消率是否异常？异常主要来自哪类酒店、渠道、市场细分、押金类型或客户类型？
2.取消率是否与 lead_time、deposit_type、market_segment、distribution_channel、customer_type 等变量存在明显关系？
3.ADR 是否存在异常值或结构性变化？均值是否被极端值污染？
4.是否存在“高取消率 + 高 ADR”或“高取消率 + 长提前期”的重点风险组合？
5.房型变更、停车位需求、特殊请求是否集中在某些酒店、月份、渠道或客群？
6.哪些异常可以自动输出预警？哪些异常必须交给人工复核？

六、最终交付物
交付物	具体要求
1. 重复性工作定义	说明酒店运营团队为什么需要周期性取消率与收益巡检。
2. Workflow 图/流程说明	至少包含数据读取、字段校验、清洗、指标计算、异常识别、报告输出、人工接管。
3. Skill 文档	写出 skill 名称、适用场景、输入、输出、工具、执行步骤、停止条件和人工接管点。
4. 工具清单	明确需要文件读取、Python 分析、图表生成、报告生成等工具；不得滥用无关工具。
5. 示例输出	至少包含取消率预警、ADR 异常、资源匹配巡检、人工复核清单。
6. 风险说明	说明数据泄漏、重复行、ADR 极端值、样本量不足等风险。

七、Workflow 设计
必须把酒店经营巡检拆成稳定 workflow，而不是直接让大模型“分析这张表”。流程如下：
7.读取 hotel_bookings.csv。
8.校验必需字段：hotel, is_canceled, adr, lead_time, market_segment, distribution_channel, deposit_type, customer_type, reserved_room_type, assigned_room_type。
9.检查缺失值：company, agent, country, children，并按规则处理。
10.检查重复行，但不得默认删除，必须输出重复风险说明。
11.构造辅助字段：total_guests, total_stays, is_room_changed, has_agent, has_company, adr_anomaly_flag。
12.标记异常：total_guests=0, total_stays=0, adr<=0, adr 极端高值。
13.计算核心指标：总体取消率、分组取消率、ADR 中位数/P95、房型变更率、停车位需求率、特殊请求率。
14.完成取消率分组监控：hotel、market_segment、distribution_channel、deposit_type、customer_type。
15.完成交叉分析：至少 2 组取消率交叉组合。
16.输出异常对象清单：高取消率、高 ADR 异常、样本量过小但波动剧烈的组合。
17.生成一页式巡检报告。
18.触发人工复核：数据泄漏风险、ADR 极端值、样本量不足、口径变化、业务策略相关异常。

八、数据处理与特征构造任务
任务	要求	
重复行处理	统计完全重复行数量，但不得默认删除。若删除，必须写明依据和影响。	智能体不能自动做高风险数据决策。
缺失值处理	company、agent、country、children 按字段性质分别处理。	缺失处理必须服务业务口径，而不是机械填补。
客人数量	构造 total_guests = adults + children + babies，并标记 total_guests=0。	识别不符合业务常识的记录。
入住时长	构造 total_stays = stays_in_weekend_nights + stays_in_week_nights，并标记 total_stays=0。	入住时长为 0 的订单需作为异常提示。
房型变化	构造 is_room_changed = reserved_room_type != assigned_room_type。	用于资源匹配巡检。
ADR 异常	标记 adr<=0、P99 以上极端值；计算均值、中位数、P95/P99。	收益指标不能只看平均值。
数据泄漏	reservation_status 不参与取消风险输入。	识别预测任务中的泄漏变量。

九、经营分析任务
任务 1：取消率主线分析
必须输出以下维度的取消率：
总体取消率
hotel × is_canceled
market_segment × is_canceled
distribution_channel × is_canceled
deposit_type × is_canceled
customer_type × is_canceled
取消率不是单纯越高越“异常”，还要结合样本量、业务策略和字段口径。Non Refund 取消率极高时，不应直接下结论说“押金策略失败”，而要提示人工复核是否存在业务定义或状态口径问题。

任务 2：交叉分析
至少完成以下交叉分析中的 2 组，并说明业务含义：
hotel × market_segment × is_canceled
deposit_type × customer_type × is_canceled
lead_time 分层 × is_canceled
adr 分层 × is_canceled
total_of_special_requests × is_canceled
distribution_channel × deposit_type × is_canceled
交叉分析必须回答：哪个组合风险最高？样本量是否足够？是否值得自动预警？是否需要人工复核？

任务 3：收益与 ADR 异常巡检
按 hotel、market_segment、distribution_channel、customer_type 计算 ADR 中位数、P95、P99。
标记 adr<=0 与 adr 高于 P99 的记录。
识别“高取消率 + 高 ADR”的风险组合。
输出一张 ADR 异常清单。

任务 4：资源匹配巡检
房型变更率 = reserved_room_type != assigned_room_type 的比例。
停车位需求率 = required_car_parking_spaces > 0 的比例。
特殊请求率 = total_of_special_requests > 0 的比例。
分析这些资源压力是否集中在某类酒店、月份、渠道或客群。

十、看板/报告结构
区域	内容	推荐图表
顶部 KPI	总体取消率、City Hotel 取消率、Resort Hotel 取消率、ADR 中位数、房型变更率	KPI 卡片
取消率诊断	按 hotel、market_segment、distribution_channel、deposit_type 展示取消率	分组条形图 / 热力图
交叉风险	识别高取消组合，如 hotel × market_segment、deposit_type × customer_type	交叉热力图
ADR 异常	ADR 分布、P95/P99、极端值清单	箱线图 / 异常表
资源匹配	房型变更、停车位需求、特殊请求分布	条形图 / 明细表
人工复核清单	样本量小但异常高、ADR 极端值、疑似泄漏或口径变化	表格


