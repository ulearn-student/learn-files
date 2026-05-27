### 🎯 阶段 2:深入 PostgreSQL 基础操作

这一阶段我们要做 3 件事:

1. **掌握 psql 命令行**(后端工程师必备技能)
2. **PostgreSQL 特有的 SQL 语法**(和 MySQL 有些差异)
3. **熟悉数据类型 + 高级特性**(JSONB、数组、全文搜索等)

### Step 1:精通 psql 命令行(必备工具)

`psql` 是 PostgreSQL 的官方命令行客户端,**远比图形化工具强大**,后端工程师日常 90% 的时间都在用它。

#### 1.1 进入 psql 的几种方式

powershell

```powershell
# ===== 方式 A:从 Windows 主机直接进入容器内的 psql =====
docker exec -it my-postgres psql -U postgres -d shop_db

# ===== 方式 B:进入容器的 bash,再启动 psql =====
docker exec -it my-postgres bash
# 进入容器后:
psql -U postgres -d shop_db
# 这种方式适合需要在容器内执行其他命令的场景(后面备份会用到)

# ===== 方式 C:用一行命令执行 SQL 后立即退出 =====
docker exec -it my-postgres psql -U postgres -d shop_db -c "SELECT * FROM users;"
# -c: 执行单条 SQL(不进入交互模式)
# 适合写脚本时使用
```

#### 1.2 psql 必学元命令(以反斜杠 \ 开头)

这些是 psql 内部命令,**不是 SQL**,但极其常用:

sql

```sql
-- ============ 数据库相关 ============
\l              -- 列出所有数据库(类似 MySQL 的 SHOW DATABASES)
\c shop_db      -- 切换到 shop_db 数据库(类似 MySQL 的 USE)
\conninfo       -- 显示当前连接信息(连了哪个库、什么用户)

-- ============ 表相关 ============
\dt             -- 列出当前数据库的所有表(类似 SHOW TABLES)
\dt+            -- 列出表 + 附加信息(大小、行数估计)
\d users        -- 查看 users 表的结构(类似 DESC users 或 SHOW CREATE TABLE)
\d+ users       -- 查看 users 表结构 + 详细信息(包括索引、约束、统计信息)

-- ============ 索引和约束 ============
\di             -- 列出所有索引
\dv             -- 列出所有视图
\df             -- 列出所有函数

-- ============ 用户和权限 ============
\du             -- 列出所有用户/角色

-- ============ 实用工具 ============
\timing on      -- 显示每条 SQL 的执行时间(性能优化必备!)
\timing off     -- 关闭执行时间显示
\x              -- 切换显示模式(行/列),长字段查询时超有用
\e              -- 用编辑器打开,写多行 SQL(比直接在终端写舒服)
\h SELECT       -- 查看 SQL 命令的帮助(SELECT 的用法)
\?              -- 查看所有 psql 元命令的帮助
\q              -- 退出 psql
```

#### 1.3 立刻动手:psql 实战练习

powershell

```powershell
# 进入 psql
docker exec -it my-postgres psql -U postgres -d shop_db
```

进入后,依次执行这些命令(每个都试一下):

sql

```sql
-- 1. 打开计时,看看每条 SQL 跑多久
\timing on

-- 2. 列出所有表
\dt

-- 应该看到:
--          List of relations
--  Schema |   Name   | Type  |  Owner   
-- --------+----------+-------+----------
--  public | orders   | table | postgres
--  public | products | table | postgres
--  public | users    | table | postgres


-- 3. 查看 users 表的详细结构
\d+ users

-- 你会看到完整的字段定义、索引、约束信息


-- 4. 执行一些查询(回顾你之前学的 SQL 知识)

-- 查询所有用户
SELECT * FROM users;

-- 用 JOIN 查询订单详情(回顾上一阶段的 INNER JOIN)
SELECT 
    o.id AS 订单号,
    u.username AS 用户,
    p.name AS 商品,
    o.quantity AS 数量,
    o.total AS 总价,
    o.status AS 状态
FROM orders o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN products p ON o.product_id = p.id
ORDER BY o.created_at DESC;

-- 用 CASE WHEN 统计订单状态(回顾上一题的写法)
SELECT 
    COUNT(CASE WHEN status = 'pending' THEN 1 END) AS 待支付,
    COUNT(CASE WHEN status = 'paid' THEN 1 END) AS 已支付,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS 已完成,
    COUNT(*) AS 总数
FROM orders;


-- 5. 长结果切换显示模式(对比一下)
SELECT * FROM users LIMIT 1;
-- 默认表格形式

\x
-- 切换到扩展显示

SELECT * FROM users LIMIT 1;
-- 现在变成"字段名 | 值"的形式,长字段更易读!

\x
-- 切回默认模式

-- 6. 退出
\q
```

### Step 2:PostgreSQL 特有语法(和 MySQL 的关键差异)

PostgreSQL 在 SQL 标准支持上比 MySQL 更完善,有些特性 MySQL 没有,这部分**必须记住**。

sql

```sql
-- 重新进入 psql
-- docker exec -it my-postgres psql -U postgres -d shop_db
```

#### 2.1 自增主键:SERIAL 而非 AUTO_INCREMENT

sql

```sql
-- MySQL 的写法(PostgreSQL 不支持!):
-- id INT AUTO_INCREMENT PRIMARY KEY

-- PostgreSQL 的 4 种自增方式:

-- ===== 方式 A:SERIAL(最常用,PostgreSQL 简写)=====
CREATE TABLE test_serial (
    id SERIAL PRIMARY KEY,           -- 自动是 INT + 自增
    name VARCHAR(50)
);
-- SERIAL: 32位整数(INT)自增,最大约 21 亿
-- BIGSERIAL: 64位整数(BIGINT)自增,几乎用不完(推荐用这个!)
-- SMALLSERIAL: 16位整数自增,最大 32767

-- ===== 方式 B:GENERATED AS IDENTITY(SQL 标准写法,PostgreSQL 10+)=====
CREATE TABLE test_identity (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50)
);
-- 现代写法,更标准、更安全
-- ALWAYS: 严格自增,不允许手动插入 id
-- BY DEFAULT: 可以手动插入 id(更灵活)

-- 验证一下
INSERT INTO test_serial (name) VALUES ('a'), ('b'), ('c');
SELECT * FROM test_serial;
-- 输出:
-- id | name
-- ---+------
--  1 | a
--  2 | b
--  3 | c

-- 清理测试表
DROP TABLE test_serial;
DROP TABLE test_identity;
```

#### 2.2 字符串类型:TEXT 是首选

sql

```sql
-- PostgreSQL 字符串类型对比:
-- VARCHAR(n)  -- 变长字符串,有长度限制
-- CHAR(n)     -- 定长字符串,不足补空格(几乎不用)
-- TEXT        -- 变长字符串,无长度限制 ⭐ 推荐!

-- 重要事实(MySQL 用户的认知颠覆):
-- 在 PostgreSQL 中,TEXT 和 VARCHAR 的【性能完全一样】!
-- PostgreSQL 内部都是用 TOAST 机制存储,没有 MySQL 那种 VARCHAR 性能优势

-- ✅ 推荐: 如果不需要严格长度限制,直接用 TEXT
CREATE TABLE articles (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,         -- 无需限定长度
    content TEXT NOT NULL,
    tags TEXT                    -- 即使存几个字也用 TEXT
);

-- 仅当业务有严格长度限制时才用 VARCHAR(n)
CREATE TABLE phone_book (
    phone VARCHAR(11) NOT NULL   -- 手机号严格 11 位,用 VARCHAR 合理
);
```

#### 2.3 日期时间:TIMESTAMPTZ 而非 DATETIME

sql

```sql
-- MySQL 的 DATETIME 在 PostgreSQL 中不存在!

-- PostgreSQL 日期时间类型:
-- DATE                           -- 仅日期: 2026-05-25
-- TIME                           -- 仅时间: 14:30:00
-- TIMESTAMP                      -- 日期时间: 2026-05-25 14:30:00 (不含时区)
-- TIMESTAMPTZ                    -- 日期时间 + 时区 ⭐ 推荐!
-- INTERVAL                       -- 时间间隔(很强大!)

-- ✅ 国际化业务必用 TIMESTAMPTZ(亚马逊跨境电商场景非常重要!)
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    title TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()  -- NOW() 是 PostgreSQL 的当前时间函数
);

-- 演示时区的威力:
SHOW timezone;                                    -- 当前时区
SELECT NOW();                                     -- 当前时间(带时区)
SELECT NOW() AT TIME ZONE 'America/Los_Angeles'; -- 转换到洛杉矶时间
SELECT NOW() AT TIME ZONE 'Asia/Tokyo';          -- 转换到东京时间

-- INTERVAL(时间间隔)的强大用法
SELECT NOW() - INTERVAL '7 days';        -- 7 天前
SELECT NOW() + INTERVAL '1 month';       -- 1 个月后
SELECT NOW() + INTERVAL '2 hours 30 minutes';  -- 2 小时 30 分钟后

-- 实战:查询最近 7 天的订单
SELECT * FROM orders 
WHERE created_at >= NOW() - INTERVAL '7 days';
```

#### 2.4 JSONB:杀手级特性(MySQL 弱项)

PostgreSQL 的 **JSONB** 是它最强的特性之一,能在关系型数据库里享受 NoSQL 的灵活性。

sql

```sql
-- JSON 和 JSONB 的区别:
-- JSON:   存储原始 JSON 文本,慢但保留格式
-- JSONB:  二进制格式,快得多,支持索引 ⭐ 推荐!

-- 实战:亚马逊商品的灵活属性存储
CREATE TABLE amazon_products (
    id BIGSERIAL PRIMARY KEY,
    asin VARCHAR(10) UNIQUE NOT NULL,        -- 亚马逊商品编号
    title TEXT NOT NULL,
    price DECIMAL(10, 2),
    attributes JSONB,                         -- 灵活的属性字段
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 插入数据(不同商品的属性差异很大,适合用 JSONB)
INSERT INTO amazon_products (asin, title, price, attributes) VALUES
    ('B08N5WRWNW', 'Echo Dot 4', 49.99, '{
        "brand": "Amazon",
        "color": "Charcoal",
        "weight_oz": 12.0,
        "features": ["Alexa", "Bluetooth", "Wi-Fi"],
        "dimensions": {"height": 3.9, "width": 3.9, "depth": 3.5},
        "rating": 4.7,
        "reviews_count": 532840
    }'),
    ('B0BDHWDR12', 'Kindle Paperwhite', 139.99, '{
        "brand": "Amazon",
        "color": "Black",
        "screen_size_inches": 6.8,
        "storage_gb": 16,
        "features": ["Waterproof", "Glare-free"],
        "rating": 4.6,
        "reviews_count": 21503
    }');

-- ===== JSONB 查询语法(超强!)=====

-- 1. 用 -> 获取 JSON 字段(返回 JSON 类型)
SELECT title, attributes -> 'brand' AS brand FROM amazon_products;

-- 2. 用 ->> 获取 JSON 字段(返回文本)
SELECT title, attributes ->> 'brand' AS brand FROM amazon_products;
-- 区别:-> 返回 JSON,如 "Amazon"(带引号);->> 返回纯文本 Amazon

-- 3. 用 #>> 获取嵌套字段
SELECT title, attributes #>> '{dimensions, height}' AS height 
FROM amazon_products;

-- 4. 用 @> 判断包含关系(超常用!)
SELECT * FROM amazon_products 
WHERE attributes @> '{"brand": "Amazon"}';
-- 查询所有 Amazon 品牌的商品

-- 5. 用 ? 判断 key 是否存在
SELECT * FROM amazon_products 
WHERE attributes ? 'storage_gb';
-- 查询所有有 storage_gb 字段的商品(Kindle 有,Echo 没有)

-- 6. JSONB 转换为数字进行比较
SELECT title, (attributes ->> 'rating')::DECIMAL AS rating 
FROM amazon_products 
WHERE (attributes ->> 'rating')::DECIMAL > 4.5
ORDER BY rating DESC;

-- 7. 给 JSONB 建索引(性能优化神器)
CREATE INDEX idx_attributes_brand ON amazon_products ((attributes ->> 'brand'));
-- 这样上面的 brand 查询就走索引了

-- 通用 GIN 索引(支持所有 JSONB 操作)
CREATE INDEX idx_attributes ON amazon_products USING GIN (attributes);
```

#### 2.5 数组类型(原生支持!)

sql

```sql
-- PostgreSQL 原生支持数组,不用搞中间表!

CREATE TABLE blog_posts (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    tags TEXT[],                  -- 字符串数组
    view_counts INT[]             -- 整数数组(比如每天的阅读量)
);

INSERT INTO blog_posts (title, tags, view_counts) VALUES
    ('PostgreSQL 入门', ARRAY['数据库', 'SQL', '教程'], ARRAY[100, 150, 200]),
    ('Docker 实战', ARRAY['Docker', 'DevOps'], ARRAY[80, 90, 120, 200]);

-- 查询包含特定标签的文章
SELECT * FROM blog_posts WHERE 'Docker' = ANY(tags);

-- 查询包含多个标签之一
SELECT * FROM blog_posts WHERE tags && ARRAY['SQL', 'Docker'];
-- && 是数组的"重叠"操作符,只要有一个匹配就返回

-- 数组聚合
SELECT array_length(view_counts, 1) AS days FROM blog_posts;
-- 1 表示维度(一维数组)
```

#### 2.6 UPSERT:INSERT 冲突时的处理

sql

```sql
-- 场景: 同步亚马逊商品,如果 ASIN 已存在就更新,不存在就插入
-- MySQL 用 ON DUPLICATE KEY UPDATE,PostgreSQL 用 ON CONFLICT

INSERT INTO amazon_products (asin, title, price)
VALUES ('B08N5WRWNW', 'Echo Dot 4 (新款)', 39.99)
ON CONFLICT (asin)                    -- 冲突字段(必须有唯一约束/主键)
DO UPDATE SET                         -- 冲突时执行更新
    title = EXCLUDED.title,            -- EXCLUDED 是要插入的新值
    price = EXCLUDED.price;
-- 如果 ASIN B08N5WRWNW 存在,就更新 title 和 price
-- 如果不存在,就插入新行

-- 如果想冲突时啥也不做:
INSERT INTO amazon_products (asin, title, price)
VALUES ('B08N5WRWNW', 'xxx', 0)
ON CONFLICT (asin) DO NOTHING;
```

#### 2.7 RETURNING 子句(节省一次查询)

sql

```sql
-- MySQL: INSERT 后要再 SELECT 查询才能拿到自增 ID
-- PostgreSQL: 一条 SQL 搞定!

-- INSERT 后立即返回插入的数据
INSERT INTO users (username, email, password_hash) 
VALUES ('赵六', 'zhaoliu@example.com', 'hash_xxx')
RETURNING id, username, created_at;
-- 直接返回新插入行的 id、username、created_at

-- UPDATE 后返回更新的数据
UPDATE products 
SET stock = stock - 1 
WHERE id = 1 AND stock > 0
RETURNING id, name, stock;
-- 返回更新后的库存,可以用来判断是否真的更新成功(避免超卖)

-- DELETE 后返回被删除的数据
DELETE FROM orders 
WHERE status = 'cancelled' 
RETURNING *;
```

### Step 3:用 SQL 文件批量执行(实战工作流)

实际开发中,SQL 不会直接在 psql 里手敲,而是写在 .sql 文件中。

#### 3.1 创建 SQL 脚本文件

powershell

```powershell
# 在 postgres-learning 目录下创建一个练习文件夹
cd $HOME\Desktop\postgres-learning
mkdir sql-practice
notepad sql-practice\practice-01.sql
```

**粘贴以下内容并保存:**

sql

```sql
-- 文件: sql-practice/practice-01.sql
-- PostgreSQL 综合练习

-- ========== 1. 创建一个亚马逊销售数据表 ==========
DROP TABLE IF EXISTS amazon_sales;  -- 先删除(如果存在)

CREATE TABLE amazon_sales (
    id BIGSERIAL PRIMARY KEY,
    asin VARCHAR(10) NOT NULL,
    product_name TEXT NOT NULL,
    category TEXT,
    units_sold INT NOT NULL DEFAULT 0,
    revenue DECIMAL(10, 2) NOT NULL DEFAULT 0,
    sale_date DATE NOT NULL,
    metadata JSONB,                              -- 灵活属性
    tags TEXT[],                                  -- 标签数组
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ========== 2. 插入测试数据(模拟 3 天销售数据)==========
INSERT INTO amazon_sales (asin, product_name, category, units_sold, revenue, sale_date, metadata, tags) VALUES
    ('B08N5WRWNW', 'Echo Dot 4', 'Electronics', 50, 2499.50, '2025-05-23', '{"country": "US", "fbm": false}', ARRAY['smart-home', 'best-seller']),
    ('B08N5WRWNW', 'Echo Dot 4', 'Electronics', 35, 1749.65, '2025-05-24', '{"country": "US", "fbm": false}', ARRAY['smart-home', 'best-seller']),
    ('B08N5WRWNW', 'Echo Dot 4', 'Electronics', 42, 2099.58, '2025-05-25', '{"country": "US", "fbm": false}', ARRAY['smart-home', 'best-seller']),
    ('B0BDHWDR12', 'Kindle Paperwhite', 'Electronics', 20, 2799.80, '2025-05-23', '{"country": "US", "fbm": false}', ARRAY['e-reader']),
    ('B0BDHWDR12', 'Kindle Paperwhite', 'Electronics', 25, 3499.75, '2025-05-24', '{"country": "US", "fbm": false}', ARRAY['e-reader']),
    ('B07FZ8S74R', 'Yoga Mat', 'Sports', 100, 1999.00, '2025-05-23', '{"country": "UK", "fbm": true}', ARRAY['fitness', 'yoga']),
    ('B07FZ8S74R', 'Yoga Mat', 'Sports', 80, 1599.20, '2025-05-24', '{"country": "UK", "fbm": true}', ARRAY['fitness', 'yoga']);

-- ========== 3. 查询练习 ==========

-- Q1: 查询每个商品的总销量和总收入(按总收入排序)
SELECT 
    asin AS 商品编号,
    product_name AS 商品名称,
    SUM(units_sold) AS 总销量,
    SUM(revenue) AS 总收入,
    ROUND(AVG(revenue), 2) AS 日均收入
FROM amazon_sales
GROUP BY asin, product_name
ORDER BY 总收入 DESC;

-- Q2: 查询每天的销售汇总
SELECT 
    sale_date AS 日期,
    COUNT(DISTINCT asin) AS 商品数,
    SUM(units_sold) AS 总销量,
    SUM(revenue) AS 总收入
FROM amazon_sales
GROUP BY sale_date
ORDER BY sale_date;

-- Q3: 用 JSONB 查询 - 只看美国市场
SELECT 
    asin, product_name, units_sold, revenue
FROM amazon_sales
WHERE metadata @> '{"country": "US"}';

-- Q4: 用数组查询 - 包含 'best-seller' 标签的商品
SELECT 
    asin, product_name, tags
FROM amazon_sales
WHERE 'best-seller' = ANY(tags);

-- Q5: 复杂查询 - 计算每个商品的销量趋势(用窗口函数)
SELECT 
    asin,
    product_name,
    sale_date,
    units_sold,
    -- LAG: 获取上一行的值(同一商品的前一天销量)
    LAG(units_sold, 1) OVER (PARTITION BY asin ORDER BY sale_date) AS 前一天销量,
    -- 计算环比变化
    units_sold - LAG(units_sold, 1) OVER (PARTITION BY asin ORDER BY sale_date) AS 销量变化
FROM amazon_sales
ORDER BY asin, sale_date;
```

#### 3.2 执行 SQL 文件的 3 种方式

powershell

```powershell
# ===== 方式 A:从 Windows 直接执行(推荐)=====
# 把文件内容传给 psql

Get-Content sql-practice\practice-01.sql | docker exec -i my-postgres psql -U postgres -d shop_db
# Get-Content: PowerShell 的读取文件命令(类似 Linux 的 cat)
# | : 管道,把内容传给下一个命令
# -i : 让容器接收标准输入(没有 -t,因为不需要交互)


# ===== 方式 B:把 SQL 文件复制到容器内再执行 =====
# 1. 复制文件
docker cp sql-practice\practice-01.sql my-postgres:/tmp/practice-01.sql
# 2. 在容器内执行
docker exec -it my-postgres psql -U postgres -d shop_db -f /tmp/practice-01.sql


# ===== 方式 C:利用 volumes 映射(最优雅,生产推荐)=====
# 编辑 docker-compose.yml,添加一行 volume 映射:
# volumes:
#   - ./sql-practice:/sql-practice
# 
# 然后就可以直接:
docker exec -it my-postgres psql -U postgres -d shop_db -f /sql-practice/practice-01.sql
```

#### 3.3 把查询结果导出到文件

powershell

```powershell
# 把 SQL 查询结果输出到 Windows 上的文件
docker exec -it my-postgres psql -U postgres -d shop_db -c "SELECT * FROM users;" > query-result.txt

# 或者导出 CSV 格式(实用!)
docker exec -it my-postgres psql -U postgres -d shop_db -c "\COPY users TO STDOUT WITH CSV HEADER" > users.csv
```

### 📋 阶段 2 检验清单

确认这些都掌握了再进入下一阶段:

```
✅ 能熟练用 \dt、\d、\l、\timing 等元命令
✅ 理解 SERIAL/BIGSERIAL vs MySQL 的 AUTO_INCREMENT
✅ 知道 TEXT 在 PostgreSQL 是首选,而非 VARCHAR
✅ 会用 TIMESTAMPTZ 和 INTERVAL 处理时间
✅ 能用 JSONB 的 @> 操作符做包含查询
✅ 能用 ON CONFLICT 实现 UPSERT
✅ 能用 RETURNING 节省一次查询
✅ 知道如何从 Windows 执行 .sql 文件
```