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

### 事实表设计

四步走：

1. **确定业务过程**：一个事实表对应一个业务过程（下单、支付、退货...）
2. **确定粒度**：最细粒度（一行 = 一次业务事件）
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
