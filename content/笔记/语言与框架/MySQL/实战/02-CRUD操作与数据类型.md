---

title: "CRUD操作与数据类型"

created: "2025-07-12"

tags:

  - 技术学习

  - mysql

  - crud

---

# CRUD操作与数据类型

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行 SQL。

---

## 一、插入数据（INSERT）
### 1.1 为什么 INSERT 是第一个学？

你得先把数据放进表里，才能查、改、删。INSERT 就是"向货架上放东西"。

### 1.2 基本语法

```sql

INSERT INTO 表名 (列1, 列2, ...) VALUES (值1, 值2, ...);

-- 实际例子：插入一个用户

INSERT INTO users (username, email, age, balance)

VALUES ('zhangsan', 'zhangsan@qq.com', 25, 100.00);

```

**结果：**

```text

Query OK, 1 row affected (0.01 sec)

```

### 1.3 三种写法对比

```sql

-- 写法 1：指定列名，按顺序填值（推荐！）

INSERT INTO users (username, email, age)

VALUES ('lisi', 'lisi@qq.com', 30);

-- 写法 2：省略列名，按建表时的字段顺序全部填值

-- ⚠️ 必须填所有字段（包括有默认值的），而且顺序不能错

INSERT INTO users

VALUES (NULL, 'wangwu', 'wangwu@qq.com', 28, 0.00, NOW());

-- 写法 3：指定部分列，其他列自动用默认值或 NULL

INSERT INTO users (username, email)

VALUES ('zhaoliu', 'zhaoliu@qq.com');

-- age 填 NULL，balance 用默认值 0.00，created_at 用当前时间

```

> [!TIP]

> **推荐写法 1，明确指定列名。** 以后别人改了表结构（比如加了个字段），你的 SQL 还能正常运行；写法 2 会直接报错。

### 1.4 批量插入

```sql

-- 一次插入多条数据（用逗号分隔）

INSERT INTO users (username, email, age, balance) VALUES

    ('user01', 'user01@qq.com', 20, 50.00),

    ('user02', 'user02@qq.com', 22, 100.00),

    ('user03', 'user03@qq.com', 25, 200.00),

    ('user04', 'user04@qq.com', 28, 150.00),

    ('user05', 'user05@qq.com', 30, 300.00);

```

### 1.5 插入订单数据（为后面的查询练习准备）

```sql

INSERT INTO orders (user_id, product, price, quantity, total) VALUES

    (1, '机械键盘', 299.00, 1, 299.00),

    (1, '鼠标', 149.00, 2, 298.00),

    (2, '显示器', 1999.00, 1, 1999.00),

    (2, '鼠标垫', 29.00, 3, 87.00),

    (3, '机械键盘', 299.00, 2, 598.00),

    (3, '显示器', 2499.00, 1, 2499.00),

    (3, '数据线', 19.90, 5, 99.50),

    (4, '鼠标', 149.00, 1, 149.00),

    (5, '机械键盘', 259.00, 1, 259.00),

    (5, '鼠标垫', 35.00, 2, 70.00);

```

---

## 二、查询数据（SELECT）
### 2.1 SELECT 为什么是最重要、最常用的操作？

现实中 90% 的 SQL 都是 SELECT。用户打开网页 → 后端查数据库 → 返回数据展示。你刷淘宝看到商品列表，背后就是一堆 SELECT。

### 2.2 基本查询

```sql

-- 查全表（所有列，所有行）

-- ⚠️ 生产环境慎用：如果表有几百万行，SELECT * 会拖垮数据库

SELECT * FROM users;

-- 查指定列

SELECT username, email FROM users;

-- 查指定列 + 给列起别名（显示时用中文，方便看）

SELECT username AS '用户名', email AS '邮箱', age AS '年龄' FROM users;

```

**`SELECT * FROM users;` 输出：**

```text

+----+----------+------------------+------+---------+---------------------+

| id | username | email            | age  | balance | created_at          |

+----+----------+------------------+------+---------+---------------------+

|  1 | zhangsan | zhangsan@qq.com  |   25 |  100.00 | 2025-01-15 10:00:00 |

|  2 | lisi     | lisi@qq.com      |   30 |    0.00 | 2025-01-15 10:00:00 |

|  3 | wangwu   | wangwu@qq.com    |   28 |    0.00 | 2025-01-15 10:00:00 |

|  4 | zhaoliu  | zhaoliu@qq.com   | NULL |    0.00 | 2025-01-15 10:00:00 |

|  5 | user01   | user01@qq.com    |   20 |   50.00 | 2025-01-15 10:00:01 |

|  6 | user02   | user02@qq.com    |   22 |  100.00 | 2025-01-15 10:00:01 |

|  7 | user03   | user03@qq.com    |   25 |  200.00 | 2025-01-15 10:00:01 |

|  8 | user04   | user04@qq.com    |   28 |  150.00 | 2025-01-15 10:00:01 |

|  9 | user05   | user05@qq.com    |   30 |  300.00 | 2025-01-15 10:00:01 |

+----+----------+------------------+------+---------+---------------------+

```

### 2.3 DISTINCT —— 去重

```sql

-- 查询用户都来自哪些年龄段（重复的年龄只显示一次）

SELECT DISTINCT age FROM users;

-- 结果: 20, 22, 25, 28, 30, NULL

-- 去重可以多列组合（两列都相同的才去重）

SELECT DISTINCT age, balance FROM users;

```

### 2.4 SELECT 的执行顺序（先知道有这个概念）

SQL 不是按你写的顺序执行的。SELECT 语句的实际执行顺序是：

```text

① FROM     —— 先知道查哪张表

② WHERE    —— 再筛掉不符合条件的行

③ SELECT   —— 然后挑出要显示的列

④ ORDER BY —— 最后排序

⑤ LIMIT    —— 最后截取

```

> [!IMPORTANT]

> **知道这个顺序很重要。** 比如你不能在 WHERE 里用 SELECT 里定义的别名，因为 WHERE 先执行。

---

## 三、更新数据（UPDATE）
### 3.1 基本语法

```sql

-- UPDATE 表名 SET 列1=值1, 列2=值2 WHERE 条件;

UPDATE users SET balance = 500.00 WHERE id = 1;

-- 一次更新多个字段

UPDATE users SET age = 26, email = 'new_email@qq.com' WHERE id = 1;

```

### 3.2 忘记 WHERE 的灾难

```sql

-- ❌❌❌ 毁灭性操作：没有 WHERE，全表都会被更新！

UPDATE users SET balance = 0.00;

-- 所有人的余额全变成 0！

-- ✅ 安全习惯：写 UPDATE / DELETE 时，先写 WHERE 条件，再补 SET。

```

### 3.3 基于原值更新

```sql

-- 给所有用户的 balance 加 50

UPDATE users SET balance = balance + 50 WHERE id IN (1, 2, 3);

-- 等价于 Python 里的: user.balance = user.balance + 50

```

---

## 四、删除数据（DELETE）
### 4.1 基本语法

```sql

-- DELETE FROM 表名 WHERE 条件;

DELETE FROM users WHERE id = 9;

-- 删除了 user05

-- 删多条

DELETE FROM users WHERE age IS NULL;

```

### 4.2 忘记 WHERE 是全表删除

```sql

-- ❌❌❌ 毁灭性操作！

DELETE FROM users;

-- 所有用户数据全没了！

```

### 4.3 DELETE vs TRUNCATE

```sql

-- DELETE：逐行删除，可以带 WHERE，AUTO_INCREMENT 不会重置，能回滚

DELETE FROM users WHERE id > 100;

-- TRUNCATE：清空整个表，不能带 WHERE，AUTO_INCREMENT 重置为 1，不能回滚

TRUNCATE TABLE users;

```

| | DELETE | TRUNCATE |
|---|---|---|
| **能加 WHERE 吗** | ✅ 可以 | ❌ 不可以，只能清空全表 |
| **速度** | 慢（逐行删除） | 快（直接销毁表再重建） |
| **AUTO_INCREMENT** | 不会重置 | 重置为 1 |
| **能回滚吗** | ✅ 在事务中可以 | ❌ 不能（DDL 语句） |
| **适用场景** | 删除特定行 | 清空整张表重新导入数据 |

---

## 速查表

| 操作 | 语法 | 一句话解释 |
|------|------|-----------|
| **插入** | `INSERT INTO 表名 (列...) VALUES (值...);` | 往货架上放一件东西 |
| **批量插入** | `INSERT INTO 表名 (列...) VALUES (行1), (行2), ...;` | 一次放多件 |
| **查询** | `SELECT 列 FROM 表名 WHERE 条件;` | 从货架上取符合条件的东西 |
| **查全表** | `SELECT * FROM 表名;` | 把所有东西都搬出来（慎用） |
| **别名** | `SELECT 列 AS '别名' FROM 表名;` | 给列起个临时名字 |
| **去重** | `SELECT DISTINCT 列 FROM 表名;` | 只看不重复的值 |
| **更新** | `UPDATE 表名 SET 列=值 WHERE 条件;` | 修改货架上某件东西的标签 |
| **基于原值更新** | `UPDATE 表名 SET 列=列+增量 WHERE 条件;` | 在原值基础上增减 |
| **删除** | `DELETE FROM 表名 WHERE 条件;` | 把某件东西从货架上拿掉 |
| **清空表** | `TRUNCATE TABLE 表名;` | 把整个货架清空重建 |

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「CRUD操作与数据类型」到底是什么？**

A：CRUD 是数据库的四种基本操作：Create 插入、Read 查询、Update 更新、Delete 删除。

**Q2：一、插入数据（INSERT） —— 怎么理解？**

A：INSERT 是"向货架上放东西"。推荐写法指名列名 `INSERT INTO users (username,email) VALUES (...)`——别人改表结构你的 SQL 仍跑得通；省略列名写法必须填所有字段且顺序不能错，易崩。还支持批量插入（多组 VALUES 逗号分隔）和指定部分列用默认值。

**Q3：二、查询数据（SELECT） —— 怎么理解？**

A：SELECT 是最重要的操作（现实 90% SQL 都是它）。`SELECT *` 查全表生产慎用（几百万行拖垮库）；可查指定列加别名。DISTINCT 去重（多列都同才去重）。关键：SQL 执行顺序 FROM→WHERE→SELECT→ORDER BY→LIMIT，所以 WHERE 里不能用 SELECT 里定义的别名（因为 WHERE 先执行）。

**Q4：三、更新数据（UPDATE） —— 怎么理解？**

A：UPDATE 改货架上某件东西的标签，`SET 列=值 WHERE 条件`。致命坑：忘写 WHERE 全表都被改（所有人余额变 0）！安全习惯先写 WHERE 再补 SET。还能基于原值更新 `balance=balance+50`（等于 Python 的 `x=x+50`）。

**Q5：四、删除数据（DELETE） —— 怎么理解？**

A：DELETE 拿掉货架上某件东西，`DELETE FROM 表 WHERE 条件`；忘 WHERE 整表数据全没。DELETE vs TRUNCATE：DELETE 逐行删、可带 WHERE、AUTO_INCREMENT 不重置、事务内可回滚；TRUNCATE 清空整表、不能 WHERE、自增重置为 1、不能回滚（DDL 语句）。

**Q6：核心速记主线有哪些？**

- INSERT 推荐指名列名；支持批量插入与部分列默认值

- SELECT 最常用；执行顺序 FROM→WHERE→SELECT→ORDER BY→LIMIT

- UPDATE 忘 WHERE 全表改，先写 WHERE 再 SET

- DELETE 忘 WHERE 全表删；TRUNCATE 不可回滚且重置自增

**口诀**

A：增指列名改表不乱，查按 FROM WHERE 序；

UPDATE 先 WHERE 后 SET，忘写全表都完蛋；

DELETE 带条件拿一件，TRUNCATE 清空不可挽；

CRUD 四字记心间，货架比喻最直观。

## 相关链接

- 目录：[[00-MySQL]]

- 上一篇：[[01-数据库基础与安装]]

- 下一篇：[[03-条件查询与排序分页]]

