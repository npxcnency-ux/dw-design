# ODS 层（Operational Data Store，数据贴源层）设计规范

## 定位

数据从源系统进入数仓的第一站。保持与源系统结构一致，不做任何业务逻辑处理。

## 命名规范

```
ods_{来源系统}_{源表名}
```

示例：
- `ods_mysql_user_info` — 来自 MySQL 的用户信息表
- `ods_api_order_detail` — 来自 API 的订单明细
- `ods_kafka_click_event` — 来自 Kafka 的点击事件
- `ods_excel_finance_report` — 来自 Excel 的财务报表

来源系统标识要在项目内统一约定，保持一致。

## 表结构设计

### 字段规则

1. **完整保留源表所有字段**，字段名与源系统保持一致
2. **追加元数据字段**：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| dt | STRING | 数据日期分区，格式 yyyy-MM-dd |
| etl_time | TIMESTAMP | ETL 入库时间 |
| data_source | STRING | 数据来源标识 |

3. **不做任何清洗或转换**：字段类型与源系统保持一致，NULL 值原样保留

### 分区策略

- **dt 日期分区是必选项**
- 数据量大的表可增加二级分区（如按小时 `hour`、按来源 `source`）
- 拉链表可用 `start_date` 和 `end_date` 代替 dt

## 数据加工方式

| 场景 | 抽取方式 | 适用条件 |
|------|---------|---------|
| 数据量小、无增量标识 | 全量快照（每日全量覆盖） | 维度表、配置表（通常 <100 万行） |
| 有更新时间字段 | 增量抽取（按 update_time） | 事实表、流水表 |
| 实时数据流 | 准实时 / 实时写入 | Kafka、Binlog 等流式数据 |
| 有删除操作的表 | 全量快照或 Binlog CDC | 需要感知删除的业务表 |

选择依据：优先增量抽取以节省资源；源表无法提供增量标识时退回全量快照。

## DDL 模板

```sql
CREATE TABLE ods_{source}_{table_name} (
    -- ===== 源表字段（原样保留） =====
    id              BIGINT      COMMENT '主键ID',
    -- ... 其他源表字段 ...

    -- ===== 元数据字段 =====
    etl_time        TIMESTAMP   COMMENT 'ETL入库时间',
    data_source     STRING      COMMENT '数据来源标识'
) COMMENT '{源表中文名}-ODS'
PARTITIONED BY (dt STRING COMMENT '数据日期')
STORED AS ORC  -- 或 Parquet，根据技术栈选择
;
```

## 注意事项

- 每张 ODS 表都应有明确的数据源文档（源库、源表、负责人、接口方式）
- 建议对每张表记录首次入仓日期和数据量级估算
- ODS 表的数据质量问题在此层 **不处理、仅记录**，在 DWD 层统一清洗
- 源系统表结构变更（加字段、改类型）时，ODS 需要同步跟进，这是常见运维工作
