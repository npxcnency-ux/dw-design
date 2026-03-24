# DWS 层（Data Warehouse Service，数据服务层/汇总层）设计规范

## 定位

基于 DWD 层数据，按业务主题进行轻度汇总，构建公共指标层。核心价值是 **复用** —— 多个 ADS 报表共享同一套指标计算逻辑。

## 命名规范

```
dws_{业务域}_{主题}_{粒度}_{统计周期}
```

示例：
- `dws_trade_user_order_1d` — 交易域-用户粒度-下单主题-日汇总
- `dws_trade_user_order_nd` — 交易域-用户粒度-下单主题-N日汇总
- `dws_trade_shop_sale_1d` — 交易域-店铺粒度-销售主题-日汇总
- `dws_user_action_1d` — 用户域-用户行为主题-日汇总
- `dws_traffic_page_visit_1d` — 流量域-页面粒度-访问主题-日汇总

### 统计周期约定

| 后缀 | 含义 | 数据特点 |
|------|------|---------|
| `1d` | 日 | 每日增量统计 |
| `nd` | 最近 N 天 | 滚动窗口，需参数化 N |
| `1w` | 周 | 每周统计 |
| `1m` | 月 | 每月统计 |
| `td` | 截至当日（历史累计） | 全量累计 |

## 设计方法

### 1. 确定主题域

按业务主题组织，常见主题：

| 主题域 | 涵盖内容 |
|--------|---------|
| 交易域 | 订单、支付、退款、物流 |
| 用户域 | 注册、活跃、留存、画像 |
| 流量域 | PV、UV、转化、路径 |
| 商品域 | 上架、销量、库存、评价 |
| 营销域 | 优惠券、活动、推广 |

### 2. 确定粒度

每张 DWS 表有且仅有一个统计粒度：

- 用户粒度（一行 = 一个用户的一天汇总）
- 商品粒度（一行 = 一个商品的一天汇总）
- 店铺粒度（一行 = 一个店铺的一天汇总）
- 交叉粒度（一行 = 一个用户在一个店铺的一天汇总）

粒度字段就是表的 GROUP BY 字段。

### 3. 确定指标

每张表包含该粒度下的所有公共指标：

| 指标类别 | 示例 | 聚合方式 |
|----------|------|---------|
| 计数类 | 下单次数、支付次数、退款次数 | COUNT |
| 金额类 | 下单金额、支付金额、优惠金额 | SUM |
| 去重计数 | 下单用户数、购买 SKU 数 | COUNT DISTINCT |
| 极值类 | 最大单笔金额、首次下单时间 | MAX / MIN |

比率类指标（如支付转化率 = 支付次数/下单次数）建议不在 DWS 存储，而是存分子分母让 ADS 层计算，这样更灵活。

### 4. 宽表设计

DWS 层的核心产出是 **宽表** —— 将同一粒度下多个业务过程的指标汇聚到一张表：

```
dws_trade_user_order_1d:
  user_id              -- 粒度字段
  order_count_1d       -- 下单次数
  order_amount_1d      -- 下单金额
  order_sku_count_1d   -- 下单SKU数
  pay_count_1d         -- 支付次数
  pay_amount_1d        -- 支付金额
  refund_count_1d      -- 退款次数
  refund_amount_1d     -- 退款金额
  dt                   -- 日期分区
```

指标字段命名建议加统计周期后缀（`_1d`、`_nd`），方便区分不同周期的同名指标。

## 数据加工方式

- 从 DWD 层事实表聚合：`GROUP BY 粒度字段` + 聚合函数
- 可关联 DIM 维度表补充维度属性（如用户等级、商品类目）
- N 日汇总表通常每日全量重算最近 N 天数据

典型 SQL 模式：
```sql
INSERT OVERWRITE TABLE dws_trade_user_order_1d PARTITION (dt = '${bizdate}')
SELECT
    user_id,
    COUNT(order_id) AS order_count_1d,
    SUM(order_amount) AS order_amount_1d,
    COUNT(DISTINCT sku_id) AS order_sku_count_1d
FROM dwd_trade_order_fact
WHERE dt = '${bizdate}'
GROUP BY user_id;
```

## DDL 模板

```sql
CREATE TABLE dws_{domain}_{subject}_{granularity}_{period} (
    -- ===== 粒度字段 =====
    user_id             BIGINT      COMMENT '用户ID',

    -- ===== 维度属性（可选，从维度表补充） =====
    user_level          STRING      COMMENT '用户等级',

    -- ===== 指标字段 =====
    order_count_1d      BIGINT      COMMENT '下单次数',
    order_amount_1d     DECIMAL(16,2) COMMENT '下单金额',
    pay_count_1d        BIGINT      COMMENT '支付次数',
    pay_amount_1d       DECIMAL(16,2) COMMENT '支付金额',

    -- ===== 元数据 =====
    etl_time            TIMESTAMP   COMMENT 'ETL处理时间'
) COMMENT '{主题描述}-{粒度}-{周期}汇总表'
PARTITIONED BY (dt STRING COMMENT '数据日期')
STORED AS ORC
;
```

## 分区策略

- 按 `dt`（日期）分区
- N 日 / 月汇总表通常只保留当日分区（每日全量重算）
- 日汇总表保留历史分区

## 注意事项

- **指标口径在此层统一定义**：同一指标只在一张 DWS 表中计算。如果发现同一指标出现在多张 DWS 表中且计算逻辑不同，必须收敛。
- **避免过度汇总**：DWS 应保留足够的维度字段，确保下游灵活使用。过度汇总到只剩一个指标值会降低复用性。
- **宽表字段控制**：单表建议不超过 100 个字段。超过则按主题拆分。
- **复用性检验**：如果一张 DWS 表只被一张 ADS 表使用，考虑合并到 ADS 或重新审视 DWS 的粒度划分。
- **比率类指标存分子分母**：不直接存比率值。如支付转化率，存 pay_count 和 order_count，由 ADS 计算 pay_count/order_count。
