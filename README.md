# DW-Design：数据仓库分层设计 Skill

一个 Claude Code Skill，帮助用户从原始数据源出发，设计完整的数仓分层架构和每一层的详细表结构。

## 安装

将本仓库克隆到任意目录，然后在该目录下启动 Claude Code 即可自动加载 Skill：

```bash
git clone <repo-url> dw-design
cd dw-design
claude
```

Skill 文件位于 `.claude/skills/dw-layered-design/`，Claude Code 启动时会自动识别。

## 使用

在 Claude Code 对话中直接描述你的数仓设计需求即可，Skill 会自动触发。示例：

```
# 完整数仓设计
我们公司是做跨境电商的，有订单表 orders、用户表 users、商品表 products，
老板要看 GMV、退货率、复购率，帮我设计一套完整的数仓

# 单层表设计
帮我设计 DWS 层的宽表，上游 DWD 有 dwd_trade_order_fact，
DIM 层有 dim_user_info，我需要一张用户维度的交易汇总宽表

# 从源数据建分析体系
我有一份 CSV，表头是 student_id,course_id,score,submit_time，
想建个分析体系看各科通过率和学生学习轨迹
```

Skill 触发后会引导你补充缺失信息（数据源、业务场景、分析需求），然后逐层输出完整的表结构 DDL 和数据流向图。

## 架构概览

采用 **ODS → DIM → DWD → DWS → ADS** 五层架构：

```
源系统 ──→ ODS（贴源层）──→ DIM（公共维度层）──→ DWS（汇总层）──→ ADS（应用层）
                          └→ DWD（明细事实层）──┘
```

| 层级 | 职责 | 关键规则 |
|------|------|----------|
| **ODS** | 原样搬运源数据，不做业务逻辑处理 | 全量/增量同步，保留原始字段 |
| **DIM** | 管理所有公共维度表（`dim_` 前缀） | SCD 类型明确标注 |
| **DWD** | 仅包含事实表（`_fact` 后缀），清洗规范化 | 不含维度表，维度通过外键关联 DIM |
| **DWS** | 统一指标口径，构建复用宽表 | 比率类指标只存分子分母 |
| **ADS** | 直接对应业务需求的报表/应用表 | 绝不从 ODS 取数 |

## 设计原则

- **90% 原则**：90% 的业务需求通过 DWS/ADS 层查询即可满足
- **3 表原则**：单个需求对应的 SQL 不超过 3 张表关联
- **向上收敛**：从 ODS 到 ADS，表数量整体趋于收敛
- **DIM/DWD 边界清晰**：`dim_` 前缀只出现在 DIM 层，`_fact` 后缀只出现在 DWD 层
- **比率指标**：DWS 层只存分子和分母，由 ADS 层计算比率值

## Skill 文件结构

```
.claude/skills/dw-layered-design/
├── SKILL.md                    # 主 skill（工作流程 + Gotchas）
├── references/
│   ├── ods-design.md           # ODS 层设计规范
│   ├── dim-design.md           # DIM 层设计规范（含 SCD 策略）
│   ├── dwd-design.md           # DWD 层设计规范（仅事实表）
│   ├── dws-design.md           # DWS 层设计规范（宽表、指标）
│   └── ads-design.md           # ADS 层设计规范（需求驱动）
├── templates/
│   └── design-output.md        # 输出文档模板（含 Mermaid 流向图）
└── evals/
    └── evals.json              # 测试用例
```

## 工作流程

1. **收集信息**：通过对话获取数据源、业务场景、分析需求
2. **设计分层架构**：确定每层表清单、数据流向、加工策略
3. **设计表结构**：按各层规范输出表名、字段、分区、加工方式、数据来源
4. **输出设计文档**：按模板生成架构总览、DDL、ETL 策略
5. **设计评审**：自检 8 项质量标准，不满足则回退优化

## 测试结果

基于 4 个场景（电商 DDL、在线教育自然语言、单层 DWS、汽车质量预警）的测试：

| 配置 | 通过率 | 平均 Tokens |
|------|--------|-------------|
| With Skill | **100%** | 54,689 |
| Baseline | 70% | 27,771 |

Skill 使模型在五层架构完整性、DIM/DWD 边界、SCD 标注、DDL 输出、Mermaid 图等方面全部达标。

## 适用场景

- 提供源数据（CSV、DDL、JSON Schema）构建报表/分析系统
- 设计完整 ODS/DIM/DWD/DWS/ADS 分层架构
- 设计某一层的表（如 DWS 宽表、维度表）
- 指标口径统一或数据链路规划
- 涉及多层数据加工和分析建模的场景

## 不适用场景

- ETL 调度编排（Airflow、DataX 等）
- Spark/Flink 性能调优
- 可视化工具配置（Grafana、Superset 等）
- OLTP 事务型数据库设计（MySQL ER 模型等）
