---
title: MySQL 数据库使用指南：企业高频面试题与解决方案
date: 2026-06-17 14:00:00
updated: 2026-06-17 14:00:00
categories:
  - 数据库
tags:
  - MySQL
  - 数据库
  - SQL
  - 面试
  - 索引
  - 事务
  - 优化
---

## 一、MySQL 架构概览

### 1.1 分层架构

```
+-----------------------------+
|        客户端 (Client)       |
+-----------------------------+
|    连接器 (Connection)       |
+-----------------------------+
|     查询缓存 (Query Cache)    |  ← MySQL 8.0 已移除
+-----------------------------+
|      分析器 (Parser)          |
+-----------------------------+
|      优化器 (Optimizer)       |
+-----------------------------+
|      执行器 (Executor)        |
+-----------------------------+
|  存储引擎 (InnoDB / MyISAM)  |
+-----------------------------+
|         文件系统              |
+-----------------------------+
```

### 1.2 各层职责

| 组件 | 职责 | 高频考点 |
|:----|:----|:--------|
| 连接器 | 认证、权限校验、管理连接 | `wait_timeout`、`max_connections` |
| 分析器 | 词法分析、语法分析、生成 AST | 解析树、预读 |
| 优化器 | 选择索引、决定 JOIN 顺序、生成执行计划 | `EXPLAIN`、索引选择、`trace` |
| 执行器 | 调用存储引擎接口，返回结果集 | `rows_examined` |
| 存储引擎 | 数据读写、事务、锁、MVCC | InnoDB vs MyISAM |

## 二、存储引擎

### 2.1 InnoDB vs MyISAM

| 特性 | InnoDB | MyISAM |
|:----|:------|:-------|
| 事务 | ✅ 支持（ACID） | ❌ 不支持 |
| 行锁 | ✅ 行级锁 | ❌ 表级锁 |
| MVCC | ✅ | ❌ |
| 外键 | ✅ | ❌ |
| 聚簇索引 | ✅ 必选 | ❌ |
| 全文索引 | ✅（5.6+） | ✅ |
| 缓存 | 缓冲池缓存数据和索引 | 仅缓存索引 |
| 文件 | `.ibd`（数据+索引） | `.MYD`（数据）+ `.MYI`（索引） |
| 崩溃恢复 | ✅ | ❌ 易损坏 |
| COUNT(*) | 慢（需扫描） | 快（单独维护） |

### 2.2 面试题：为什么 InnoDB 用 B+ Tree 而不是 B Tree 或 Hash？

- **B Tree vs B+ Tree**：B+ Tree 只有叶子节点存数据，非叶子节点存指针，同样高度可容纳更多索引条目；叶子节点形成有序链表，支持范围查询
- **Hash**：等值查询 O(1)，但不支持范围查询、ORDER BY、模糊匹配
- **B+ Tree 扇出**：MySQL 页默认 16KB，一个非叶子节点约可存储 1170 个索引项，3 层 B+ Tree 可存约 2000 万行

```sql
# 查看 InnoDB 页大小
SHOW VARIABLES LIKE 'innodb_page_size';  -- 默认 16384
```

### 2.3 面试题：MyISAM 的 COUNT(*) 为什么快？

MyISAM 在表的元数据中单独维护了行数计数器，`COUNT(*)` 无需扫描表直接返回。但带 WHERE 条件的 COUNT 仍然需要扫描。

InnoDB 不维护行数是因为 MVCC 下不同事务看到不同行数，无法统一维护。

## 三、索引

### 3.1 索引分类

| 类型 | 说明 | 语法 |
|:----|:----|:-----|
| 主键索引 | 聚簇索引，叶子节点存整行数据 | `PRIMARY KEY (id)` |
| 唯一索引 | 索引值必须唯一 | `UNIQUE KEY (col)` |
| 普通索引 | 辅助索引，叶子节点存主键值 | `INDEX (col)` |
| 联合索引 | 多列组合，最左前缀原则 | `INDEX (a, b, c)` |
| 全文索引 | 全文检索（InnoDB 5.6+） | `FULLTEXT (col)` |
| 覆盖索引 | 查询列全在索引中，无需回表 | 优化手段，非DDL |

### 3.2 最左前缀原则

```sql
CREATE INDEX idx_a_b_c ON t (a, b, c);

-- 能用到索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 ORDER BY b       -- 索引排序
WHERE a LIKE 'foo%'           -- 前缀匹配

-- 不能用到索引
WHERE b = 2                   -- 跳过第一列
WHERE c = 3                   -- 跳过第一、二列
WHERE a LIKE '%foo'           -- 后缀模糊
WHERE a = 1 OR b = 2          -- OR 可能导致索引失效
```

### 3.3 索引下推 (ICP, Index Condition Pushdown)

MySQL 5.6 引入，在存储引擎层过滤索引记录，减少回表次数：

```sql
CREATE INDEX idx_name_age ON user (name, age);

-- 没有 ICP：从索引找到 name LIKE '张%' 的所有记录，逐一回表查 age
-- 有 ICP：在索引层同时检查 age，过滤后只回表
SELECT * FROM user
WHERE name LIKE '张%' AND age = 20;
```

### 3.4 面试题：为什么建议用自增主键？

1. **性能**：自增主键是递增插入，B+ Tree 新记录追加到右端，页分裂少
2. **空间**：UUID 或业务主键（身份证号）占用空间大，二级索引叶子节点存主键值，放大效应明显
3. **缓存**：顺序写入对预读友好，随机写入导致大量随机 I/O

```sql
-- UUID 主键的代价：二级索引每多一个，就多存一次 UUID（36 字节）
CREATE TABLE bad (
  id CHAR(36) PRIMARY KEY,
  name VARCHAR(50),
  INDEX idx_name (name)
);
-- 建议
CREATE TABLE good (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  uuid CHAR(36) NOT NULL,       -- 业务唯一标识
  name VARCHAR(50),
  UNIQUE KEY uk_uuid (uuid),
  INDEX idx_name (name)
);
```

### 3.5 面试题：联合索引 (a, b) 和 (b, a) 的区别？

- `INDEX(a, b)`：先按 a 排序，a 相同按 b 排序。适用于 `WHERE a=1`、`WHERE a=1 AND b=2`、`ORDER BY a`
- `INDEX(b, a)`：先按 b 排序，b 相同按 a 排序。适用于 `WHERE b=2`、`WHERE a=1 AND b=2`（取决于优化器选择）
- 如果业务中 a 和 b 都频繁独立查询，应建 `INDEX(a)` 和 `INDEX(b)` 各一个

## 四、SQL 优化

### 4.1 EXPLAIN 详解

```sql
EXPLAIN SELECT * FROM user WHERE id = 1\G
```

| 列名 | 含义 | 优化目标 |
|:----|:----|:--------|
| `type` | 访问类型 | const > ref > range > index > ALL |
| `possible_keys` | 可能使用的索引 | 检查是否漏建索引 |
| `key` | 实际使用的索引 | 是否如预期 |
| `rows` | 预估扫描行数 | 越小越好 |
| `Extra` | 额外信息 | Using filesort、Using temporary 是危险信号 |

### 4.2 type 访问类型（性能从好到差）

| type | 含义 | 示例 |
|:----|:----|:-----|
| `system` | 表只有一行 | 极少 |
| `const` | 主键或唯一索引等值查询 | `WHERE id = 1` |
| `eq_ref` | JOIN 中主键或唯一索引关联 | `JOIN b ON a.id = b.id` |
| `ref` | 普通索引等值查询 | `WHERE name = 'foo'` |
| `range` | 索引范围扫描 | `WHERE id > 100` |
| `index` | 全索引扫描 | 比 ALL 略好 |
| `ALL` | 全表扫描 | **必须优化** |

### 4.3 常见优化场景

#### 分页优化（延迟 JOIN）

```sql
-- 常见写法（大分页时越来越慢）
SELECT * FROM user ORDER BY id LIMIT 100000, 20;

-- 优化方案 1：子查询 + 覆盖索引
SELECT * FROM user
WHERE id > (SELECT id FROM user ORDER BY id LIMIT 100000, 1)
ORDER BY id LIMIT 20;

-- 优化方案 2：记录上次最后 ID（适合连续翻页）
SELECT * FROM user
WHERE id > :last_id
ORDER BY id LIMIT 20;
```

#### JOIN 优化

```sql
-- 小表驱动大表，确保 JOIN 列有索引
-- 假设 orders 表远大于 users
-- 错误：驱动表太大
SELECT * FROM orders o LEFT JOIN users u ON o.user_id = u.id;

-- 正确：小表驱动大表
SELECT * FROM users u INNER JOIN orders o ON u.id = o.user_id;

-- MySQL 优化器通常能自动选择 join 顺序，但复杂查询建议 STRAIGHT_JOIN
SELECT STRAIGHT_JOIN * FROM users u INNER JOIN orders o ON u.id = o.user_id;
```

#### GROUP BY / ORDER BY 优化

```sql
-- 危险信号：Using filesort; Using temporary
EXPLAIN SELECT category, COUNT(*)
FROM product
GROUP BY category
ORDER BY COUNT(*) DESC;

-- 优化：建覆盖索引 (category, id)
CREATE INDEX idx_category_id ON product (category, id);
```

### 4.4 面试题：索引失效场景

| 场景 | 示例 | 原因 |
|:----|:----|:-----|
| 隐式类型转换 | `WHERE phone = 13800001111`（phone 是 VARCHAR） | 类型转换使索引失效 |
| 对索引列使用函数 | `WHERE DATE(create_time) = '2026-01-01'` | 函数操作无法使用索引 |
| 模糊匹配后缀 | `WHERE name LIKE '%foo'` | 无法确定前缀 |
| OR 条件 | `WHERE a = 1 OR b = 2`（a 有索引，b 没有） | 需要全表扫描 |
| NOT IN / != | `WHERE status != 1` | 范围过大，优化器放弃索引 |
| 联合索引跳列 | `WHERE b = 1`（索引 (a,b)） | 违反最左前缀 |

```sql
-- 正确写法
WHERE phone = '13800001111'
WHERE create_time >= '2026-01-01' AND create_time < '2026-01-02'
WHERE name LIKE 'foo%'
```

## 五、事务与锁

### 5.1 ACID

| 特性 | 含义 | InnoDB 实现 |
|:----|:----|:-----------|
| A (原子性) | 事务要么全部成功，要么全部回滚 | undo log |
| C (一致性) | 数据始终满足约束 | 应用层 + 数据库约束 |
| I (隔离性) | 事务间互不干扰 | 锁 + MVCC |
| D (持久性) | 已提交事务的数据不丢失 | redo log + doublewrite buffer |

### 5.2 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|:----|:----|:---------|:----|:--------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 不加锁 |
| READ COMMITTED | ❌ | ✅ | ✅ | 语句级快照（每条 SQL 重新生成 ReadView） |
| REPEATABLE READ | ❌ | ❌ | ❌ (InnoDB 解决了) | 事务级快照 + gap lock |
| SERIALIZABLE | ❌ | ❌ | ❌ | 所有读加 S 锁 |

> MySQL 默认隔离级别是 **REPEATABLE READ**。InnoDB 通过 gap lock + next-key lock 解决了幻读。

### 5.3 MVCC 实现原理

```
ReadView 结构：
├── creator_trx_id     -- 当前事务 ID
├── m_ids              -- 活跃的读写事务 ID 列表
├── min_trx_id         -- m_ids 中的最小值
└── max_trx_id         -- 下一个要分配的事务 ID

可见性判定规则：
- trx_id < min_trx_id：已提交，可见
- trx_id >= max_trx_id：未来事务，不可见
- trx_id in m_ids：未提交，不可见（自身除外）
- trx_id < max_trx_id && not in m_ids：已提交，可见
```

### 5.4 行锁类型

```sql
-- 共享锁（S Lock）：允许其他事务读，阻止修改
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;

-- 排他锁（X Lock）：阻止其他事务读和修改
SELECT * FROM user WHERE id = 1 FOR UPDATE;

-- 间隙锁（Gap Lock）：锁定索引记录之间的间隙，防止幻读
-- 只在 REPEATABLE READ 级别生效

-- 临键锁（Next-Key Lock）：行锁 + 间隙锁的组合
```

### 5.5 面试题：死锁分析与处理

```sql
-- 场景：两个事务交叉加锁
-- 事务 A: UPDATE user SET name='a' WHERE id=1;
-- 事务 B: UPDATE user SET name='b' WHERE id=2;
-- 事务 A: UPDATE user SET name='a2' WHERE id=2; -- 等待 B 释放
-- 事务 B: UPDATE user SET name='b2' WHERE id=1; -- 等待 A 释放
-- => 死锁

-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G

-- 查看当前锁等待
SELECT * FROM performance_schema.data_lock_waits\G
SELECT * FROM performance_schema.data_locks\G
```

**预防死锁**：
1. 固定访问顺序：总是按相同顺序访问表和行
2. 减少事务粒度：尽量短的事务
3. 使用 `NOWAIT` 或 `SKIP LOCKED`（MySQL 8.0+）

```sql
-- MySQL 8.0+ 超时或跳过锁
SELECT * FROM user WHERE id = 1 FOR UPDATE NOWAIT;
SELECT * FROM user WHERE id = 1 FOR UPDATE SKIP LOCKED;

-- 设置锁等待超时
SET innodb_lock_wait_timeout = 5;  -- 默认 50 秒
```

### 5.6 面试题：MVCC 能否完全替代锁？

不能。MVCC 解决的是 **读不阻塞写、写不阻塞读**的问题，但以下场景仍需锁：

- `SELECT ... FOR UPDATE` 显式加锁
- `UPDATE`、`DELETE`、`INSERT` 的写操作
- 外键约束检查
- 自增主键计数器

## 六、高可用与扩展

### 6.1 主从复制

```
Master            Slave
  |                 |
  |-- binlog ------>|-- relay log --> SQL 线程
  |     dump 线程    |
```

| 复制方式 | 说明 | 问题 |
|:--------|:----|:----|
| 异步复制 | 主库不等待从库确认 | 主库崩溃可能丢数据 |
| 半同步复制 | 至少一个从库写入 relay log 才返回 | 性能略有下降 |
| 组复制 (MGR) | Paxos 协议，多主写入 | MySQL 5.7+，运维复杂 |

### 6.2 分库分表

| 策略 | 说明 | 适用场景 |
|:----|:----|:--------|
| 垂直分库 | 按业务拆分（订单库、用户库） | 业务耦合度高 |
| 垂直分表 | 大表拆成宽表和窄表 | 字段过多、TEXT/BLOB 分离 |
| 水平分库 | 按 key 分到不同实例 | 数据量亿级 |
| 水平分表 | 按 key 分到同实例不同表 | 单表数据量千万级 |

```sql
-- 水平分表示例：按 user_id 取模
-- 分表键选择原则：尽量按查询最多的维度分
-- 分 64 张表，同实例
CREATE TABLE order_00 LIKE `order`;
CREATE TABLE order_01 LIKE `order`;
-- ...
CREATE TABLE order_63 LIKE `order`;
```

**分库分表面试问题**：

| 问题 | 解决方案 |
|:----|:--------|
| 跨分片 JOIN | 应用层聚合 / 冗余宽表 |
| 全局唯一 ID | 雪花算法 / Redis incr / Leaf |
| 分布式事务 | TCC / Seata / 本地消息表 |
| 分片键变更 | 双写法 / 数据迁移工具 |

### 6.3 读写分离

```sql
-- 应用程序层面实现
-- 写操作走主库，读操作走从库
-- 考虑主从延迟问题

-- 监控主从延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master：延迟秒数
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
```

**读不到刚写的数据（主从延迟）**：
1. 强制读主库（session 级标记「刚写入」）
2. 延迟读从库（`SELECT ... WAIT N SECONDS`）
3. 缓存中间层（Redis 写后立即写入缓存）

## 七、SQL 高频面试题

### 7.1 连续登录问题

```sql
-- 表结构：login(user_id, login_date)
-- 找出连续登录 3 天及以上的用户

WITH t AS (
  SELECT user_id, login_date,
         DATE_SUB(login_date, INTERVAL ROW_NUMBER()
           OVER (PARTITION BY user_id ORDER BY login_date) DAY) AS grp
  FROM (
    SELECT DISTINCT user_id, login_date FROM login
  ) tmp
)
SELECT user_id, MIN(login_date) AS start_date,
       MAX(login_date) AS end_date, COUNT(*) AS days
FROM t
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

### 7.2 部门工资最高的人

```sql
-- 表：employee(id, name, salary, department_id)
-- 找出每个部门工资最高的员工

-- 方法 1：窗口函数
SELECT id, name, salary, department_id
FROM (
  SELECT *, RANK() OVER (
    PARTITION BY department_id ORDER BY salary DESC
  ) AS rk
  FROM employee
) t
WHERE rk = 1;

-- 方法 2：关联子查询
SELECT e.*
FROM employee e
WHERE NOT EXISTS (
  SELECT 1 FROM employee e2
  WHERE e2.department_id = e.department_id
    AND e2.salary > e.salary
);
```

### 7.3 行列转换

```sql
-- 表：score(student_id, subject, score)
-- 行转列（每个学生各科成绩一行）

SELECT student_id,
  MAX(CASE WHEN subject = '语文' THEN score END) AS 语文,
  MAX(CASE WHEN subject = '数学' THEN score END) AS 数学,
  MAX(CASE WHEN subject = '英语' THEN score END) AS 英语
FROM score
GROUP BY student_id;

-- 列转行
-- 表：score2(student_id, 语文, 数学, 英语)
SELECT student_id, '语文' AS subject, 语文 AS score FROM score2
UNION ALL
SELECT student_id, '数学' AS subject, 数学 AS score FROM score2
UNION ALL
SELECT student_id, '英语' AS subject, 英语 AS score FROM score2
ORDER BY student_id;
```

### 7.4 查找重复数据

```sql
-- 找出 email 重复的用户
SELECT email, COUNT(*) AS cnt
FROM user
GROUP BY email
HAVING COUNT(*) > 1;

-- 删除重复保留 ID 最小的
DELETE FROM user
WHERE id NOT IN (
  SELECT MIN(id) FROM user GROUP BY email
);
-- 注意：MySQL 不允许在子查询中引用要删除的表
-- 需要多包一层
DELETE FROM user
WHERE id NOT IN (
  SELECT t.min_id FROM (
    SELECT MIN(id) AS min_id FROM user GROUP BY email
  ) t
);
```

### 7.5 累加求和

```sql
-- 表：sales(month DATE, amount DECIMAL)
-- 计算月度累计销售额

SELECT month, amount,
  SUM(amount) OVER (ORDER BY month) AS cumulative
FROM sales
ORDER BY month;
```

### 7.6 中位数

```sql
-- 计算员工薪资中位数
SELECT AVG(salary) AS median
FROM (
  SELECT salary,
    ROW_NUMBER() OVER (ORDER BY salary) AS rn,
    COUNT(*) OVER () AS total
  FROM employee
) t
WHERE rn BETWEEN total / 2 AND total / 2 + 1;
```

## 八、性能监控与调优

### 8.1 慢查询日志

```sql
-- 开启慢查询
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;      -- 超过 1 秒的 SQL
SET GLOBAL log_queries_not_using_indexes = ON;

-- 分析慢查询
mysqldumpslow -t 10 /var/log/mysql/slow.log
```

### 8.2 常用诊断 SQL

```sql
-- 查看当前正在执行的 SQL
SELECT * FROM sys.processlist WHERE conn_id != CONNECTION_ID();

-- 查看表大小
SELECT table_schema, table_name,
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','sys','performance_schema')
ORDER BY size_mb DESC;

-- 查看索引使用情况
SELECT index_name, seq_in_index, column_name, cardinality
FROM information_schema.statistics
WHERE table_schema = 'db' AND table_name = 't';

-- 缓存命中率
SHOW STATUS LIKE 'Innodb_buffer_pool_read_%';
-- 读命中率 = Innodb_buffer_pool_read_requests
--           / (Innodb_buffer_pool_read_requests + Innodb_buffer_pool_reads)
```

### 8.3 配置优化参考

```ini
# my.cnf 优化（16GB 内存服务器参考）
[mysqld]
# 缓冲池设置为内存的 60-80%
innodb_buffer_pool_size = 10G
# 日志文件大小
innodb_log_file_size = 2G
# 日志缓冲
innodb_log_buffer_size = 64M
# 脏页刷新策略（0=每秒刷, 1=事务提交前刷, 2=事务提交后刷）
innodb_flush_log_at_trx_commit = 2
# 连接数
max_connections = 500
# 临时表
tmp_table_size = 64M
max_heap_table_size = 64M
# 排序缓冲
sort_buffer_size = 4M
join_buffer_size = 4M
```

## 九、常见故障处理

### 9.1 连接数满

```sql
-- 错误：Too many connections
-- 紧急处理：使用 extra_port（MySQL 8.0.14+）或预留连接
SHOW VARIABLES LIKE 'max_connections';

-- 查看当前连接
SHOW PROCESSLIST;

-- 杀掉空闲连接
KILL CONNECTION 123;

-- 或批量杀掉指定状态的连接
SELECT CONCAT('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 300;
```

### 9.2 主从延迟

```sql
-- 查看延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 12345

-- 常见原因与解决：
-- 1. 从库单线程复制 → 开启多线程复制
STOP SLAVE;
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
START SLAVE;

-- 2. 大事务 → 拆分成小事务
-- 3. 从库硬件不足 → 提升从库配置
-- 4. 主库 binlog 刷盘频率 → sync_binlog=1
```

### 9.3 磁盘空间满

```sql
-- 查看各库大小
SELECT table_schema, ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS size_gb
FROM information_schema.tables
GROUP BY table_schema;

-- 清理 binlog（确保从库已追上）
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;
-- 或直接设置过期时间
SET GLOBAL expire_logs_days = 7;

-- 重建大表释放空间（不阻塞读写）：
-- 使用 pt-online-schema-change 或 gh-ost
-- 或手动：建新表 → 写旧表数据 → 重命名
```

## 十、总结

### 高频考点速查表

| 考点 | 核心要点 | 一句话回答 |
|:----|:--------|:---------|
| InnoDB vs MyISAM | 事务、行锁、MVCC、聚簇索引 | 线上环境永远用 InnoDB |
| 索引选择 | B+ Tree、最左前缀、覆盖索引 | 区分度高的列优先，不要建太多索引 |
| 索引失效 | 类型转换、函数、后缀模糊 | 索引列保持干净 |
| 事务隔离级别 | RR 默认、MVCC、gap lock | RR 解决了幻读 |
| 死锁 | SHOW ENGINE INNODB STATUS | 固定访问顺序、缩小事务 |
| 分页优化 | 覆盖索引 + 子查询 | 不要 OFFSET 太大 |
| 主从延迟 | 半同步、多线程复制、读主库 | 先确认业务是否能接受最终一致性 |
| 分库分表 | 分片键、分布式 ID、跨分片查询 | 能不拆尽量不拆 |

### 推荐工具

| 工具 | 用途 |
|:----|:----|
| `EXPLAIN` | 查看执行计划 |
| `mysqldumpslow` | 分析慢查询日志 |
| `pt-query-digest` | Percona 慢查询分析 |
| `sys schema` | MySQL 5.7+ 系统视图 |
| `gh-ost` | 在线 DDL 变更 |
| `MyCLI` | 命令行自动补全 |
