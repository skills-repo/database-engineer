---
name: distributed-rate-limit
description: 基于 Redis 的限流与幂等：固定/滑动窗口与令牌桶、Lua 原子实现、去重键与分布式锁
source:
  type: derived
  repo: upstash/skills
  path: skills/upstash-ratelimit-js/SKILL.md
  updated: 2026-11-02
  url: https://github.com/upstash/skills/blob/main/skills/upstash-ratelimit-js/SKILL.md
  also:
    - affaan-m/ecc@redis-patterns（7.6K 安装）https://github.com/affaan-m/ecc
    - redis/agent-skills@redis-core（官方，4.2K 安装）
metadata:
  category: 限流
  platform: Redis
  difficulty: 进阶
---

# 限流、幂等与分布式锁（数据层实现）

> 本技能回答的是**算法与原子性**：限流怎么算才不多放不漏放、幂等键怎么设计才不会重复扣款。
> 「在 Express/FastAPI 里怎么挂一个限流中间件」属于应用层，见
> `ai-fullstack-engineer/backend-patterns`——那侧只有选型建议，不含实现。

## 能力

- **算法选型**：固定窗口 / 滑动窗口日志 / 滑动窗口计数 / 令牌桶 的适用性与误差
- **原子实现**：用 Lua 或 `ZSET`/`INCR` 保证"检查与扣减"不可分割
- **维度设计**：按用户 / IP / API Key / 租户限流，以及多维叠加时的优先级
- **幂等**：Idempotency-Key 的存储、TTL、重试语义与响应回放
- **互斥**：分布式锁的正确边界（以及为什么大多数场景该换思路而不是加锁）

## 算法选型

| 算法 | Redis 实现 | 优点 | 代价 | 适用 |
|---|---|---|---|---|
| 固定窗口 | `INCR` + `EXPIRE` | 最省内存、实现最简 | 窗口边界**可放过 2 倍流量** | 粗粒度配额、成本防护 |
| 滑动窗口日志 | `ZSET`（score=时间戳） | 精确 | 内存随请求数线性涨 | 低 QPS 高精度（金融下单） |
| 滑动窗口计数 | 两个固定窗口加权 | 内存恒定、近似精确 | 有近似误差 | **默认选择**，API 网关 |
| 令牌桶 | Lua + `last_refill` | 允许突发、恢复平滑 | 需 Lua 保原子 | 客户端 SDK、后台任务 |

> 边界放大是固定窗口的固有问题：`limit=100/min` 在最坏相位下 1 秒内可放进 200 个请求。
> 若下游只扛得住 100/min，固定窗口就不够格。

## 原子性是硬要求

**错误写法**（典型竞态：两个请求同时读到 99，各自 +1 后放进 101）

```
count = GET key          # 读
if count < limit: INCR   # 判 + 写，分离
```

**正确写法**：把"读-判-改-设 TTL"封进一次 Lua（Redis 单线程顺序执行脚本，天然原子）

```lua
-- KEYS[1]=桶键  ARGV[1]=limit  ARGV[2]=窗口秒  ARGV[3]=now_ms  ARGV[4]=唯一id
local cur = redis.call('INCR', KEYS[1])
if cur == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
if cur > tonumber(ARGV[1]) then
  return {0, tonumber(ARGV[1]) - cur + tonumber(ARGV[2]) * 0}
end
return {1, tonumber(ARGV[1]) - cur}
```

要点：
- `INCR` 后**只在返回 1 时**设 EXPIRE，否则每请求都续期会让窗口永不过期
- 返回剩余配额，让调用方写 `X-RateLimit-Remaining`，而不是自己再读一次（又引入竞态）
- 用 `EVALSHA` + 脚本缓存，别每次传整段脚本文本

## 限流维度与"该限谁"

```
认证接口 → 按 (tenant, user, route)；
匿名接口 → 按 IP，但**必须处理共享出口 IP**（企业 NAT、移动网络、公司办公网），
           纯 IP 限流会误伤一整栋楼 → 结合 UA/指纹或放宽 IP 维度、收紧路由维度。
成本出口 → 按 API Key（上游供应商配额）与按租户双层，内层更紧。
```

**多维叠加时的原则**：取最紧的那个作为拒绝依据，并把命中的维度回写到响应头，
否则排障时没人知道是谁被哪条规则挡住。

## 幂等（比限流更容易出事故）

```
1. 客户端生成 Idempotency-Key（每次业务动作一个，不是每次 HTTP 请求一个）
2. 服务端 SETNX idem:{key} → 抢到才执行业务
3. 执行完写入响应摘要 + TTL（覆盖"重试要回放同一个结果"）
4. 抢锁失败但已有结果 → 直接回放已存响应；有结果正在生成中 → 返回 409/202 让客户端稍后
```

- TTL 要 **≥ 客户端最大重试跨度**，常见 24h
- 只存"成功响应"不够：失败也要能区分"这次是真失败"还是"重复请求"
- 幂等键必须**参与业务唯一约束**（DB 唯一索引兜底），Redis 丢了不至于重复扣款

## 分布式锁的诚实建议

| 需求 | 首选 |
|---|---|
| 保证不重复执行同一任务 | **唯一约束 / CAS / 队列 ACK**，而不是锁 |
| 必须互斥的临界区 | `SET key token NX PX ttl` + 释放时 Lua 比对 token |
| 锁持有时间不可预测 | 别用固定 TTL 锁（会提前失效），改租约 + 续约或换队列 |

> `Redlock` 在多时钟/主从切换场景下不保证安全。把"加锁"当默认解，通常是设计问题——
> 先问能不能用唯一键或幂等消费消掉这个临界区。

## 观测与失效模式

- 必埋指标：拒绝率（按维度）、脚本耗时、`Redis` 侧 `blocked_clients`
- **Redis 不可用时限流必须 fail-open 还是 fail-close？** 这是业务决策，要写在配置里而不是
  依赖默认行为；成本类接口应 fail-close，登录类通常 fail-open 并降级到本地粗限流
- 逐出策略会静默改变语义：`maxmemory-policy allkeys-lru` 下限流桶可能被逐出导致配额重置
  → 限流键空间应使用 `volatile-*` 或独立实例/DB

## 边界

- 缓存与数据结构选型 → [`redis-cache-patterns`](../redis-cache-patterns/SKILL.md)
- 应用层中间件挂载与配置 → `ai-fullstack-engineer/backend-patterns`
- API 网关与配额产品设计 → `ai-fullstack-engineer/api-design`
- 慢查询与实例观测 → [`database-observability`](../database-observability/SKILL.md)
