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

```go
func GetUser(rdb *redis.Client, userID int) (*User, error) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)

    data, err := rdb.Get(ctx, key).Result()
    if err == nil {
        if data == "NULL" {
            return nil, nil
        }
        var u User
        json.Unmarshal([]byte(data), &u)
        return &u, nil
    }

    u, err := db.QueryUser(userID)
    if err != nil {
        return nil, err
    }
    if u == nil {
        rdb.SetEx(ctx, key, "NULL", 60*time.Second)
        return nil, nil
    }
    b, _ := json.Marshal(u)
    rdb.SetEx(ctx, key, string(b), 3600*time.Second)
    return u, nil
}
```

```cpp
#include <sw/redis++/redis++.h>
using namespace sw::redis;

std::optional<User> GetUser(Redis& rdb, int user_id) {
    auto key = "user:" + std::to_string(user_id);
    auto data = rdb.get(key);
    if (data) {
        if (*data == "NULL") return std::nullopt;
        return User::FromJson(*data);
    }
    auto u = Db::QueryUser(user_id);
    if (!u.has_value()) {
        rdb.setex(key, 60, "NULL");
        return std::nullopt;
    }
    rdb.setex(key, 3600, u->ToJson());
    return u;
}
```

方案 2：布隆过滤器

```go
type BloomFilter struct {
    rdb       *redis.Client
    key       string
    bitSize   uint64
    hashCount uint
}

func NewBloomFilter(rdb *redis.Client, key string, capacity uint64, errRate float64) *BloomFilter {
    bitSize := uint64(-float64(capacity) * math.Log(errRate) / (math.Ln2 * math.Ln2))
    hashCount := uint(float64(bitSize) * math.Ln2 / float64(capacity))
    return &BloomFilter{rdb, key, bitSize, hashCount}
}

func (bf *BloomFilter) Add(item string) {
    for seed := uint(0); seed < bf.hashCount; seed++ {
        h := xxhash.Seed(seed).Sum64(item) % bf.bitSize
        bf.rdb.SetBit(context.Background(), bf.key, int64(h), 1)
    }
}

func (bf *BloomFilter) MightContain(item string) bool {
    for seed := uint(0); seed < bf.hashCount; seed++ {
        h := xxhash.Seed(seed).Sum64(item) % bf.bitSize
        v, _ := bf.rdb.GetBit(context.Background(), bf.key, int64(h)).Result()
        if v == 0 {
            return false
        }
    }
    return true
}
```

```cpp
class BloomFilter {
    Redis& rdb_;
    std::string key_;
    uint64_t bit_size_;
    uint32_t hash_count_;
public:
    BloomFilter(Redis& rdb, std::string key, uint64_t capacity, double err_rate)
        : rdb_(rdb), key_(std::move(key)) {
        bit_size_ = static_cast<uint64_t>(-static_cast<double>(capacity)
            * std::log(err_rate) / (std::log(2) * std::log(2)));
        hash_count_ = static_cast<uint32_t>(
            static_cast<double>(bit_size_) * std::log(2) / capacity);
    }
    void Add(const std::string& item) {
        for (uint32_t seed = 0; seed < hash_count_; ++seed) {
            auto h = XXHash::Sum64(item, seed) % bit_size_;
            rdb_.setbit(key_, h, 1);
        }
    }
    bool MightContain(const std::string& item) {
        for (uint32_t seed = 0; seed < hash_count_; ++seed) {
            auto h = XXHash::Sum64(item, seed) % bit_size_;
            if (rdb_.getbit(key_, h) == 0) return false;
        }
        return true;
    }
};
```

### 3.2 击穿

问题：热点 key 过期瞬间，大量请求穿透到 DB

方案 1：互斥锁

```go
func GetHotData(rdb *redis.Client, key string) (string, error) {
    ctx := context.Background()

    data, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return data, nil
    }

    lockKey := "lock:" + key
    ok, err := rdb.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
    if err != nil {
        return "", err
    }
    if ok {
        defer rdb.Del(ctx, lockKey)
        data, err := db.QueryExpensive(key)
        if err != nil {
            return "", err
        }
        rdb.SetEx(ctx, key, data, 3600*time.Second)
        return data, nil
    }

    time.Sleep(50 * time.Millisecond)
    return GetHotData(rdb, key)
}
```

```cpp
std::string GetHotData(Redis& rdb, const std::string& key) {
    auto data = rdb.get(key);
    if (data) return *data;

    auto lock_key = "lock:" + key;
    auto ok = rdb.setnx(lock_key, "1");
    if (ok) {
        auto expire = std::chrono::seconds(10);
        rdb.expire(lock_key, expire);
        try {
            auto val = Db::QueryExpensive(key);
            rdb.setex(key, 3600, val);
            rdb.del(lock_key);
            return val;
        } catch (...) {
            rdb.del(lock_key);
            throw;
        }
    }
    std::this_thread::sleep_for(50ms);
    return GetHotData(rdb, key);
}
```

方案 2：逻辑过期

```go
func GetHotDataLogic(rdb *redis.Client, key string) (string, error) {
    data, err := rdb.Get(context.Background(), key).Result()
    if err != nil {
        return "", nil
    }
    val, expireAt := parseData(data)
    if time.Now().Unix() > expireAt-600 {
        lockKey := "rebuild:" + key
        ok, _ := rdb.SetNX(context.Background(), lockKey, "1", 60*time.Second).Result()
        if ok {
            go rebuildHotData(rdb, key)
        }
    }
    return val, nil
}

func rebuildHotData(rdb *redis.Client, key string) {
    newData := db.QueryExpensive(key)
    serialized := serializeWithExpire(newData, time.Now().Add(3600*time.Second).Unix())
    rdb.SetEx(context.Background(), key, serialized, 7200*time.Second)
    rdb.Del(context.Background(), "rebuild:"+key)
}
```

```cpp
std::string GetHotDataLogic(Redis& rdb, const std::string& key) {
    auto data = rdb.get(key);
    if (!data) return "";

    auto [val, expire_at] = ParseData(*data);
    if (time(nullptr) > expire_at - 600) {
        auto lock_key = "rebuild:" + key;
        if (rdb.setnx(lock_key, "1")) {
            rdb.expire(lock_key, std::chrono::seconds(60));
            std::thread([&rdb, key] {
                auto fresh = Db::QueryExpensive(key);
                auto serialized = SerializeWithExpire(fresh, time(nullptr) + 3600);
                rdb.setex(key, 7200, serialized);
                rdb.del("rebuild:" + key);
            }).detach();
        }
    }
    return val;
}
```

### 3.3 雪崩

问题：大量 key 同时过期或 Redis 宕机

```go
func SetCache(rdb *redis.Client, key, value string, baseTTL time.Duration) {
    jitter := time.Duration(rand.Intn(540)+60) * time.Second
    rdb.SetEx(context.Background(), key, value, baseTTL+jitter)
}

// 多级缓存
var localCache = make(map[string]cacheEntry)
var localMu sync.RWMutex

func GetData(rdb *redis.Client, key string) (string, error) {
    localMu.RLock()
    entry, ok := localCache[key]
    localMu.RUnlock()
    if ok && time.Now().Before(entry.expireAt) {
        return entry.value, nil
    }

    data, err := rdb.Get(context.Background(), key).Result()
    if err == nil {
        localMu.Lock()
        localCache[key] = cacheEntry{value: data, expireAt: time.Now().Add(1 * time.Second)}
        localMu.Unlock()
        return data, nil
    }

    data, err = db.Query(key)
    if err != nil {
        return "", err
    }
    rdb.SetEx(context.Background(), key, data, randomTTL())
    localMu.Lock()
    localCache[key] = cacheEntry{value: data, expireAt: time.Now().Add(1 * time.Second)}
    localMu.Unlock()
    return data, nil
}
```

```cpp
void SetCache(Redis& rdb, const std::string& key,
              const std::string& value, int base_ttl) {
    static std::mt19937 rng(std::random_device{}());
    std::uniform_int_distribution<> dist(60, 600);
    rdb.setex(key, base_ttl + dist(rng), value);
}

std::string GetData(Redis& rdb, const std::string& key) {
    // L1: local cache (omitted for brevity, use mutex + unordered_map)
    // L2: redis
    auto data = rdb.get(key);
    if (data) return *data;
    // L3: db
    auto val = Db::Query(key);
    // random TTL
    static std::mt19937 rng(std::random_device{}());
    std::uniform_int_distribution<> dist(300, 3600);
    rdb.setex(key, dist(rng), val);
    return val;
}
```

### 3.4 对比

| 问题 | 原因 | 解决 |
|:----|:----|:----|
| 穿透 | 查不存在的数据 | 空值缓存、布隆过滤器 |
| 击穿 | 热点 key 过期 | 互斥锁、逻辑过期 |
| 雪崩 | 大量 key 同时过期 / 宕机 | 随机 TTL、多级缓存、高可用 |

## 四、分布式锁

### 4.1 SETNX + Lua

```go
type RedisLock struct {
    rdb       *redis.Client
    lockKey   string
    ttl       time.Duration
    value     string
    acquired  bool
}

func NewRedisLock(rdb *redis.Client, key string, ttl time.Duration) *RedisLock {
    return &RedisLock{
        rdb:     rdb,
        lockKey: "lock:" + key,
        ttl:     ttl,
        value:   uuid.New().String(),
    }
}

func (l *RedisLock) Acquire(ctx context.Context, timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        ok, err := l.rdb.SetNX(ctx, l.lockKey, l.value, l.ttl).Result()
        if err == nil && ok {
            l.acquired = true
            return true
        }
        time.Sleep(10 * time.Millisecond)
    }
    return false
}

func (l *RedisLock) Release(ctx context.Context) {
    if !l.acquired {
        return
    }
    script := `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    end
    return 0
    `
    l.rdb.Eval(ctx, script, []string{l.lockKey}, l.value)
    l.acquired = false
}
```

```cpp
class RedisLock {
    Redis& rdb_;
    std::string lock_key_;
    std::chrono::seconds ttl_;
    std::string value_;
    bool acquired_ = false;
public:
    RedisLock(Redis& rdb, const std::string& key, std::chrono::seconds ttl)
        : rdb_(rdb), lock_key_("lock:" + key), ttl_(ttl),
          value_(UUID::Generate()) {}

    bool Acquire(std::chrono::milliseconds timeout) {
        auto deadline = std::chrono::steady_clock::now() + timeout;
        while (std::chrono::steady_clock::now() < deadline) {
            if (rdb_.set(lock_key_, value_,
                         std::chrono::seconds(ttl_),
                         UpdateType::NOT_EXIST)) {
                acquired_ = true;
                return true;
            }
            std::this_thread::sleep_for(10ms);
        }
        return false;
    }

    void Release() {
        if (!acquired_) return;
        auto script = R"(
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("DEL", KEYS[1])
            end
            return 0
        )";
        rdb_.eval(script, {lock_key_}, {value_});
        acquired_ = false;
    }

    ~RedisLock() { Release(); }
};
```

### 4.2 Redlock

```go
type Redlock struct {
    nodes   []*redis.Client
    lockKey string
    ttl     time.Duration
    value   string
    quorum  int
}

func NewRedlock(nodes []*redis.Client, key string, ttl time.Duration) *Redlock {
    return &Redlock{
        nodes:   nodes,
        lockKey: "lock:" + key,
        ttl:     ttl,
        value:   uuid.New().String(),
        quorum:  len(nodes)/2 + 1,
    }
}

func (r *Redlock) Acquire(ctx context.Context) bool {
    acquired := 0
    start := time.Now()
    for _, node := range r.nodes {
        ok, err := node.SetNX(ctx, r.lockKey, r.value, r.ttl).Result()
        if err == nil && ok {
            acquired++
        }
    }
    return acquired >= r.quorum && time.Since(start) < r.ttl
}
```

```cpp
class Redlock {
    std::vector<Redis*> nodes_;
    std::string lock_key_;
    std::chrono::seconds ttl_;
    std::string value_;
    int quorum_;
public:
    Redlock(std::vector<Redis*> nodes, const std::string& key,
            std::chrono::seconds ttl)
        : nodes_(std::move(nodes)), lock_key_("lock:" + key),
          ttl_(ttl), value_(UUID::Generate()),
          quorum_(nodes_.size() / 2 + 1) {}

    bool Acquire() {
        int acquired = 0;
        auto start = std::chrono::steady_clock::now();
        for (auto* node : nodes_) {
            try {
                if (node->set(lock_key_, value_, ttl_, UpdateType::NOT_EXIST)) {
                    ++acquired;
                }
            } catch (...) {}
        }
        auto elapsed = std::chrono::steady_clock::now() - start;
        return acquired >= quorum_ && elapsed < ttl_;
    }
};
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

```go
func FixedWindowLimit(rdb *redis.Client, userID string, limit int) bool {
    ctx := context.Background()
    key := fmt.Sprintf("ratelimit:%s:%d", userID, time.Now().Unix())
    count, _ := rdb.Incr(ctx, key).Result()
    if count == 1 {
        rdb.Expire(ctx, key, time.Second)
    }
    return count <= int64(limit)
}
```

```cpp
bool FixedWindowLimit(Redis& rdb, const std::string& user_id, int limit) {
    auto key = "ratelimit:" + user_id + ":" + std::to_string(time(nullptr));
    auto count = rdb.incr(key);
    if (count == 1) {
        rdb.expire(key, std::chrono::seconds(1));
    }
    return count <= limit;
}
```

### 5.2 滑动窗口（ZSET）

```go
func SlidingWindowLimit(rdb *redis.Client, userID string, limit int, window time.Duration) bool {
    ctx := context.Background()
    key := "ratelimit:" + userID
    now := float64(time.Now().Unix())
    windowSec := window.Seconds()

    rdb.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%.0f", now-windowSec))
    rdb.ZAdd(ctx, key, redis.Z{Score: now, Member: fmt.Sprintf("%.0f", now)})
    rdb.Expire(ctx, key, window+time.Second)

    count, _ := rdb.ZCard(ctx, key).Result()
    return count <= int64(limit)
}
```

```cpp
bool SlidingWindowLimit(Redis& rdb, const std::string& user_id, int limit, int window_sec) {
    auto key = "ratelimit:" + user_id;
    auto now = std::chrono::duration_cast<std::chrono::seconds>(
        std::chrono::system_clock::now().time_since_epoch()).count();

    rdb.zremrangebyscore(key, "-inf", "(" + std::to_string(now - window_sec));
    rdb.zadd(key, MemberScore{std::to_string(now), static_cast<double>(now)});
    rdb.expire(key, std::chrono::seconds(window_sec + 1));

    return rdb.zcard(key) <= static_cast<size_t>(limit);
}
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

```go
func TokenBucketLimit(rdb *redis.Client, userID string, capacity, rate int) bool {
    script := redis.NewScript(`
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local bucket = redis.call("HMGET", key, "tokens", "last_time")
        local tokens = tonumber(bucket[1]) or capacity
        local last_time = tonumber(bucket[2]) or now
        local elapsed = math.max(now - last_time, 0)
        local new_tokens = math.min(capacity, tokens + elapsed * rate)
        if new_tokens >= 1 then
            redis.call("HMSET", key, "tokens", new_tokens - 1, "last_time", now)
            redis.call("EXPIRE", key, 10)
            return 1
        else return 0 end
    `)
    now := time.Now().Unix()
    result, _ := script.Run(context.Background(), rdb,
        []string{"tb:" + userID}, capacity, rate, now).Result()
    return result == int64(1)
}
```

```cpp
bool TokenBucketLimit(Redis& rdb, const std::string& user_id, int capacity, int rate) {
    auto key = "tb:" + user_id;
    auto now = std::chrono::system_clock::to_time_t(
        std::chrono::system_clock::now());
    try {
        auto result = rdb.eval(R"(
            local key = KEYS[1]
            local cap = tonumber(ARGV[1])
            local rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local b = redis.call("HMGET", key, "tokens", "last_time")
            local tokens = tonumber(b[1]) or cap
            local last = tonumber(b[2]) or now
            local elapsed = math.max(now - last, 0)
            local new_tokens = math.min(cap, tokens + elapsed * rate)
            if new_tokens >= 1 then
                redis.call("HMSET", key, "tokens", new_tokens - 1, "last_time", now)
                redis.call("EXPIRE", key, 10)
                return 1
            end
            return 0
        )", {key}, {std::to_string(capacity), std::to_string(rate), std::to_string(now)});
        return result == 1;
    } catch (...) { return false; }
}
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

```go
func Produce(rdb *redis.Client, stream string, data map[string]interface{}) (string, error) {
    return rdb.XAdd(context.Background(), &redis.XAddArgs{
        Stream: stream,
        MaxLen: 10000,
        Values: data,
    }).Result()
}

func Consume(rdb *redis.Client, stream, group, consumer string) {
    for {
        msgs, err := rdb.XReadGroup(context.Background(), &redis.XReadGroupArgs{
            Group:    group,
            Consumer: consumer,
            Streams:  []string{stream, ">"},
            Count:    10,
            Block:    2 * time.Second,
        }).Result()
        if err != nil || len(msgs) == 0 {
            continue
        }
        for _, msg := range msgs {
            for _, m := range msg.Messages {
                if err := processMessage(m.Values); err != nil {
                    rdb.XAdd(context.Background(), &redis.XAddArgs{
                        Stream: "dlq:" + stream,
                        Values: map[string]interface{}{
                            "original_id": m.ID,
                            "error":       err.Error(),
                        },
                    })
                } else {
                    rdb.XAck(context.Background(), stream, group, m.ID)
                }
            }
        }
    }
}
```

```cpp
std::string Produce(Redis& rdb, const std::string& stream,
                    const std::unordered_map<std::string, std::string>& data) {
    return rdb.xadd(stream, "MAXLEN", "~", "10000", "*",
                    data.begin(), data.end());
}

void Consume(Redis& rdb, const std::string& stream,
             const std::string& group, const std::string& consumer) {
    while (true) {
        try {
            auto items = rdb.xreadgroup(group, consumer, stream + ">",
                std::chrono::seconds(2), 10);
            for (auto& [id, data] : items) {
                try {
                    ProcessMessage(data);
                    rdb.xack(stream, group, id);
                } catch (const std::exception& e) {
                    rdb.xadd("dlq:" + stream, "*",
                        "original_id", id, "error", e.what());
                }
            }
        } catch (...) { continue; }
    }
}
```

### 6.2 延迟队列（ZSET）

```go
func EnqueueDelay(rdb *redis.Client, queueKey string, data interface{}, delay time.Duration) {
    b, _ := json.Marshal(data)
    rdb.ZAdd(context.Background(), "delay:"+queueKey, redis.Z{
        Score:  float64(time.Now().Add(delay).Unix()),
        Member: string(b),
    })
}

func ProcessDelayQueue(rdb *redis.Client, queueKey string) {
    key := "delay:" + queueKey
    for {
        now := time.Now().Unix()
        tasks, _ := rdb.ZRangeByScore(context.Background(), key, &redis.ZRangeBy{
            Min: "0", Max: fmt.Sprintf("%d", now), Count: 100,
        }).Result()
        for _, task := range tasks {
            script := redis.NewScript("if redis.call('ZREM', KEYS[1], ARGV[1]) == 1 then return 1 end return 0")
            ok, _ := script.Run(context.Background(), rdb, []string{key}, task).Result()
            if ok == int64(1) {
                var data TaskData
                json.Unmarshal([]byte(task), &data)
                processTask(data)
            }
        }
        time.Sleep(time.Second)
    }
}
```

```cpp
void EnqueueDelay(Redis& rdb, const std::string& queue_key,
                  const std::string& data, int delay_sec) {
    auto score = std::chrono::system_clock::to_time_t(
        std::chrono::system_clock::now()) + delay_sec;
    rdb.zadd("delay:" + queue_key, MemberScore{data, static_cast<double>(score)});
}

void ProcessDelayQueue(Redis& rdb, const std::string& queue_key) {
    auto key = "delay:" + queue_key;
    while (true) {
        auto now = std::chrono::system_clock::to_time_t(
            std::chrono::system_clock::now());
        auto tasks = rdb.zrangebyscore(key, "-inf",
            "(" + std::to_string(now), Limitation{0, 100});
        for (auto& task : tasks) {
            auto ok = rdb.eval(R"(
                if redis.call('ZREM', KEYS[1], ARGV[1]) == 1 then return 1 end
                return 0
            )", {key}, {task});
            if (ok == 1) {
                ProcessTask(task);
            }
        }
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}
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

```go
func Seckill(rdb *redis.Client, userID, productID string) string {
    ctx := context.Background()
    stockKey := "stock:" + productID
    userKey := "seckill_user:" + productID

    exists, _ := rdb.SIsMember(ctx, userKey, userID).Result()
    if exists {
        return "已抢购过"
    }

    script := redis.NewScript(`
        local stock = redis.call("GET", KEYS[1])
        if not stock or tonumber(stock) <= 0 then return 0 end
        redis.call("DECR", KEYS[1])
        redis.call("SADD", KEYS[2], ARGV[1])
        return 1
    `)
    ok, _ := script.Run(ctx, rdb, []string{stockKey, userKey}, userID).Result()
    if ok == int64(0) {
        return "已售罄"
    }
    go SendToOrderQueue(productID, userID)
    return "抢购成功"
}
```

```cpp
std::string Seckill(Redis& rdb, const std::string& user_id,
                    const std::string& product_id) {
    auto stock_key = "stock:" + product_id;
    auto user_key = "seckill_user:" + product_id;

    if (rdb.sismember(user_key, user_id)) return "已抢购过";

    auto result = rdb.eval(R"(
        local stock = redis.call("GET", KEYS[1])
        if not stock or tonumber(stock) <= 0 then return 0 end
        redis.call("DECR", KEYS[1])
        redis.call("SADD", KEYS[2], ARGV[1])
        return 1
    )", {stock_key, user_key}, {user_id});

    if (result == 0) return "已售罄";
    std::thread([=] { SendToOrderQueue(product_id, user_id); }).detach();
    return "抢购成功";
}
```

### 7.2 UV 统计（HyperLogLog）

```go
func RecordUV(rdb *redis.Client, pageID, userID string) {
    key := fmt.Sprintf("uv:%s:%s", pageID, time.Now().Format("2006-01-02"))
    rdb.PFAdd(context.Background(), key, userID)
}

func GetUV(rdb *redis.Client, pageID string) int64 {
    key := fmt.Sprintf("uv:%s:%s", pageID, time.Now().Format("2006-01-02"))
    count, _ := rdb.PFCount(context.Background(), key).Result()
    return count
}

func GetMonthlyUV(rdb *redis.Client, pageID string, year, month int) int64 {
    var keys []string
    for day := 1; day <= 31; day++ {
        key := fmt.Sprintf("uv:%s:%d-%02d-%02d", pageID, year, month, day)
        keys = append(keys, key)
    }
    count, _ := rdb.PFCount(context.Background(), keys...).Result()
    return count
}
```

```cpp
void RecordUV(Redis& rdb, const std::string& page_id, const std::string& user_id) {
    auto key = "uv:" + page_id + ":" + CurrentDateStr();
    rdb.pfadd(key, user_id);
}

int64_t GetUV(Redis& rdb, const std::string& page_id) {
    auto key = "uv:" + page_id + ":" + CurrentDateStr();
    return rdb.pfcount(key);
}

int64_t GetMonthlyUV(Redis& rdb, const std::string& page_id, int year, int month) {
    std::vector<std::string> keys;
    for (int d = 1; d <= 31; ++d) {
        char buf[32];
        snprintf(buf, sizeof(buf), "uv:%s:%d-%02d-%02d", page_id.c_str(), year, month, d);
        keys.push_back(buf);
    }
    return rdb.pfcount(keys.begin(), keys.end());
}
```

### 7.3 附近的人（GEO）

```go
func SetLocation(rdb *redis.Client, userID string, longitude, latitude float64) {
    rdb.GeoAdd(context.Background(), "user_locations", &redis.GeoLocation{
        Name:      userID,
        Longitude: longitude,
        Latitude:  latitude,
    })
}

func FindNearby(rdb *redis.Client, userID string, radius float64) []NearbyUser {
    pos := rdb.GeoPos(context.Background(), "user_locations", userID).Val()
    if len(pos) == 0 {
        return nil
    }
    result, _ := rdb.GeoRadiusByMember(context.Background(), "user_locations",
        userID, &redis.GeoRadiusQuery{
            Radius:    radius,
            Unit:      "km",
            WithDist:  true,
            WithCoord: true,
            Count:     50,
            Sort:      "ASC",
        }).Result()

    var nearby []NearbyUser
    for _, loc := range result {
        nearby = append(nearby, NearbyUser{
            UserID:      loc.Name,
            DistanceKM:  loc.Dist,
            Longitude:   loc.Longitude,
            Latitude:    loc.Latitude,
        })
    }
    return nearby
}
```

```cpp
void SetLocation(Redis& rdb, const std::string& user_id,
                 double longitude, double latitude) {
    rdb.geoadd("user_locations",
               std::make_tuple(longitude, latitude, user_id));
}

std::vector<NearbyUser> FindNearby(Redis& rdb, const std::string& user_id, double radius) {
    auto result = rdb.georadiusbymember("user_locations", user_id, radius,
        GeoRadiusOptions::WITHCOORD | GeoRadiusOptions::WITHDIST,
        Limitation{0, 50});
    std::vector<NearbyUser> nearby;
    for (auto& [uid, dist, _, coord] : result) {
        nearby.push_back({uid, dist, coord.first, coord.second});
    }
    return nearby;
}
```

### 7.4 排行榜（ZSET）

```go
func UpdateScore(rdb *redis.Client, playerID string, delta float64) {
    rdb.ZIncrBy(context.Background(), "leaderboard", delta, playerID)
}

func GetTopN(rdb *redis.Client, n int64) []redis.Z {
    result, _ := rdb.ZRevRangeWithScores(context.Background(), "leaderboard", 0, n-1).Result()
    return result
}

func GetRank(rdb *redis.Client, playerID string) int {
    rank, _ := rdb.ZRevRank(context.Background(), "leaderboard", playerID).Result()
    return int(rank) + 1
}
```

```cpp
void UpdateScore(Redis& rdb, const std::string& player_id, double delta) {
    rdb.zincrby("leaderboard", delta, player_id);
}

std::vector<MemberScore> GetTopN(Redis& rdb, int64_t n) {
    return rdb.zrevrange_withscores("leaderboard", 0, n - 1);
}

int64_t GetRank(Redis& rdb, const std::string& player_id) {
    auto rank = rdb.zrevrank("leaderboard", player_id);
    return rank ? *rank + 1 : -1;
}
```

### 7.5 分布式 Session

```go
import (
    "github.com/gin-gonic/gin"
    "github.com/gorilla/sessions"
    "github.com/gorilla/sessions/redis"
)

func main() {
    store, _ := redis.NewStore(10, "tcp", "redis.example.com:6379", "", []byte("secret-key"))
    store.Options(sessions.Options{
        MaxAge: 86400, // 24 hours
    })

    r := gin.Default()
    r.Use(sessions.Sessions("session", store))

    r.GET("/login", func(c *gin.Context) {
        session := sessions.Default(c)
        session.Set("user_id", 123)
        session.Set("role", "admin")
        session.Save(c.Request, c.Writer)
        c.String(200, "OK")
    })

    r.GET("/profile", func(c *gin.Context) {
        session := sessions.Default(c)
        userID := session.Get("user_id")
        if userID == nil {
            c.String(401, "未登录")
            return
        }
        c.String(200, "User: %d", userID)
    })
}
```

```cpp
// Using cpp_redis for session storage. In practice, integrate with your HTTP framework.
// Example using crow + cpp_redis:
#include <crow.h>
#include <sw/redis++/redis++.h>

class SessionManager {
    Redis& rdb_;
public:
    SessionManager(Redis& rdb) : rdb_(rdb) {}

    void SetSession(const std::string& session_id,
                    const std::string& key, const std::string& value) {
        rdb_.hset("session:" + session_id, key, value);
        rdb_.expire("session:" + session_id, std::chrono::seconds(86400));
    }

    std::string GetSession(const std::string& session_id,
                           const std::string& key) {
        auto val = rdb_.hget("session:" + session_id, key);
        return val ? *val : "";
    }
};

int main() {
    Redis rdb("tcp://redis.example.com:6379");
    SessionManager sm(rdb);

    crow::SimpleApp app;
    CROW_ROUTE(app, "/login").methods("POST"_method)
    ([&](const crow::request& req) {
        auto session_id = UUID::Generate();
        sm.SetSession(session_id, "user_id", "123");
        crow::response res("OK");
        res.set_header("Set-Cookie", "session_id=" + session_id);
        return res;
    });

    app.port(8080).multithreaded().run();
}
```

### 7.6 接口幂等性

```go
func IdempotentHandler(rdb *redis.Client, c *gin.Context) {
    key := c.GetHeader("Idempotent-Key")
    if key == "" {
        c.JSON(400, gin.H{"error": "缺少幂等键"})
        return
    }

    ctx := context.Background()
    ok, _ := rdb.SetNX(ctx, "idempotent:"+key, "processed", 86400*time.Second).Result()
    if !ok {
        result, _ := rdb.Get(ctx, "idempotent_result:"+key).Result()
        c.JSON(200, gin.H{"result": result})
        return
    }

    result := ProcessPayment(c)
    rdb.SetEx(ctx, "idempotent_result:"+key, result, 86400*time.Second)
    c.JSON(200, gin.H{"result": result})
}
```

```cpp
void IdempotentHandler(Redis& rdb, const HttpRequest& req, HttpResponse& res) {
    auto idempotent_key = req.Header("Idempotent-Key");
    if (idempotent_key.empty()) {
        res.Status(400).Json({{"error", "缺少幂等键"}});
        return;
    }

    auto lock_key = "idempotent:" + idempotent_key;
    auto result_key = "idempotent_result:" + idempotent_key;

    if (rdb.set(lock_key, "processed",
                std::chrono::seconds(86400), UpdateType::NOT_EXIST)) {
        auto result = ProcessPayment(req);
        rdb.setex(result_key, 86400, result);
        res.Json({{"result", result}});
    } else {
        auto cached = rdb.get(result_key);
        res.Json({{"result", cached ? *cached : ""}});
    }
}
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

```go
// Cache Aside 模式
func UpdateUser(rdb *redis.Client, u *User) {
    db.UpdateUser(u)
    rdb.Del(context.Background(), fmt.Sprintf("user:%d", u.ID))
}

func GetUser(rdb *redis.Client, userID int) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)
    data, err := rdb.Get(context.Background(), key).Result()
    if err == nil {
        var u User
        json.Unmarshal([]byte(data), &u)
        return &u, nil
    }
    u, err := db.QueryUser(userID)
    if err != nil {
        return nil, err
    }
    b, _ := json.Marshal(u)
    rdb.SetEx(context.Background(), key, string(b), 3600*time.Second)
    return u, nil
}
```

```cpp
void UpdateUser(Redis& rdb, const User& u) {
    Db::UpdateUser(u);
    rdb.del("user:" + std::to_string(u.id));
}

std::optional<User> GetUser(Redis& rdb, int user_id) {
    auto key = "user:" + std::to_string(user_id);
    auto data = rdb.get(key);
    if (data) {
        return User::FromJson(*data);
    }
    auto u = Db::QueryUser(user_id);
    if (u) {
        rdb.setex(key, 3600, u->ToJson());
    }
    return u;
}
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
