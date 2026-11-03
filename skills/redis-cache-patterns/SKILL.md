---
name: redis-cache-patterns
description: Redis 缓存设计：数据结构选型、Key 命名、Cache-Aside/Write-Through、标签失效与击穿防护
source:
  type: derived
  repo: redis/agent-skills
  path: skills/redis-core/SKILL.md
  updated: 2026-10-27
  url: https://github.com/redis/agent-skills/blob/main/skills/redis-core/SKILL.md
  also:
    - affaan-m/ecc@redis-patterns（7.6K 安装）https://github.com/affaan-m/ecc
    - upstash/skills@upstash-redis-js（24.4K 安装）https://github.com/upstash/skills
metadata:
  category: 缓存
  platform: Redis
  difficulty: 进阶
---

# Redis 缓存模式（数据结构 → 失效策略 → 防护）

> `database-design` 明确写了「NoSQL（MongoDB/Redis）仅涉及选型建议，不深入设计」。
> 本篇补的就是那层设计：**怎么把缓存做对**——选错数据结构、Key 命名混乱、失效策略缺失，
> 是线上缓存事故的三大主因。

## 能力

- **数据结构选型**：String / Hash / Set / Sorted Set / List / Stream 各自的适用场景与反例
- **Key 命名规范**：分层命名、版本位、TTL 语义，避免"改一次结构就要清全库"
- **缓存模式**：Cache-Aside（懒加载）、Write-Through、Read-Through 的取舍
- **失效策略**：TTL 抖动、tag 集合失效、写后失效的顺序与竞态处理
- **防护**：缓存击穿/穿透/雪崩的识别与对症实现

## 选型决策树

```
Q1: 数据形态？
 ├─ 单值（序列化对象/计数器）        → String（计数器用 INCR，不要 GET+SET，非原子）
 ├─ 多字段且常按字段读写（用户资料）  → Hash（避免整对象 String 反复序列化）
 ├─ 去重集合 / 判断"在不在"          → Set（或大基数用 Sorted Set 计数近似）
 ├─ 排序 / TopN / 延迟队列           → Sorted Set（score = 时间戳或分值）
 ├─ 简单 FIFO 任务队列               → List（优先 Stream，List 无消费者组）
 └─ 事件流 / 多消费者 / 需 ACK       → Stream（消费者组 + pending 重投）

Q2: 会不会有并发写同一 key？
 ├─ 会 → 必须原子（Lua / WATCH 事务 / INCR / HINCRBY），禁止 read-modify-write
 └─ 不会 → 仍需设 TTL，无 TTL 的 key 是内存泄漏

Q3: 缓存与库的一致性要求？
 ├─ 容忍短暂不一致（读多写少展示数据）→ Cache-Aside + 失效（不是更新）
 ├─ 要求写即一致                     → Write-Through 或直接不缓存
 └─ 要求"绝不做读源"（钱、库存）     → 不缓存，或只用 Redis 做原子扣减
```

## Key 命名与 TTL（可直接抄的约定）

```
{业务}:{实体}:{维度}:{版本}     例  shop:product:view:10086:v2
```

- **末段带版本号**：结构变更时改版本号即可让旧缓存自然过期，避免全库 SCAN 清理
- **TTL 加抖动**：`ttl = base + rand(0, base*0.2)`，防止同批 key 同时过期打爆 DB（雪崩）
- **禁止无 TTL 的缓存 key**；持久数据要显式说明它是"存储"不是"缓存"

## 核心模式

### Cache-Aside（默认首选）

```
读：GET → 命中返回 → 未命中查 DB → SET(带抖动 TTL) → 返回
写：写 DB → **删除缓存**（不要"更新缓存"）
```

为什么删不写：更新缓存要重复一遍序列化逻辑，且并发写下"先查后写"会覆盖新值。

### Tag 失效（一个实体变更要连带失效多个视图 key）

```
SADD  tag:product:10086  shop:product:view:10086:v2
SADD  tag:product:10086  shop:list:cat:9:page:1:v2
# 失效：SMEMBERS tag → DEL 每个成员 → DEL tag
```

> 不要用 `KEYS pattern` 做失效——O(N) 阻塞主线程，生产环境用 tag 集合或 SCAN(count) 分批。

### 三类事故的对症

| 事故 | 现象 | 对策 |
|---|---|---|
| 击穿 | 单个热 key 过期瞬间大量请求打 DB | `SET NX` 互斥重建 / 逻辑过期后台刷新 |
| 穿透 | 查不存在的 key，次次回源 | 空值短 TTL 缓存 / 布隆过滤器前置 |
| 雪崩 | 大批 key 同时过期 | TTL 抖动 / 多级缓存 / 熔断降级 |

## 常见坑

| 坑 | 后果 | 修正 |
|---|---|---|
| `KEYS *` 做批量操作 | 主线程阻塞，线上抖动 | `SCAN` 游标 + count，或 tag 索引 |
| read-modify-write 非原子 | 并发丢更新 | Lua 或 `HINCRBY`/`INCR` |
| 大 value（>1MB）当缓存 | 网络与内存放大，慢查询 | 拆分/压缩，或改存对象存储只缓存索引 |
| 只加缓存不加失效 | 数据长期陈旧，用户投诉 | 每个缓存 key 必须有失效路径 |
| 把 Redis 当唯一真相存储 | 丢数据 | 明确 `maxmemory-policy` 与持久化定位 |
| 无 Redis 观测 | 命中率/逐出不可见 | 接 `../database-observability/` 看 hit ratio、evicted_keys |

## 边界与协作

- **限流 / 幂等 / 分布式锁**（同样基于 Redis 原子操作，但是另一类问题）→
  [`distributed-rate-limit`](../distributed-rate-limit/SKILL.md)
- 应用层中间件如何"挂上"限流/CORS → `ai-fullstack-engineer/backend-patterns`（本技能只讲数据层实现）
- MySQL/PostgreSQL 侧的设计 → [`postgres-patterns`](../postgres-patterns/SKILL.md)、[`mysql`](../mysql/SKILL.md)
- 观测指标与慢查询 → [`database-observability`](../database-observability/SKILL.md)
