# ADS 层（Application Data Service，数据应用层）设计规范

## 定位

直接面向最终业务需求，提供可直接消费的指标数据。对接 BI 报表、数据大屏、App 接口等。这是唯一完全由业务需求驱动的层。

## 命名规范

```
ads_{应用场景}_{描述}
```

示例：
- `ads_daily_trade_report` — 每日交易报表
- `ads_user_retention_analysis` — 用户留存分析
- `ads_realtime_gmv_dashboard` — 实时 GMV 大屏
- `ads_product_ranking_top100` — 商品排行 Top100
- `ads_user_rfm_tag` — 用户 RFM 标签表
- `ads_funnel_register_to_pay` — 注册到支付转化漏斗

命名时让表名直接体现业务含义，因为 ADS 表直接面向业务方。

## 设计方法

### 需求驱动

ADS 表的设计流程与其他层相反 —— 从需求出发倒推：

1. 明确报表/应用的具体需求（要展示什么指标、什么维度）
2. 确定需要的维度和指标字段
3. 从 DWS 层取数组装（**优先**），必要时才下探到 DWD

### 表结构 = 报表结构

ADS 表的结构应直接对应前端展示：
- 字段名即展示的列名（用中文注释对应）
- 一行数据即报表的一行
- 尽量减少下游处理逻辑

### 常见 ADS 表类型

| 类型 | 说明 | 典型字段 |
|------|------|---------|
| 汇总报表 | 多维汇总指标 | 日期、维度、各指标值 |
| 排行榜 | TopN 排名 | 排名、实体ID、指标值 |
| 画像标签 | 用户/商品标签体系 | 实体ID、各标签值、标签权重 |
| 漏斗分析 | 转化路径分析 | 步骤名、人数、转化率 |
| 留存分析 | N日留存矩阵 | 注册日期、第N日留存人数、留存率 |
| 预警指标 | 阈值监控 | 指标名、当前值、阈值、是否异常 |
| 对比分析 | 同环比分析 | 日期、指标值、同比值、环比值、变化率 |

## 数据加工方式

取数优先级：

```
DWS → DWD → 绝不从 ODS 取数
```

- **优先从 DWS 取数**：简单的 SELECT + WHERE + JOIN，这是最常见的模式
- **必要时从 DWD 取数**：DWS 未覆盖的细粒度需求
- **绝不从 ODS 取数**：如果需要穿透到 ODS，说明 DWD/DWS 设计有缺陷，优先补全上游

ADS 层的 SQL 应尽量简洁（通常不超过 3 张表关联）。如果组装逻辑复杂，考虑在 DWS 层补充中间表。

## DDL 模板

```sql
CREATE TABLE ads_{scenario}_{description} (
    -- ===== 维度字段 =====
    stat_date       STRING      COMMENT '统计日期',
    area_name       STRING      COMMENT '地区名称',

    -- ===== 指标字段 =====
    order_count     BIGINT      COMMENT '订单数',
    order_amount    DECIMAL(16,2) COMMENT '订单金额',
    pay_rate        DECIMAL(5,4) COMMENT '支付转化率',

    -- ===== 同环比（可选） =====
    order_amount_wow DECIMAL(10,4) COMMENT '订单金额周环比',

    -- ===== 元数据 =====
    etl_time        TIMESTAMP   COMMENT 'ETL处理时间'
) COMMENT '{应用场景描述}'
PARTITIONED BY (dt STRING COMMENT '数据日期')
STORED AS ORC
;
```

## 分区策略

- 按 `dt`（日期）分区
- 部分报表可增加业务维度分区（如按城市 `city`、按渠道 `channel`），视查询模式决定

## 数据输出

ADS 层数据的典型消费方式：

| 消费方式 | 适用场景 | 技术方案 |
|---------|---------|---------|
| 同步到 MySQL/PostgreSQL | BI 工具查询 | DataX / Sqoop |
| 推送到 Redis | 实时接口 | 自定义同步程序 |
| 推送到 Elasticsearch | 搜索/聚合分析 | Logstash / 自定义 |
| 导出 Excel/CSV | 邮件报表 | 调度平台导出 |
| 写入消息队列 | 下游系统消费 | Kafka Producer |

## 注意事项

- ADS 表数量会随业务需求增长，这是正常的。但要定期清理不再使用的表。
- ADS 层 SQL 应尽量简洁。如果一个 ADS 表的加工 SQL 超过 50 行，考虑是否该在 DWS 层补充中间表。
- 字段注释要清晰易懂，因为 ADS 直接面向业务方，他们可能不懂技术术语。
- 比率类指标（如转化率、留存率）在 ADS 层计算，使用 DWS 层提供的分子分母。
- 考虑数据时效性：每日批量报表 vs 实时/准实时大屏，技术方案不同。
