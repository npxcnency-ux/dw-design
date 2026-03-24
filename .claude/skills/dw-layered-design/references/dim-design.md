# DIM 层（Dimension，公共维度层）设计规范

## 定位

独立管理所有公共维度表，为 DWD 事实表及上层提供统一、标准、可复用的维度数据。DIM 层与 DWD 层同级，从 ODS 层加工而来。

DIM 层独立存在的价值：
- **统一管理**：维度表的生命周期（SCD）、数据清洗规则集中维护
- **跨域复用**：用户维度、时间维度等被多个业务域的事实表共同引用
- **职责分离**：DWD 专注事实表建模，DIM 专注维度表建模，各司其职

## 命名规范

```
dim_{维度实体}[_{补充说明}]
```

示例：
- `dim_user_info` — 用户维度表
- `dim_product_sku` — 商品 SKU 维度表
- `dim_area` — 地区维度表
- `dim_date` — 日期维度表
- `dim_channel` — 渠道维度表
- `dim_supplier` — 供应商维度表
- `dim_dealer` — 经销商维度表

## 维度表分类

### 按维度来源

| 类型 | 说明 | 示例 |
|------|------|------|
| 基础维度 | 直接从业务系统同步的实体 | 用户、商品、店铺、供应商 |
| 公共维度 | 跨业务域通用的维度 | 时间、地区、渠道 |
| 枚举维度 | 低基数编码映射 | 性别、订单状态、支付方式 |

### 按变化类型（SCD）

| SCD 类型 | 策略 | 适用场景 | 实现方式 |
|----------|------|---------|---------|
| Type 1 | 直接覆盖 | 不需要历史的属性（如用户昵称纠错） | 全量覆盖 |
| Type 2 | 拉链表（增加新行 + 有效期） | 需要追溯历史（如用户等级变化） | start_date / end_date / is_current |
| Type 3 | 增加新列 | 只需前后两个值（如上次地址、当前地址） | previous_xxx 列 |
| 静态 | 不变化 | 时间维度、地区维度 | 一次性初始化 |

选择依据：优先 Type 1（简单高效），需要历史追溯时才用 Type 2，Type 3 极少使用。

## 表结构设计

### 标准字段结构

| 字段分类 | 示例字段 | 说明 |
|----------|---------|------|
| 代理键（可选） | user_key | 代理主键，SCD Type 2 推荐使用 |
| 自然键 | user_id | 业务主键 |
| 维度属性 | user_name, gender, age, level | 描述性字段 |
| SCD 字段 | start_date, end_date, is_current | Type 2 需要 |
| 元数据 | etl_time | 技术字段 |

### 数据清洗规则

DIM 层对维度数据执行以下标准化：

1. **去重**：按自然键去重，保留最新记录
2. **空值处理**：维度属性空值填充为"未知"或约定默认值
3. **编码统一**：将源系统不同编码映射为统一业务编码
4. **类型统一**：多个来源的同一维度字段统一为标准类型
5. **层级补全**：层级维度（如商品类目）补全各级名称

## 常见维度表模板

### 用户维度表（SCD Type 2 示例）

```sql
CREATE TABLE dim_user_info (
    user_key        BIGINT      COMMENT '代理键',
    user_id         BIGINT      COMMENT '用户ID（自然键）',
    user_name       STRING      COMMENT '用户名',
    gender          STRING      COMMENT '性别',
    age             INT         COMMENT '年龄',
    phone           STRING      COMMENT '手机号（脱敏）',
    level           STRING      COMMENT '用户等级',
    register_time   TIMESTAMP   COMMENT '注册时间',
    register_channel STRING     COMMENT '注册渠道',
    start_date      STRING      COMMENT '生效日期',
    end_date        STRING      COMMENT '失效日期（9999-12-31表示当前有效）',
    is_current      INT         COMMENT '是否当前有效 1-是 0-否',
    etl_time        TIMESTAMP   COMMENT 'ETL处理时间'
) COMMENT '用户维度表（拉链表）'
STORED AS ORC
;
```

### 商品维度表（SCD Type 1 示例）

```sql
CREATE TABLE dim_product_sku (
    sku_id          BIGINT      COMMENT 'SKU ID',
    sku_name        STRING      COMMENT 'SKU名称',
    category_id     BIGINT      COMMENT '类目ID',
    category_name   STRING      COMMENT '类目名称',
    brand           STRING      COMMENT '品牌',
    price           DECIMAL(16,2) COMMENT '标准售价',
    status          STRING      COMMENT '商品状态',
    etl_time        TIMESTAMP   COMMENT 'ETL处理时间'
) COMMENT '商品SKU维度表'
PARTITIONED BY (dt STRING COMMENT '数据日期')
STORED AS ORC
;
```

### 日期维度表（静态维度）

```sql
CREATE TABLE dim_date (
    date_id         STRING      COMMENT '日期 yyyy-MM-dd',
    year            INT         COMMENT '年',
    quarter         INT         COMMENT '季度',
    month           INT         COMMENT '月',
    week_of_year    INT         COMMENT '年中第几周',
    day_of_week     INT         COMMENT '星期几(1-7)',
    day_of_month    INT         COMMENT '月中第几天',
    is_weekend      INT         COMMENT '是否周末 1-是 0-否',
    is_holiday      INT         COMMENT '是否节假日 1-是 0-否',
    holiday_name    STRING      COMMENT '节假日名称'
) COMMENT '日期维度表'
STORED AS ORC
;
```

## 分区策略

| 维度类型 | 分区方式 | 说明 |
|----------|---------|------|
| SCD Type 1 | 按 dt 分区（每日全量快照） | 简单、可回溯到任意历史日期 |
| SCD Type 2（拉链表） | 不分区 | 通过 start_date/end_date 管理历史 |
| 静态维度 | 不分区 | 一次加载，极少更新 |

## 数据加工方式

| 维度类型 | 加工方式 | 调度频率 |
|----------|---------|---------|
| SCD Type 1 | 全量覆盖 | 每日 |
| SCD Type 2 | 拉链表合并（新旧全量对比，识别变更行） | 每日 |
| 静态维度 | 初始化加载，仅在变更时手动更新 | 按需 |

## 注意事项

- **维度表一定要跨业务域复用**：如果一张维度表只被一个事实表使用，考虑退化为事实表中的退化维度
- **控制维度表宽度**：维度表字段数建议不超过 30 个。过多属性考虑拆分为主表 + 扩展表
- **敏感字段脱敏**：手机号、身份证等在 DIM 层即完成脱敏
- **枚举维度可以不建表**：低基数的状态码映射（如 1=待付款, 2=已付款）可以在 DWD 事实表中直接退化，不必单独建维度表
- **日期维度提前生成**：建议一次性生成未来 5-10 年的日期维度，避免运行时动态计算日期属性
