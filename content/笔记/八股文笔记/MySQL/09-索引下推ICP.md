---
title: "索引下推 ICP（MySQL 5.6+）"
created: "2025-07-12"
tags:
  - 八股文
  - mysql
---

# 索引下推 ICP（MySQL 5.6+）

## 一句话总结

> **没有 ICP：二级索引查到 id → 回表拿整行 → 在 server 层过滤其他条件。有 ICP：回表之前先在索引层面把不符合条件的行过滤掉，减少回表次数。核心是"把过滤操作从 server 层下推到存储引擎层"。**

---

## 🌰 没有 ICP 的世界

```sql
-- 联合索引：INDEX (name, age)

SELECT * FROM user WHERE name LIKE '张%' AND age > 25;
```

**没有 ICP（MySQL 5.6 之前）：**

```text
Step 1: 用 idx_name_age 找到所有 name LIKE '张%' 的 id
        └─ 找到了 id = [5, 8, 12, 15, 20, ...]  共 100 个

Step 2: 拿着这 100 个 id → 回表 100 次
        └─ 去聚簇索引拿整行数据

Step 3: 在 server 层过滤 age > 25
        └─ 100 行里只有 30 行满足 → 丢掉 70 行
        └─ 那 70 次回表白干了！
```

**问题：age > 25 这个条件明明在索引里有（idx_name_age 包含了 age），但因为没有 ICP，MySQL 傻乎乎地先把所有 name 匹配的行都回表了，然后在 server 层才过滤 age。**

**70 次回表 = 70 次磁盘 IO = 白白浪费的。**

---

## 有 ICP 的世界

```sql
-- 同样的索引，同样的 SQL
SELECT * FROM user WHERE name LIKE '张%' AND age > 25;
```

**有 ICP（MySQL 5.6+）：**

```text
Step 1: 用 idx_name_age 找到所有 name LIKE '张%' 的行
        └─ 遍历索引时，顺手检查 age > 25

Step 2: 只有同时满足 name LIKE '张%' AND age > 25 的 id 才回表
        └─ 30 个 id → 回表 30 次

Step 3: 直接返回
```

**ICP 把 age > 25 的过滤"下推"到了存储引擎层（索引遍历时做），而不是全部回表再到 server 层过滤。回表次数从 100 次降到了 30 次。**

---

## 覆盖索引 vs ICP

容易跟覆盖索引搞混。核心区别：

| | 覆盖索引 | ICP |
| -- | --------- | ----- |
| **回表了吗** | ❌ 不回 | ✅ 回（但更少）|
| **做了什么** | 查的列全在索引里，不需要回表 | 条件提前在索引里过滤，只回表需要的行 |
| **Extra** | `Using index` | `Using index condition` |
| **效果** | 省掉回表 | 减少回表次数 |

### 三种情况对比

```sql
-- 索引：INDEX (name, age)

-- 情况 1：没 ICP、没覆盖 → 最慢
SELECT * FROM user WHERE name LIKE '张%' AND age > 25;
-- 没有 ICP 时：回表 100 次 → server 层过滤 100 行

-- 情况 2：有 ICP → 更快
-- MySQL 5.6+ 默认启用 ICP
SELECT * FROM user WHERE name LIKE '张%' AND age > 25;
-- Extra: Using index condition
-- 回表 30 次（只回满足两个条件的行）

-- 情况 3：覆盖索引 → 最快
SELECT name, age FROM user WHERE name LIKE '张%' AND age > 25;
-- Extra: Using index
-- 0 次回表
```

---

## ICP 的适用条件

### 必须同时满足

1. **联合索引** — ICP 在单列索引上没有意义（只有一个条件）
2. **索引能定位到范围** — `LIKE '张%'`、`>`、`<`、`BETWEEN` 等范围匹配
3. **条件涉及索引里的额外列** — 除了最左前缀条件外的其他索引列
4. **InnoDB 或 MyISAM** — 两种引擎都支持

### 什么时候不生效

```sql
-- ❌ 条件列不在索引里
-- 索引：INDEX (name)
SELECT * FROM user WHERE name LIKE '张%' AND age > 25;
-- age 不在索引里 → 没法下推

-- ❌ 主键索引
-- ICP 只对二级索引生效，聚簇索引不需要 ICP（已经在数据页了）

-- ❌ 条件无法在索引层面判断
-- 索引：INDEX (name)
SELECT * FROM user WHERE name = '张三' AND age > 25;
-- name 精确匹配时只返回一条或几条，回表成本很低，ICP 收益不大
-- （MySQL 实际也会下推，但效果不明显）
```

---

## 经验数字：ICP 能省多少？

| 场景 | 无 ICP 回表次数 | 有 ICP 回表次数 | 节省 |
| ------ | --------------- | ---------------- | ------ |
| `LIKE '张%'` 匹配 1000 行，`age` 过滤剩 100 行 | 1000 次 | 100 次 | **90%** |
| `LIKE '张%'` 匹配 100 行，`age` 过滤剩 80 行 | 100 次 | 80 次 | 20% |
| `name > 'A'` 匹配 10 万行，`age` 过滤剩 1 万行 | 10 万次 | 1 万次 | **90%** |

**范围匹配越宽泛、后续条件过滤性越强 → ICP 收益越大。**

---

## 怎么聊 ICP

```text
延伸提问："说说索引下推？"

你：
"ICP 是 MySQL 5.6 引入的优化，核心思想是：
把 WHERE 条件里那些可以用索引判断的部分，
从 server 层"下推"到存储引擎层提前过滤。

举个例子：联合索引 (name, age)，查 WHERE name LIKE '张%' AND age > 25。
没有 ICP 的话，MySQL 拿着 '张%' 找到的 id 全部回表，到 server 层再过滤 age。
有 ICP，遍历索引时顺便把 age > 25 检查了，只回表满足两个条件的行。

效果取决于后面条件的过滤性——
age > 25 能过滤掉 90% 的行，回表就省 90%。
如果过滤性差，省得就少，但总归不会慢。"

延伸提问："怎么看有没有用到 ICP？"
"EXPLAIN 的 Extra 列显示 Using index condition。"
```

---

## 记忆口诀

> **ICP = 回表前先过滤，少做无用功。**
> **联合索引范围查，后续条件下推用。**
> **覆盖索引不回表，ICP 少回表——两个加一起，性能翻倍跑。**
> **EXPLAIN 看 Extra：Using index condition，ICP 在工作。**

## 
> ▶ 对应实操：[[09-事务实操|09-事务实操]]

相关链接

- 📋 目录：[[00-MySQL]]
- 📚 学习清单：[[八股文学习清单]]
