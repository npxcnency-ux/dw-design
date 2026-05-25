# DWD 层（Data Warehouse Detail，数据明细层）设计规范

## 定位

对 ODS 层数据进行清洗、规范化、整合，形成高质量的明细事实数据。以业务过程为建模核心，是数仓质量的基石。

DWD 层专注于 **事实表**，维度表由独立的 DIM 层管理。DWD 事实表通过外键关联 DIM 维度表。

## 命名规范

```
dwd_{业务域}_{业务过程}_fact[_{补充说明}]
```

示例：
- `dwd_trade_order_fact` — 交易域-下单事实表
- `dwd_trade_payment_fact` — 交易域-支付事实表
- `dwd_user_register_fact` — 用户域-注册事实表
- `dwd_traffic_page_view_fact` — 流量域-页面浏览事实表

业务域根据实际业务确定，常见：trade（交易）、user（用户）、product（商品）、traffic（流量）、common（公共）。

## 建模方法

采用 **维度建模（Kimball）** 方法。DWD 层专注事实表，维度表在 DIM 层（参见 `references/dim-design.md`）。

### 事实表三种类型（必须先选型）

Kimball 维度建模的事实表分三类，**粒度、ETL 模式、典型场景完全不同，混用会导致严重的建模错误**：

| 类型 | 粒度 | ETL 模式 | 典型场景 |
|---|---|---|---|
| 事务事实表 (Transaction) | 一次业务事件一行 | INSERT OVERWRITE 当日分区 | 下单、点击、支付流水、登录、加购 |
| 周期快照事实表 (Periodic Snapshot) | 实体 × 周期点一行 | 每周期点全量重算，按周期点分区 | 库存日快照、账户余额、设备日状态、用户当日画像 |
| 累积快照事实表 (Accumulating Snapshot) | 一个生命周期实例一行 | MERGE INTO 按业务主键回写 | 订单全生命周期、工单流转、保单审核、招聘流程 |

### 选型决策树

按以下顺序判断：

1. **数据写入后是否还会变化？**
   - 否 → 事务事实表（如点击事件、支付流水）
   - 是 → 进入 2

2. **变化是离散的"里程碑"还是连续的"状态"？**
   - 一个流程有有限个里程碑节点（如订单的下单→支付→发货→签收→完成）→ **累积快照事实表**
   - 状态在时间轴上连续演进，需要看任意时点的"当下值"（如库存余额、账户金额）→ **周期快照事实表**

3. **常见误用反模式**：
   - 用事务事实表 + "取最新状态"视图模拟累积快照 → 性能差、可读性差，半年后维护者看不懂
   - 用累积快照事实表存"加购点击" → 没有生命周期，应是事务事实表

### 事务事实表 DDL 模板

```sql
CREATE TABLE dwd_trade_order_fact (
    order_id        BIGINT        COMMENT '订单ID（业务主键）',
    user_id         BIGINT        COMMENT '用户ID（外键->dim_user_info）',
    product_id      BIGINT        COMMENT '商品ID（外键->dim_product_sku）',
    order_amount    DECIMAL(16,2) COMMENT '订单金额',
    quantity        INT           COMMENT '购买数量',
    order_type      STRING        COMMENT '订单类型（退化维度）',
    create_time     TIMESTAMP     COMMENT '下单时间',
    etl_time        TIMESTAMP     COMMENT 'ETL处理时间'
) COMMENT '交易域-下单事实表（事务事实表）'
PARTITIONED BY (dt STRING COMMENT '业务日期=create_date')
STORED AS ORC;
```

**ETL**：每日 `INSERT OVERWRITE TABLE ... PARTITION (dt='${bizdate}') SELECT ... WHERE create_date='${bizdate}'`。历史分区不变。

### 累积快照事实表 DDL 模板（订单全生命周期示例）

```sql
CREATE TABLE dwd_trade_order_lifecycle_fact (
    order_id          BIGINT        COMMENT '订单ID（业务主键，唯一）',
    user_id           BIGINT        COMMENT '用户ID',
    product_id        BIGINT        COMMENT '主商品ID',
    order_amount      DECIMAL(16,2) COMMENT '订单金额',
    -- ===== 里程碑时间字段（核心）=====
    created_time      TIMESTAMP     COMMENT '下单时间（必有）',
    paid_time         TIMESTAMP     COMMENT '支付时间（NULL=未支付）',
    shipped_time      TIMESTAMP     COMMENT '发货时间',
    signed_time       TIMESTAMP     COMMENT '签收时间',
    completed_time    TIMESTAMP     COMMENT '完成时间',
    cancelled_time    TIMESTAMP     COMMENT '取消时间',
    refunded_time     TIMESTAMP     COMMENT '退款时间',
    -- ===== 派生时长（便于下游直接用）=====
    pay_duration_sec  BIGINT        COMMENT '下单到支付时长（秒）',
    ship_duration_sec BIGINT        COMMENT '支付到发货时长（秒）',
    -- ===== 当前状态 =====
    current_status    STRING        COMMENT '当前状态：created/paid/shipped/signed/completed/cancelled/refunded',
    is_finished       INT           COMMENT '生命周期是否结束 1/0',
    etl_time          TIMESTAMP     COMMENT 'ETL处理时间'
) COMMENT '交易域-订单全生命周期事实表（累积快照）'
PARTITIONED BY (dt STRING COMMENT '快照日期，每日全量')
STORED AS ORC;
```

**ETL（MERGE 语义，每日运行）**：
```sql
-- 伪代码：以最新订单状态全量更新当日快照分区
INSERT OVERWRITE TABLE dwd_trade_order_lifecycle_fact PARTITION (dt = '${bizdate}')
SELECT
    o.order_id, o.user_id, o.product_id, o.order_amount,
    o.created_time,
    pay.pay_time           AS paid_time,
    ship.ship_time         AS shipped_time,
    sign.sign_time         AS signed_time,
    done.complete_time     AS completed_time,
    cancel.cancel_time     AS cancelled_time,
    refund.refund_time     AS refunded_time,
    UNIX_TIMESTAMP(pay.pay_time) - UNIX_TIMESTAMP(o.created_time)   AS pay_duration_sec,
    UNIX_TIMESTAMP(ship.ship_time) - UNIX_TIMESTAMP(pay.pay_time)   AS ship_duration_sec,
    CASE
      WHEN refund.refund_time IS NOT NULL THEN 'refunded'
      WHEN cancel.cancel_time IS NOT NULL THEN 'cancelled'
      WHEN done.complete_time IS NOT NULL THEN 'completed'
      WHEN sign.sign_time     IS NOT NULL THEN 'signed'
      WHEN ship.ship_time     IS NOT NULL THEN 'shipped'
      WHEN pay.pay_time       IS NOT NULL THEN 'paid'
      ELSE 'created'
    END AS current_status,
    CASE WHEN done.complete_time IS NOT NULL OR cancel.cancel_time IS NOT NULL
              OR refund.refund_time IS NOT NULL THEN 1 ELSE 0 END  AS is_finished,
    CURRENT_TIMESTAMP
FROM ods_mysql_orders o
LEFT JOIN ods_mysql_payments pay   ON o.order_id = pay.order_id
LEFT JOIN ods_mysql_shipments ship ON o.order_id = ship.order_id
-- ... 其他里程碑表 ...
WHERE o.is_active = 1;  -- 仅处理未归档订单
```

> 累积快照表**不按下单日分区**，按"快照日期"分区，每日全量重算未完结订单 + 当日新单。已完结订单（is_finished=1）30 天后可归档冷存以控制成本。

### 周期快照事实表 DDL 模板（库存日快照示例）

```sql
CREATE TABLE dwd_inventory_daily_snapshot_fact (
    warehouse_id    BIGINT        COMMENT '仓库ID',
    sku_id          BIGINT        COMMENT 'SKU ID',
    snapshot_date   STRING        COMMENT '快照日期 yyyy-MM-dd',
    stock_qty       INT           COMMENT '当日期末库存数',
    stock_amount    DECIMAL(18,2) COMMENT '当日期末库存金额',
    in_qty          INT           COMMENT '当日入库数',
    out_qty         INT           COMMENT '当日出库数',
    safety_stock    INT           COMMENT '当日安全库存',
    is_low_stock    INT           COMMENT '是否低于安全库存 1/0',
    etl_time        TIMESTAMP     COMMENT 'ETL处理时间'
) COMMENT '库存日快照事实表（周期快照）'
PARTITIONED BY (dt STRING COMMENT '快照日期')
STORED AS ORC;
```

**ETL**：每日 EOD 时按 `仓库 × SKU` 全量计算当日期末状态写入新分区。**关键约束**：(warehouse_id, sku_id, dt) 唯一。

### 事实表设计

四步走（**先确定类型，再走以下四步**）：

1. **确定业务过程**：一个事实表对应一个业务过程（下单、支付、退货...）
2. **确定粒度**：最细粒度（一行 = 一次业务事件 / 一个生命周期实例 / 一个实体在一个周期点）
3. **确定维度**：谁（用户）、什么（商品）、何时（时间）、何地（地区）、如何（渠道）
4. **确定度量**：金额、数量等可聚合的数值字段

事实表标准字段结构：

| 字段分类 | 示例字段 | 说明 |
|----------|---------|------|
| 业务主键 | order_id | 业务唯一标识 |
| 维度外键 | user_id, product_id, area_id | 关联 DIM 层维度表 |
| 度量值 | order_amount, quantity, discount_amount | 可聚合数值 |
| 退化维度 | order_type, channel_code | 低基数维度（无需独立维度表） |
| 业务时间 | create_time, pay_time | 业务发生时间 |
| 元数据 | dt, etl_time | 技术字段 |

### 维度退化

低基数维度（如订单类型、支付方式、渠道编码）可直接放入事实表作为退化维度，无需在 DIM 层建独立维度表。判断标准：基数 < 20 且无需独立维度属性时退化。

## 数据清洗规则

DWD 层执行以下标准化操作（按优先级排序）：

1. **去重**：处理源系统重复数据（按业务主键 + 业务时间去重）
2. **空值处理**：统一 NULL 策略
   - 数值型默认 0
   - 字符串型默认 ''
   - 时间型默认 '1970-01-01 00:00:00'
   - 根据业务含义决定是否保留 NULL（如"未填写"和"值为0"语义不同时保留 NULL）
3. **数据类型统一**：同一含义字段在不同来源中类型可能不同，在此统一
4. **编码映射**：将源系统编码映射为统一业务编码（如性别 M/F → 男/女）
5. **时间标准化**：统一时间格式和时区
6. **数据过滤**：过滤测试数据、无效数据（如金额为负的异常订单）

## 分区策略

- 事实表：按 `dt`（日期）分区

## DDL 模板

### 事实表

```sql
CREATE TABLE dwd_{domain}_{process}_fact (
    -- ===== 业务主键 =====
    order_id        BIGINT      COMMENT '订单ID',

    -- ===== 维度外键 =====
    user_id         BIGINT      COMMENT '用户ID',
    product_id      BIGINT      COMMENT '商品ID',

    -- ===== 度量值 =====
    order_amount    DECIMAL(16,2) COMMENT '订单金额',
    quantity        INT         COMMENT '购买数量',

    -- ===== 退化维度 =====
    order_type      STRING      COMMENT '订单类型',

    -- ===== 业务时间 =====
    create_time     TIMESTAMP   COMMENT '下单时间',

    -- ===== 元数据 =====
    etl_time        TIMESTAMP   COMMENT 'ETL处理时间'
) COMMENT '{业务过程}事实表'
PARTITIONED BY (dt STRING COMMENT '数据日期')
STORED AS ORC
;
```

## 注意事项

- DWD 是数仓质量的基石，数据质量问题 **必须** 在此层解决
- 不在 DWD 做聚合计算，保持最细粒度
- 一个业务过程对应一张事实表，不要把多个业务过程塞进一张事实表
- 事实表通过外键（user_id, product_id 等）关联 DIM 层维度表，不在 DWD 层重复维度属性
- 维度表统一在 DIM 层管理，DWD 层不创建 `_dim` 后缀的表
- **事实表类型不可混用**：一张事实表只能是事务/周期快照/累积快照中的一种。订单全生命周期必须用累积快照，不要用事务事实表配合"取最新状态"的视图绕开 —— 这是制造业、金融业建模常见反模式，性能与可读性双输。
