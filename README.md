# Database Engineer — 数据库工程师技能库

> 面向独立开发者和后端工程师的数据库技能集合。覆盖 MySQL、PostgreSQL、MongoDB 三大数据库引擎的查询优化、索引设计、性能诊断和可观测性。

## 技能清单

| 技能 | 描述 | 安装量 | 来源 |
|------|------|--------|------|
| mysql | MySQL 查询优化、索引策略、Schema 设计、复制与高可用 | 6.6K | [skills.sh](https://skills.sh/planetscale/database-skills/mysql) |
| postgres-patterns | PostgreSQL 高级查询、索引类型、并发控制、扩展生态 | 7.6K | [skills.sh](https://skills.sh/affaan-m/everything-claude-code/postgres-patterns) |
| mongodb-query-optimizer | MongoDB 聚合管道优化、索引策略、Schema 设计、性能诊断 | 3.5K | [skills.sh](https://skills.sh/mongodb/agent-skills/mongodb-query-optimizer) |
| database-observability | 数据库监控指标、慢查询告警、性能基线、容量规划 | 1.9K | [skills.sh](https://skills.sh/grafana/skills/database-observability) |

## 工作流

```
MySQL ────────┐
PostgreSQL ───┼──→ Database Observability
MongoDB ──────┘    (监控与告警)
(引擎选择与优化)
```

## 安装

```bash
# 安装全部数据库技能
npx skills add skills-repo/database-engineer

# 或按需安装单个技能
npx skills add skills-repo/database-engineer@mysql
npx skills add skills-repo/database-engineer@postgres-patterns
npx skills add skills-repo/database-engineer@mongodb-query-optimizer
npx skills add skills-repo/database-engineer@database-observability
```

## 与本组织其他仓库的关系

- **backend-developer** — 后端架构和 API 设计，本仓库聚焦数据库引擎深度优化
- **data-scientist** — 数据分析和可视化，本仓库聚焦数据库运维和性能
- **devops-engineer** — CI/CD 和基础设施，本仓库聚焦数据库层可观测性