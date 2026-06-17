---
title: Redis 使用指南：企业实战与高频问题解决方案
date: 2026-06-17 16:00:00
updated: 2026-06-17 16:00:00
categories:
  - 数据库
tags:
  - Redis
  - 缓存
  - 数据库
  - 面试
  - 分布式
  - 高并发
---

## 一、Redis 核心数据结构

### 1.1 五种基础类型

| 类型 | 底层编码 | 最大存储 | 典型场景 | 时间复杂度 |
|:----|:--------|:--------|:--------|:----------|
| STRING | int / embstr / raw | 512 MB | 缓存、计数器、分布式锁 | O(1) |
| LIST | quicklist | 2^32-1 | 消息队列、时间线 | 头尾 O(1) |
| SET | intset / hashtable | 2^32-1 | 标签、去重、共同好友 | O(1) |
| ZSET | ziplist / skiplist | 2^32-1 | 排行榜、延迟队列 | O(log N) |
| HASH | ziplist / hashtable | 2^32-1 | 对象缓存、用户信息 | O(1) |

### 1.2 高级类型

| 类型 | 说明 | 版本 | 场景 |
|:----|:----|:----|:-----|
| Bitmap | 位图 | 2.2+ | 签到统计、布隆过滤 |
| HyperLogLog | 概率基数统计 | 2.8+ | UV 统计 |
| GEO | 地理空间索引 | 3.2+ | 附近的人 |
| Stream | 消息流 | 5.0+ | 消息队列 |

### 1.3 面试题：ZSET 为什么用跳表而非 B+ Tree？

1. 实现简单：B+ Tree 涉及页分裂/合并/平衡
2. 内存占用：跳表指针密度低于 B+ Tree 页结构
3. 范围查询：双向链表同样高效
4. Redis 单线程：不需要 B+ Tree 的并发控制优势

```bash
ZADD leaderboard 100 "alice" 200 "bob" 150 "carol"
ZRANGE leaderboard 0 -1 WITHSCORES
ZRANK leaderboard "bob"
```

## 二、持久化

### 2.1 RDB

| 项目 | 说明 |
|:----|:----|
| 触发方式 | SAVE（同步）、BGSAVE（异步）、自动配置 |
| 文件 | dump.rdb，二进制紧凑格式 |
| 优点 | 文件小、加载快 |
| 缺点 | 可能丢数据 |

```bash
save 900 1       # 900 秒内 1 次修改
save 300 10      # 300 秒内 10 次修改
save 60 10000    # 60 秒内 10000 次修改
```

### 2.2 AOF

| 项目 | 说明 |
|:----|:----|
| 写策略 | always / everysec / no |
| 文件 | appendonly.aof |
| 优点 | 最多丢 1 秒数据 |
| 缺点 | 文件大、加载慢 |

```bash
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

### 2.3 混合持久化（4.0+）

```bash
aof-use-rdb-preamble yes
```

### 2.4 面试题：RDB vs AOF 选型

| 场景 | 推荐 | 原因 |
|:----|:----|:----|
| 缓存为主 | RDB | 性能好，文件小 |
| 数据不能丢 | AOF everysec | 最多丢 1 秒 |
| 生产环境 | 混合持久化 | RDB 加载快 + AOF 安全 |
| 纯缓存 | 关闭持久化 | 纯内存性能最优 |

## 三、缓存三大问题

### 3.1 穿透

问题：查询不存在的数据，每次穿透到数据库

方案 1：缓存空值

```python
def get_user(user_id):
    data = redis.get(f"user:{user_id}")
    if data is not None:
        return None if data == "NULL" else deserialize(data)
    data = db.query(user_id)
    if data is None:
        redis.setex(f"user:{user_id}", 60, "NULL")
    else:
        redis.setex(f"user:{user_id}", 3600, serialize(data))
    return data
```

方案 2：布隆过滤器

```python
class BloomFilter:
    def __init__(self, redis, key, capacity=1000000, error_rate=0.01):
        self.redis = redis
        self.key = key
        import math
        self.bit_size = int(-capacity * math.log(error_rate) / (math.log(2) ** 2))
        self.hash_count = int(self.bit_size * math.log(2) / capacity)

    def add(self, item):
        import mmh3
        for seed in range(self.hash_count):
            h = mmh3.hash(item, seed) % self.bit_size
            self.redis.setbit(self.key, h, 1)

    def might_contain(self, item):
        import mmh3
        for seed in range(self.hash_count):
            h = mmh3.hash(item, seed) % self.bit_size
            if not self.redis.getbit(self.key, h):
                return False
        return True
```

### 3.2 击穿

问题：热点 key 过期瞬间，大量请求穿透到 DB

方案 1：互斥锁

```python
def get_hot_data(key):
    data = redis.get(key)
    if data is not None:
        return data
    lock_key = f"lock:{key}"
    if redis.setnx(lock_key, "1", ex=10):
        try:
            data = db.query_expensive(key)
            redis.setex(key, 3600, data)
            return data
        finally:
            redis.delete(lock_key)
    else:
        time.sleep(0.05)
        return get_hot_data(key)
```

方案 2：逻辑过期

```python
HOT_KEY_TTL = 3600
REBUILD_TIME = 600

def get_hot_data_logic(key):
    data = redis.get(key)
    if data is None:
        return None
    value, expire_time = parse_data(data)
    if time.time() > expire_time - REBUILD_TIME:
        lock_key = f"rebuild_lock:{key}"
        if redis.setnx(lock_key, "1", ex=60):
            threading.Thread(target=rebuild_hot_data, args=(key,)).start()
    return value

def rebuild_hot_data(key):
    new_data = db.query_expensive(key)
    redis.setex(key, HOT_KEY_TTL * 2, serialize(new_data, time.time() + HOT_KEY_TTL))
    redis.delete(f"rebuild_lock:{key}")
```

### 3.3 雪崩

问题：大量 key 同时过期或 Redis 宕机

```python
import random

def set_cache(key, value, base_ttl=3600):
    jitter = random.randint(60, 600)
    redis.setex(key, base_ttl + jitter, value)

# 多级缓存：本地缓存 + Redis + DB
import functools

@functools.lru_cache(maxsize=10000)
def local_get(key):
    return None

def get_data(key):
    data = local_get(key)
    if data:
        return data
    data = redis.get(key)
    if data:
        local_set(key, data)
        return data
    data = db.query(key)
    redis.setex(key, random_ttl(), data)
    local_set(key, data)
    return data
```

### 3.4 对比

| 问题 | 原因 | 解决 |
|:----|:----|:----|
| 穿透 | 查不存在的数据 | 空值缓存、布隆过滤器 |
| 击穿 | 热点 key 过期 | 互斥锁、逻辑过期 |
| 雪崩 | 大量 key 同时过期 / 宕机 | 随机 TTL、多级缓存、高可用 |

## 四、分布式锁

### 4.1 SETNX + Lua

```python
import uuid

class RedisLock:
    def __init__(self, redis, lock_key, ttl=10):
        self.redis = redis
        self.lock_key = f"lock:{lock_key}"
        self.ttl = ttl
        self.value = str(uuid.uuid4())
        self.acquired = False

    def acquire(self, blocking=True, timeout=5):
        deadline = time.time() + timeout
        while time.time() < deadline:
            ok = self.redis.set(self.lock_key, self.value, nx=True, ex=self.ttl)
            if ok:
                self.acquired = True
                return True
            if not blocking:
                return False
            time.sleep(0.01)
        return False

    def release(self):
        if not self.acquired:
            return False
        lua = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        self.redis.eval(lua, 1, self.lock_key, self.value)
        self.acquired = False

    def __enter__(self):
        self.acquire()
        return self

    def __exit__(self, *args):
        self.release()

with RedisLock(redis, "order:123") as lock:
    if lock.acquired:
        process_order(123)
```

### 4.2 Redlock

```python
class Redlock:
    def __init__(self, nodes, lock_key, ttl=10):
        self.nodes = nodes
        self.lock_key = lock_key
        self.ttl = ttl
        self.value = str(uuid.uuid4())
        self.quorum = len(nodes) // 2 + 1

    def acquire(self):
        acquired = 0
        start = time.time()
        for node in self.nodes:
            try:
                if node.set(self.lock_key, self.value, nx=True, ex=self.ttl):
                    acquired += 1
            except Exception:
                pass
        return acquired >= self.quorum and (time.time() - start) < self.ttl
```

### 4.3 Redis vs ZooKeeper

| 对比 | Redis | ZooKeeper |
|:----|:-----|:---------|
| 原理 | SETNX + TTL | 临时顺序节点 + Watch |
| 一致性 | AP（最终一致性） | CP（强一致性） |
| 性能 | 高（纯内存） | 中 |
| 适用 | 高并发，可接受短暂失效 | 金融、配置中心 |

## 五、限流

### 5.1 固定窗口

```python
def fixed_window_limit(user_id):
    key = f"ratelimit:{user_id}:{int(time.time())}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 1)
    return count <= 100
```

### 5.2 滑动窗口（ZSET）

```python
def sliding_window_limit(user_id, max_reqs=100, window=1):
    key = f"ratelimit:{user_id}"
    now = time.time()
    redis.zremrangebyscore(key, 0, now - window)
    redis.zadd(key, {str(now): now})
    redis.expire(key, window + 1)
    return redis.zcard(key) <= max_reqs
```

### 5.3 令牌桶（Lua）

```lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])
local bucket = redis.call("HMGET", key, "tokens", "last_time")
local tokens = tonumber(bucket[1]) or capacity
local last_time = tonumber(bucket[2]) or now
local elapsed = math.max(now - last_time, 0)
local new_tokens = math.min(capacity, tokens + elapsed * rate)
if new_tokens >= requested then
    redis.call("HMSET", key, "tokens", new_tokens - requested, "last_time", now)
    redis.call("EXPIRE", key, 10)
    return 1
else
    return 0
end
```

```python
def token_bucket_limit(user_id, capacity=100, rate=10):
    return redis.eval(token_bucket_lua, 1, f"tb:{user_id}", capacity, rate, time.time(), 1)
```

### 5.4 限流算法对比

| 算法 | 特点 | 问题 | 适用 |
|:----|:----|:----|:----|
| 固定窗口 | 简单 | 窗口边界流量突刺 | 简单限流 |
| 滑动窗口 | 平滑 | ZSET 内存略高 | API 限流 |
| 令牌桶 | 支持突发 | 需 Lua 脚本 | 微服务网关 |
| 漏桶 | 恒定速率 | 不允许突发 | 流量整形 |

## 六、消息队列

### 6.1 Stream

```python
def produce(stream_key, data):
    return redis.xadd(stream_key, data, maxlen=10000)

def consume(stream_key, group_name, consumer_name):
    while True:
        msgs = redis.xreadgroup(group_name, consumer_name,
            {stream_key: ">"}, count=10, block=2000)
        if not msgs:
            continue
        for stream, messages in msgs:
            for msg_id, data in messages:
                try:
                    process_message(data)
                    redis.xack(stream_key, group_name, msg_id)
                except Exception as e:
                    redis.xadd(f"dlq:{stream_key}", {
                        "original_id": msg_id, "error": str(e)
                    })
```

### 6.2 延迟队列（ZSET）

```python
def enqueue_delay(queue_key, data, delay_seconds):
    redis.zadd(f"delay:{queue_key}", {json.dumps(data): time.time() + delay_seconds})

def process_delay_queue(queue_key):
    key = f"delay:{queue_key}"
    while True:
        now = time.time()
        tasks = redis.zrangebyscore(key, 0, now, start=0, num=100)
        if not tasks:
            time.sleep(1)
            continue
        for task in tasks:
            lua = "if redis.call('ZREM', KEYS[1], ARGV[1]) == 1 then return 1 end return 0"
            if redis.eval(lua, 1, key, task):
                process_task(json.loads(task))
```

### 6.3 Stream vs Kafka vs RabbitMQ

| 对比 | Redis Stream | Kafka | RabbitMQ |
|:----|:-----------|:------|:---------|
| 吞吐量 | 10万+/s | 百万+/s | 万+/s |
| 持久化 | RDB/AOF | 磁盘顺序写 | 磁盘 |
| 消息堆积 | 不擅长 | 天生支持 | 支持 |
| 适用 | 轻量队列 | 大数据、日志 | 可靠投递 |

## 七、实战场景

### 7.1 秒杀系统

```python
def seckill(user_id, product_id):
    stock_key = f"stock:{product_id}"
    user_key = f"seckill_user:{product_id}"
    if redis.sismember(user_key, user_id):
        return "已抢购过"
    lua = """
    local stock = redis.call("GET", KEYS[1])
    if not stock or tonumber(stock) <= 0 then return 0 end
    redis.call("DECR", KEYS[1])
    redis.call("SADD", KEYS[2], ARGV[1])
    return 1
    """
    if redis.eval(lua, 2, stock_key, user_key, user_id) == 0:
        return "已售罄"
    send_to_order_queue(product_id, user_id)
    return "抢购成功"
```

### 7.2 UV 统计（HyperLogLog）

```python
def record_uv(page_id, user_id):
    redis.pfadd(f"uv:{page_id}:{datetime.date.today()}", user_id)

def get_uv(page_id):
    return redis.pfcount(f"uv:{page_id}:{datetime.date.today()}")

def get_monthly_uv(page_id, year, month):
    keys = [f"uv:{page_id}:{year}-{month:02d}-{day:02d}" for day in range(1, 32)]
    return redis.pfcount(*keys)
```

### 7.3 附近的人（GEO）

```python
def set_location(user_id, longitude, latitude):
    redis.geoadd("user_locations", longitude, latitude, user_id)

def find_nearby(user_id, radius_km=5):
    result = redis.georadiusbymember("user_locations", user_id, radius_km,
        unit="km", withdist=True, withcoord=True, count=50, sort="ASC")
    return [{"user_id": uid.decode(), "distance_km": round(dist, 2),
             "longitude": coord[0], "latitude": coord[1]}
            for uid, dist, coord in result]
```

### 7.4 排行榜（ZSET）

```python
def update_score(player_id, delta):
    redis.zincrby("leaderboard", delta, player_id)

def get_top_n(n=10):
    return redis.zrevrange("leaderboard", 0, n-1, withscores=True)

def get_rank(player_id):
    rank = redis.zrevrank("leaderboard", player_id)
    return rank + 1 if rank is not None else None
```

### 7.5 分布式 Session

```python
from flask import Flask, session
from flask_session import RedisSessionInterface

app = Flask(__name__)
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='redis.example.com', port=6379)
app.config['PERMANENT_SESSION_LIFETIME'] = 86400
app.session_interface = RedisSessionInterface(
    redis=app.config['SESSION_REDIS'], key_prefix='session:',
    use_signer=True, permanent=True)

@app.route('/login')
def login():
    session['user_id'] = 123
    return 'OK'

@app.route('/profile')
def profile():
    user_id = session.get('user_id')
    return f'User: {user_id}' if user_id else ('未登录', 401)
```

### 7.6 接口幂等性

```python
def idempotent_api(request):
    key = request.headers.get("Idempotent-Key")
    if not key:
        return error("缺少幂等键")
    if not redis.set(f"idempotent:{key}", "processed", nx=True, ex=86400):
        return redis.get(f"idempotent_result:{key}")
    result = process_payment(request)
    redis.setex(f"idempotent_result:{key}", 86400, result)
    return result
```

## 八、生产运维

### 8.1 Big Key

| 危害 | 解决 |
|:----|:----|
| 阻塞网络 | 拆分为多个小 key |
| DEL 阻塞 | UNLINK key（4.0+ 异步删除） |
| 内存不均 | 拆分或本地缓存 |

### 8.2 Hot Key

| 解决 | 说明 |
|:----|:----|
| 本地缓存 | 客户端缓存 |
| 读写分离 | 从节点分担读 |
| 散列 | key 加后缀分散 |

### 8.3 内存淘汰

```bash
maxmemory 4gb
maxmemory-policy allkeys-lru
# noeviction / allkeys-lru / volatile-lru / allkeys-lfu / volatile-ttl
```

### 8.4 监控命令

```bash
redis-cli INFO stats
redis-cli INFO memory
redis-cli SLOWLOG GET 10
redis-cli --stat
```

## 九、高频面试题

### 9.1 Redis 为什么快？

1. 纯内存操作
2. 单线程避免上下文切换和锁竞争
3. I/O 多路复用（epoll/kqueue）
4. 数据结构高效

### 9.2 过期策略

- 惰性删除：访问时检查
- 定期删除：每秒 10 次，随机抽 20 个 key
- 内存淘汰：maxmemory-policy 触发

### 9.3 主从复制原理

Slave 发起 PSYNC → Master 生成 RDB 并缓存增量命令 → Slave 加载 RDB → 追增量

### 9.4 脑裂问题

主从切换时出现多个主节点写。解决：
- min-slaves-to-write
- min-slaves-max-lag
- 哨兵 majority 决策

### 9.5 数据一致性

```python
# Cache Aside 模式
def update_user(user):
    db.update(user)
    redis.delete(f"user:{user.id}")

def get_user(user_id):
    data = redis.get(f"user:{user_id}")
    if data:
        return deserialize(data)
    data = db.query(user_id)
    redis.setex(f"user:{user_id}", 3600, serialize(data))
    return data
```

## 十、总结

### 速查表

| 考点 | 要点 |
|:----|:----|
| 数据结构 | STRING/LIST/SET/ZSET/HASH + Bitmap/HyperLogLog/GEO/Stream |
| 持久化 | RDB、AOF、混合（4.0+） |
| 缓存问题 | 穿透（空值/布隆）、击穿（互斥锁/逻辑过期）、雪崩（随机TTL/多级缓存） |
| 分布式锁 | SETNX + Lua / Redlock |
| 限流 | 固定窗口、滑动窗口（ZSET）、令牌桶（Lua） |
| 消息队列 | Stream（5.0+）、延迟队列（ZSET） |
| 淘汰策略 | allkeys-lru（推荐） |

### 常用命令速记

```bash
SET key value [NX|XX] [EX seconds]
SETNX key value
GET key
DEL / UNLINK key
EXPIRE key seconds
INCR / DECR key
LPUSH / RPUSH / LPOP / RPOP
SADD / SISMEMBER / SMEMBERS
ZADD / ZRANGE / ZRANK / ZINCRBY
HSET / HGET / HGETALL / HINCRBY
GEOADD / GEORADIUSBYMEMBER
PFADD / PFCOUNT / PFMERGE
XADD / XREADGROUP / XACK
PUBLISH / SUBSCRIBE
```
