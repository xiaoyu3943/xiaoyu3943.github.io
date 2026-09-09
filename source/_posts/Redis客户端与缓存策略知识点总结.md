---
title: Redis 客户端与缓存策略
date: 2026-06-07
tags: [Redis, 缓存, 分布式锁, 幂等性, Python]
categories: 运维
---

## 一、Redis 客户端初始化

## 一、Redis 客户端初始化

```python
import redis.asyncio as aioredis
from app.config import get_settings

settings = get_settings()

async def get_redis():
    pool = aioredis.ConnectionPool.from_url(
        settings.redis_url,
        max_connections=20,
        decode_responses=True,
    )
    client = aioredis.Redis(connection_pool=pool)
    try:
        yield client
    finally:
        await pool.disconnect()
```

---

## 二、aioredis 的演进

```python
# 新写法（redis-py >= 4.2，内置异步支持）
import redis.asyncio as aioredis
redis = aioredis.Redis.from_url("redis://...")
```

| 阶段 | 包名 | 说明 |
|------|------|------|
| 现在 | `redis.asyncio`（redis-py 内置） | `pip install redis` 即可 |

---

## 三、连接池模式

### 3.1 为什么用连接池？

```
无连接池（每次请求新建 TCP 连接）:
请求1:  TCP三次握手 → Redis操作 → TCP四次挥手
请求2:  TCP三次握手 → Redis操作 → TCP四次挥手
请求3:  TCP三次握手 → Redis操作 → TCP四次挥手
→ 每次请求都花大量时间在建立/断开连接上

连接池（复用已有连接）:
初始化:  创建 20 个连接放入池中
请求1:  从池中取连接 → Redis操作 → 归还连接
请求2:  从池中取连接 → Redis操作 → 归还连接
请求3:  从池中取连接 → Redis操作 → 归还连接
→ 连接复用，省去 TCP 握手/挥手开销
```

### 3.2 连接池参数

```python
pool = aioredis.ConnectionPool.from_url(
    settings.redis_url,      # "redis://:password@host:6379/0"
    max_connections=20,      # 最大连接数
    decode_responses=True,   # 自动 decode
)
```

| 参数 | 含义 | 推荐值 |
|------|------|--------|
| `max_connections` | 连接池最大连接数 | 20（与 DB 连接池一致） |
| `decode_responses` | 自动将 Redis 返回的 `bytes` 解码为 `str` | **True**（强烈推荐） |

### 3.3 `decode_responses=True` 的意义

```python
# decode_responses=False（默认）
value = await redis.get("key")  # b"hello" ← bytes，需要手动 decode

# decode_responses=True
value = await redis.get("key")  # "hello"  ← str，开箱即用
```

---

## 四、依赖注入与生命周期

```python
async def get_redis():
    pool = aioredis.ConnectionPool.from_url(...)  # ① 创建连接池
    client = aioredis.Redis(connection_pool=pool)  # ② 创建客户端
    try:
        yield client           # ③ 交给业务代码使用
    finally:
        await pool.disconnect() # ④ 请求结束，断开连接池
```

### 4.1 生命周期图解

```
请求到达
  │
  ├── ① ConnectionPool.from_url() — 建立 20 个 TCP 连接
  ├── ② Redis(connection_pool=pool) — 封装客户端
  ├── ③ yield client — 业务代码使用（get/set/hset...）
  ├── ④ pool.disconnect() — 释放所有连接
  │
请求结束
```

---

## 五、本项目 Redis 的三大用途

### 5.1 幂等性缓存

```python
# 存储: 请求处理完后的结果
await redis.set(
    f"idempotency:{idempotency_key}",
    json.dumps(result),
    ex=86400  # 24小时过期
)

# 查询: 新请求先查缓存
cached = await redis.get(f"idempotency:{idempotency_key}")
if cached:
    return json.loads(cached)  # 直接返回，不重复处理
```

| 参数 | 值 | 说明 |
|------|-----|------|
| Key 格式 | `idempotency:{key}` | 统一前缀，便于清理 |
| TTL | 86400 秒（24小时） | 与 `IDEMPOTENCY_KEY_TTL` 配置一致 |

### 5.2 账户余额缓存

```python
# 读取余额时缓存
await redis.set(
    f"account:balance:{account_id}",
    str(balance),
    ex=300  # 5分钟过期
)

# 余额变更时主动失效(避免后发生的数据库操作，先更新完缓存；先发生的数据库操作，慢了一步，最后用旧值覆盖缓存！)
await redis.delete(f"account:balance:{account_id}")
```

| 策略 | 说明 |
|------|------|
| **Cache-Aside** | 先查缓存 → 缓存未命中 → 查 DB → 回写缓存 |
| **主动失效** | 余额变更时 `DELETE` 缓存，而非更新（避免并发写不一致） |

### 5.3 分布式锁

```python
# 获取锁
lock_acquired = await redis.set(
    f"lock:account:{account_id}",
    "1",
    nx=True,   # 仅当 key 不存在时设置
    ex=10,     # 10秒自动过期（防止死锁）
)

if not lock_acquired:
    raise RuntimeError("账户操作过于频繁，请稍后重试")

try:
    # 执行需要互斥的操作
    await transfer(...)
finally:
    # 释放锁（用 Lua 脚本保证原子性）
    await redis.eval(
        "if redis.call('get', KEYS[1]) == ARGV[1] then "
        "return redis.call('del', KEYS[1]) end return 0",
        1, f"lock:account:{account_id}", "1"
    )
```

| 要点 | 说明 |
|------|------|
| `nx=True` | 原子操作，确保只有一个客户端能抢到锁 |
| `ex=10` | 自动过期，防止进程崩溃导致死锁 |
| Lua 脚本释放 | 先检查再删除，确保只释放自己持有的锁 |

---

## 六、Redis 命令速查（Python 异步版）

### 6.1 字符串操作

```python
await redis.set("key", "value")
await redis.set("key", "value", ex=3600)   # 带过期时间（秒）
await redis.get("key")                     # 返回 str 或 None
await redis.delete("key")
await redis.exists("key")                  # 返回 bool
await redis.expire("key", 3600)            # 设置/重置过期时间
```

### 6.2 原子操作

```python
await redis.set("key", "value", nx=True)        # 仅 key 不存在时设置
await redis.set("key", "value", xx=True)        # 仅 key 存在时设置
await redis.incr("counter")                     # 自增 1
await redis.incrby("counter", 10)               # 自增 N
```

### 6.3 哈希操作

```python
await redis.hset("user:1", mapping={"name": "alice", "age": "25"})
await redis.hget("user:1", "name")              # "alice"
await redis.hgetall("user:1")                   # {"name": "alice", "age": "25"}
await redis.hdel("user:1", "age")
```

### 6.4 列表操作

```python
await redis.lpush("queue", "task1", "task2")    # 左侧推入
await redis.rpop("queue")                       # 右侧弹出
await redis.lrange("queue", 0, -1)              # 获取全部元素
```

---

## 七、常见问题

### Q1: 连接池大小设多少合适？

与数据库连接池类似：`pool_size ≈ 2 * CPU核心数`。Redis 本身单线程，连接过多也不会并行处理，20~50 通常足够。

### Q2: `decode_responses=True` 有什么副作用？

Redis 只能存 bytes，设置后自动解码为 str。如果存的是二进制数据（如 pickle），需要设为 `False`。本系统存的是 JSON 字符串，用 `True` 更方便。

### Q3: 分布式锁的 10 秒过期够吗？

锁的过期时间应**大于**业务操作的最长执行时间。转账操作通常 100ms 内完成，10 秒已经非常安全。如果操作可能超时，需要实现"看门狗"自动续期。
