# 数据质量规则 (DQC) 设计规范

## 定位

横切于 ODS / DIM / DWD / DWS / ADS 五层的数据质量保障机制。任何表的设计文档**必须连同 DQC 规则一起交付**，否则视为设计未完成。

DQC（Data Quality Control）的核心价值：
- **早发现**：问题在数据进入下游前被拦截，避免污染传播
- **可追溯**：每条业务方的数据质疑都能定位到具体规则与告警
- **可审计**：合规审查、数据资产盘点、SLA 报告的事实基础

## 三层规则模型

| 规则层 | 关注点 | 触发频率 | 实现成本 |
|--------|--------|---------|---------|
| 表级 | 数据是否到了、量级正不正常 | 每次入仓后 | 低 |
| 字段级 | 字段值是否合法 | 每次入仓后 | 中 |
| **业务级** | 跨表是否一致、业务公式是否成立 | 每次入仓后 | 高（需写 SQL） |

> **业务级规则比字段级规则重要 10 倍**：字段级（非空、唯一）多数 ETL 框架/约束会兜底；真正会让业务方半夜电话的是跨表一致性问题（订单金额 ≠ 明细加总、漏斗反向递增）。

## 各分层 DQC 重点

| 层 | 必备规则类型 | 典型规则示例 |
|---|---|---|
| ODS | 新鲜度、行数下限、空分区检测 | dt 分区每日 02:00 前到达；行数 ≥ 昨日 90%；连续 3 天无数据告警 |
| DIM | 自然键唯一、当前有效行数=1（拉链表）、枚举值合法 | dim_user_info 同一 user_id 至多 1 条 is_current=1；gender ∈ {男,女,未知} |
| DWD | 业务主键唯一、维度外键引用完整、业务公式校验 | order_id 唯一；user_id 必须在 dim_user_info 中存在；订单金额 = 明细加总 |
| DWS | 粒度键唯一、指标值域、口径一致性 | (user_id, dt) 组合唯一；order_amount_1d ≥ 0；同指标在多表口径一致 |
| ADS | 指标同环比波动、上游覆盖率 | 日 GMV 周环比绝对值 ≤ 50%（异常告警）；ADS 用户数 ≤ DIM 用户总数 |

## 业务级规则（最关键，每张事实表至少 1 条）

业务级规则覆盖三类典型场景：

### A. 跨表一致性

父子表加总应相等：
- **订单金额一致性**：`dwd_trade_order_fact.order_amount = SUM(dwd_trade_order_item_fact.item_amount)` over `order_id`，容差 ≤ 0.01
- **库存平衡**：`yesterday.stock + today.in - today.out = today.stock`
- **会计借贷平衡**：`SUM(debit) = SUM(credit)` over voucher_id

### B. 父子粒度一致性

汇总层结果与上游加总应相等：
- **GMV 父子一致**：`SUM(dws_trade_user_order_1d.order_amount_1d) = ads_daily_gmv_report.gmv`
- **去重计数一致性**：`COUNT(DISTINCT user_id)` 在 DWD 与 DWS 应一致

### C. 业务漏斗递减

漏斗各环节应单调不增：
- **电商漏斗**：UV ≥ 加购 UV ≥ 下单 UV ≥ 支付 UV ≥ 复购 UV
- **教育漏斗**：注册数 ≥ 登录数 ≥ 选课数 ≥ 完课数
- **金融漏斗**：申请数 ≥ 审批通过数 ≥ 放款数 ≥ 还款数

发现反向（如下单 UV > 加购 UV）必为埋点或建模问题。

## DQC 规则定义模板（DSL 风格）

可对齐 dbt tests / Great Expectations / DataHub Assertion 等工具：

```yaml
# 表级
- table: dwd_trade_order_fact
  type: freshness
  rule: dt 分区在 T+1 03:00 前可用
  severity: P1

- table: dwd_trade_order_fact
  type: row_count
  rule: row_count >= row_count_yesterday * 0.9
  severity: P1

# 字段级
- table: dwd_trade_order_fact
  column: order_id
  type: unique_not_null
  severity: P0

- table: dwd_trade_order_fact
  column: order_amount
  type: range
  rule: order_amount >= 0
  severity: P0

- table: dwd_trade_order_fact
  column: user_id
  type: foreign_key
  rule: exists in dim_user_info where is_current = 1
  severity: P0

# 业务级
- name: 订单金额一致性
  type: cross_table_consistency
  rule: |
    SELECT order_id
    FROM dwd_trade_order_fact o
    LEFT JOIN (
      SELECT order_id, SUM(item_amount) AS sum_item
      FROM dwd_trade_order_item_fact
      WHERE dt = '${bizdate}'
      GROUP BY order_id
    ) i ON o.order_id = i.order_id
    WHERE o.dt = '${bizdate}'
      AND ABS(o.order_amount - COALESCE(i.sum_item, 0)) > 0.01
  expect: result_count = 0
  severity: P0

- name: 日 GMV 周环比波动
  type: anomaly_detection
  rule: ABS(gmv_today - gmv_7days_ago) / gmv_7days_ago < 0.5
  severity: P2
```

## 工具映射（参考）

| 规则类型 | dbt | Great Expectations | DataHub | 自研 SQL |
|---|---|---|---|---|
| 唯一/非空 | `unique` / `not_null` | `expect_column_values_to_be_unique` | Assertion: Field | `SELECT col, COUNT(*) ... HAVING > 1` |
| 值域 | `accepted_values` / `dbt_utils.expression_is_true` | `expect_column_values_to_be_in_set` | Assertion: Field | `WHERE col NOT BETWEEN ...` |
| 外键引用 | `relationships` | `expect_column_values_to_be_in_set` (动态) | Assertion: Cross-Table | `LEFT JOIN ... WHERE ref IS NULL` |
| 跨表一致性 | 自定义 SQL test | `custom expectation` | Assertion: Custom | 见上 DSL 示例 |
| 同环比波动 | `dbt_expectations.expect_column_value_to_be_within_n_stdevs` | `expect_column_kl_divergence_to_be_less_than` | Monitor | 自定义 SQL |

## 告警分级与处置

| 等级 | 含义 | 处置策略 |
|---|---|---|
| **P0（阻断）** | 业务主键重复、外键缺失、金额一致性破坏 | **阻断下游任务**，由数据 oncall 立即介入 |
| **P1（必修）** | 新鲜度延迟、行数异常、关键字段空值率超阈值 | 当日修复，不阻断下游但触发告警 |
| **P2（观察）** | 同环比波动、长尾字段空值率上升 | 入观察池，每周复盘 |

> **P0 必须阻断**：让"先有数据后修问题"成为反模式。坏数据进入下游的修复成本是阻断成本的 10–100 倍。

## DQC 在设计文档中的呈现

每张表设计章节末尾追加 DQC 规则块：

```markdown
#### dwd_trade_order_fact

[DDL...]

**DQC 规则**：
- 表级：T+1 03:00 前到达；行数 ≥ 昨日 90% [P1]
- 字段级：order_id UNIQUE NOT NULL [P0]；order_amount ≥ 0 [P0]
- 业务级：order_amount = SUM(item_amount) over order_id（容差 0.01）[P0]
```

或在文档末尾用三个汇总表（表级/字段级/业务级）统一呈现，参见 `templates/design-output.md` 第九章。

## 注意事项

- **DQC 规则越早写越好**：在表设计阶段就把规则写出来，能反向暴露建模缺陷（如外键无法对应说明 DIM 缺表）
- **不要堆砌规则**：30 张表配 200 条规则会导致告警疲劳。每张事实表 5–10 条已足够（含 1 条业务级）
- **业务级规则需要业务方确认**：规则即业务定义，必须经业务负责人评审签字，否则规则违反时谁背锅都说不清
- **历史数据例外**：上线前的历史脏数据应被允许豁免（标记 known_issue），新增数据严格执行
- **告警必须有 owner**：每条规则必须明确数据 oncall + 业务联系人，否则 P0 告警没人处理就是"狼来了"
