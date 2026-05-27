# 阶段 3:Node.js 调用 PostgreSQL 完整指南

> 从最底层的原生驱动,到生产级的 ORM 应用,以及高并发、事务、连接池等核心知识。所有代码带详细中文注释。

---

## 📚 目录

- [一、环境准备](#一环境准备)
- [二、Step 1:初始化 Node.js 项目](#二step-1初始化-nodejs-项目)
- [三、Step 2:用原生 pg 库连接数据库](#三step-2用原生-pg-库连接数据库)
- [四、Step 3:连接池(Pool)详解](#四step-3连接池pool详解)
- [五、Step 4:CRUD 完整实战](#五step-4crud-完整实战)
- [六、Step 5:事务处理(Transaction)](#六step-5事务处理transaction)
- [七、Step 6:用 Express 搭建 RESTful API](#七step-6用-express-搭建-restful-api)
- [八、Step 7:用 Prisma ORM 重写](#八step-7用-prisma-orm-重写)
- [九、Step 8:对比与最佳实践](#九step-8对比与最佳实践)
- [十、常见错误排查](#十常见错误排查)

---

## 一、环境准备

### 前置条件检查

```powershell
# 确认 PostgreSQL 容器在运行
docker compose ps
# 应该看到 my-postgres 状态为 Up (healthy)

# 确认 Node.js 已安装
node -v
# 应该显示 v18 或更高版本

npm -v
# 显示 npm 版本号
```

### 学习路径全景图

```
┌─────────────────────────────────────────────────────┐
│ 路径 1: 原生 pg 库            → 理解底层原理        │
│   ↓                                                  │
│ 路径 2: 连接池配置            → 高并发的基础        │
│   ↓                                                  │
│ 路径 3: 完整的 CRUD API       → 业务实战            │
│   ↓                                                  │
│ 路径 4: Express + REST API   → 工程化开发           │
│   ↓                                                  │
│ 路径 5: Prisma ORM 重写       → 现代化开发          │
│   ↓                                                  │
│ 路径 6: 对比和最佳实践        → 架构思维            │
└─────────────────────────────────────────────────────┘
```

---

## 二、Step 1:初始化 Node.js 项目

### 1.1 创建项目

```powershell
# 进入之前的项目文件夹
cd $HOME\Desktop\postgres-learning

# 创建 Node.js 子项目
mkdir node-app
cd node-app

# 初始化 package.json
# -y 表示全部用默认值,不用一个个回答问题
npm init -y

# 安装核心依赖
npm install pg dotenv
# pg: PostgreSQL 的 Node.js 官方驱动(node-postgres),最权威的库
# dotenv: 从 .env 文件读取环境变量,避免把数据库密码写在代码里

# 安装开发工具(只在开发环境用)
npm install --save-dev nodemon
# nodemon: 文件变化时自动重启 Node 应用,提升开发效率
```

### 1.2 修改 package.json

打开 `package.json`,在 `"scripts"` 部分添加启动脚本:

```json
{
  "name": "node-app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "type": "commonjs",
  "dependencies": {
    "dotenv": "^16.4.5",
    "pg": "^8.13.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.7"
  }
}
```

### 1.3 创建配置文件 .env

```powershell
# 创建环境变量文件
notepad .env
```

粘贴以下内容:

```bash
# .env 文件 - 环境变量配置(不要提交到 Git!)

# ============ 数据库连接配置 ============
DB_HOST=localhost          # 数据库主机(Docker 映射到本机的 localhost)
DB_PORT=5432               # 数据库端口
DB_USER=postgres           # 用户名
DB_PASSWORD=mysecret123    # 密码(和 docker-compose.yml 中一致)
DB_NAME=shop_db            # 数据库名

# ============ 应用配置 ============
NODE_ENV=development       # 环境标识:development/production
PORT=3000                  # Node 应用监听的端口
```

### 1.4 创建 .gitignore(养成好习惯)

```powershell
notepad .gitignore
```

粘贴以下内容:

```
# Node.js
node_modules/
npm-debug.log*

# 环境变量(包含敏感信息)
.env
.env.local

# 编辑器
.vscode/
.idea/

# 系统文件
.DS_Store
Thumbs.db

# 日志
*.log
logs/
```

### 1.5 项目结构

完成后,你的项目结构应该是:

```
postgres-learning/
├── docker-compose.yml
├── init-scripts/
├── backups/
└── node-app/                  ← 新建的
    ├── package.json
    ├── package-lock.json
    ├── .env                   ← 数据库配置
    ├── .gitignore
    └── node_modules/          ← 自动生成
```

---

## 三、Step 2:用原生 pg 库连接数据库

先用最底层的方式连接,理解原理后再用 ORM。

### 2.1 最简单的连接示例

创建 `01-simple-connect.js`:

```javascript
// 文件: 01-simple-connect.js
// 最简单的数据库连接示例 - 用 Client(单连接)

// ========== 1. 导入依赖 ==========
const { Client } = require('pg');           // 从 pg 库导入 Client 类
require('dotenv').config();                  // 加载 .env 文件中的环境变量

// ========== 2. 创建数据库客户端 ==========
const client = new Client({
    host: process.env.DB_HOST,               // 从环境变量读取(避免硬编码)
    port: process.env.DB_PORT,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
});

// ========== 3. 异步主函数 ==========
// 用 async/await 是 Node.js 处理异步操作的现代写法
async function main() {
    try {
        // 3.1 连接数据库
        await client.connect();
        console.log('✅ 数据库连接成功!');
        
        // 3.2 执行一条简单查询
        // query() 是 pg 库的核心方法,返回一个 Promise
        const result = await client.query('SELECT NOW() AS current_time');
        // result 对象包含很多信息,常用的:
        // - result.rows:    查询返回的数据数组
        // - result.rowCount: 影响的行数
        // - result.fields:  字段元信息
        
        console.log('当前数据库时间:', result.rows[0].current_time);
        
        // 3.3 查询用户表(回顾之前学的 SQL)
        const users = await client.query('SELECT * FROM users');
        console.log(`\n📊 查询到 ${users.rowCount} 个用户:`);
        console.table(users.rows);  // 用表格形式打印,超直观!
        
    } catch (error) {
        // 任何步骤出错都会进入这里
        console.error('❌ 数据库操作失败:', error.message);
    } finally {
        // 4. 无论成功失败,都要关闭连接(类似 try-with-resources)
        await client.end();
        console.log('🔚 数据库连接已关闭');
    }
}

// ========== 4. 执行主函数 ==========
main();
```

**运行:**

```powershell
node 01-simple-connect.js
```

**预期输出:**

```
✅ 数据库连接成功!
当前数据库时间: 2026-05-25T...
📊 查询到 3 个用户:
┌─────────┬────┬──────────┬─────────────────────────┐
│ (index) │ id │ username │ email                   │
├─────────┼────┼──────────┼─────────────────────────┤
│ 0       │ 1  │ '张三'   │ 'zhangsan@example.com'  │
│ 1       │ 2  │ '李四'   │ 'lisi@example.com'      │
│ 2       │ 3  │ '王五'   │ 'wangwu@example.com'    │
└─────────┴────┴──────────┴─────────────────────────┘
🔚 数据库连接已关闭
```

### 2.2 参数化查询(防止 SQL 注入!)

⚠️ **极其重要的安全知识!** 永远不要用字符串拼接来构造 SQL!

创建 `02-parameterized-query.js`:

```javascript
// 文件: 02-parameterized-query.js
// 参数化查询 - 防止 SQL 注入的标准做法

const { Client } = require('pg');
require('dotenv').config();

const client = new Client({
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
});

async function main() {
    try {
        await client.connect();
        
        // 假设这是用户从前端传来的查询参数(可能是恶意输入!)
        const userInputId = 1;
        const userInputName = "张三";
        
        // ===== ❌ 错误写法:字符串拼接(SQL 注入风险!)=====
        /*
        const dangerousSQL = `SELECT * FROM users WHERE id = ${userInputId}`;
        // 如果用户输入 "1 OR 1=1",就会变成:
        // SELECT * FROM users WHERE id = 1 OR 1=1
        // 结果是返回所有用户数据!严重安全漏洞!
        
        // 更危险的情况:用户输入 "1; DROP TABLE users; --"
        // 整个表都会被删除!
        */
        
        // ===== ✅ 正确写法:参数化查询 =====
        // PostgreSQL 用 $1, $2, $3... 作为占位符(注意:不是 ? 也不是 %s)
        const safeSQL = 'SELECT * FROM users WHERE id = $1';
        const params = [userInputId];
        
        const result = await client.query(safeSQL, params);
        // pg 库会自动:
        // 1. 把 $1 替换为参数值
        // 2. 自动转义特殊字符
        // 3. 防止 SQL 注入攻击
        
        console.log('✅ 查询结果:');
        console.table(result.rows);
        
        // ===== 多个参数的写法 =====
        const multiParamSQL = `
            SELECT * FROM users 
            WHERE username = $1 AND id = $2
        `;
        const result2 = await client.query(multiParamSQL, [userInputName, 1]);
        console.log('\n多参数查询结果:');
        console.table(result2.rows);
        
        // ===== INSERT 的参数化(带 RETURNING)=====
        const insertSQL = `
            INSERT INTO users (username, email, password_hash, age)
            VALUES ($1, $2, $3, $4)
            RETURNING id, username, created_at
        `;
        const newUser = await client.query(insertSQL, [
            '测试用户' + Date.now(),  // 用时间戳避免重复
            `test${Date.now()}@example.com`,
            'hashed_password',
            22
        ]);
        console.log('\n✅ 新插入用户:');
        console.log(newUser.rows[0]);
        // RETURNING 返回插入的数据,包括自动生成的 id 和 created_at
        
    } catch (error) {
        console.error('❌ 错误:', error.message);
    } finally {
        await client.end();
    }
}

main();
```

**关键知识点:**

```
🔑 PostgreSQL 参数占位符规则:
  - 用 $1, $2, $3... (顺序编号)
  - 不是 MySQL 的 ?
  - 不是 Python 的 %s
  - 占位符不要加引号,pg 会自动处理

🔑 SQL 注入防范:
  - 永远用参数化查询
  - 永远不要拼接用户输入
  - 即使是数字也要用参数化(数字也可能被注入!)
```

---

## 四、Step 3:连接池(Pool)详解

**连接池是高并发应用的核心!** 直接用 Client 在生产环境会让数据库瞬间崩溃。

### 3.1 为什么需要连接池?

```
❌ 没有连接池的问题:

请求 1 → 创建连接 → 查询 → 关闭连接   (耗时 ~50ms)
请求 2 → 创建连接 → 查询 → 关闭连接   (耗时 ~50ms)
请求 3 → 创建连接 → 查询 → 关闭连接   (耗时 ~50ms)
...
高并发时(1000 req/s):
- TCP 握手开销巨大
- 数据库认证消耗大量 CPU
- PostgreSQL 默认 max_connections = 100,直接撑爆!

✅ 有连接池:

应用启动时 → 预先创建 20 个连接,放入"池子"

请求 1 → 从池子借连接 → 查询 → 还回池子   (耗时 ~5ms)
请求 2 → 从池子借连接 → 查询 → 还回池子   (耗时 ~5ms)
请求 3 → 等待空闲连接 → 查询 → 还回池子   (排队等待)

性能提升 10 倍以上!
```

### 3.2 创建连接池

创建 `db.js`(整个应用共用的数据库模块):

```javascript
// 文件: db.js
// 数据库连接池模块 - 整个应用共享一个连接池实例

const { Pool } = require('pg');
require('dotenv').config();

// ========== 创建连接池 ==========
const pool = new Pool({
    // 基础连接配置
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    
    // ========== 连接池关键参数(必须理解!)==========
    
    max: 20,
    // 连接池最大连接数
    // 默认 10,生产环境根据并发量调整
    // 经验值: max ≈ CPU 核心数 × 4 ~ 10
    // ⚠️ 不要设太大!超过 PostgreSQL 的 max_connections 会报错
    
    min: 2,
    // 连接池最小连接数(保持的"热"连接)
    // 默认 0,设为 2-5 可以减少冷启动时间
    
    idleTimeoutMillis: 30000,
    // 连接空闲多久后被回收(毫秒)
    // 默认 10000(10秒),设 30000 减少频繁创建
    
    connectionTimeoutMillis: 5000,
    // 获取连接的超时时间(毫秒)
    // 池子满了的话,新请求等多久就放弃
    // 默认 0(永不超时,危险!),建议 5000
    
    // ========== 高级配置 ==========
    
    statement_timeout: 10000,
    // 单个 SQL 语句的最大执行时间(毫秒)
    // 防止慢查询拖垮数据库,默认无限制
    
    query_timeout: 10000,
    // 查询的总超时(包括连接获取等)
    
    application_name: 'node-app',
    // 应用名(在 PostgreSQL 的 pg_stat_activity 中可见,便于排查)
});

// ========== 监听连接池事件(便于调试)==========

pool.on('connect', (client) => {
    console.log('🔌 新连接建立,当前连接数:', pool.totalCount);
});

pool.on('acquire', (client) => {
    // 从池中借用连接时触发(频繁,调试时再打开)
    // console.log('📤 借用连接,池中空闲:', pool.idleCount);
});

pool.on('remove', (client) => {
    console.log('🗑️  连接被移除,当前连接数:', pool.totalCount);
});

pool.on('error', (err, client) => {
    // 关键!监听错误事件,避免应用崩溃
    console.error('💥 连接池错误:', err.message);
});

// ========== 优雅关闭(应用退出时清理资源)==========
process.on('SIGINT', async () => {
    console.log('\n👋 收到退出信号,正在关闭连接池...');
    await pool.end();  // 等待所有正在进行的查询完成后关闭
    process.exit(0);
});

// 导出 pool 给其他文件使用
module.exports = pool;
```

### 3.3 测试连接池

创建 `03-test-pool.js`:

```javascript
// 文件: 03-test-pool.js
// 测试连接池的工作方式 - 模拟并发请求

const pool = require('./db');

// 模拟 50 个并发查询
async function simulateConcurrentQueries() {
    console.log('🚀 开始模拟 50 个并发查询...\n');
    
    const startTime = Date.now();
    
    // 创建 50 个并发查询的 Promise 数组
    const queries = [];
    for (let i = 0; i < 50; i++) {
        // pool.query() 自动从池中借连接、执行查询、归还连接
        const queryPromise = pool.query('SELECT pg_sleep(0.1), $1 AS query_id', [i])
            .then(result => {
                console.log(`✅ 查询 ${result.rows[0].query_id} 完成`);
            })
            .catch(err => {
                console.error(`❌ 查询 ${i} 失败:`, err.message);
            });
        
        queries.push(queryPromise);
    }
    
    // 等待所有查询完成
    await Promise.all(queries);
    
    const duration = Date.now() - startTime;
    console.log(`\n📊 50 个查询全部完成,总耗时: ${duration}ms`);
    console.log(`📊 连接池状态:`);
    console.log(`   总连接数: ${pool.totalCount}`);
    console.log(`   空闲连接: ${pool.idleCount}`);
    console.log(`   等待中的请求: ${pool.waitingCount}`);
    
    // 关闭连接池
    await pool.end();
}

simulateConcurrentQueries();

// ========== 关键观察点 ==========
// 1. 虽然有 50 个查询,但只创建了 20 个数据库连接(max=20)
// 2. 多余的查询会"排队"等待空闲连接
// 3. 总耗时不是 50 × 100ms = 5000ms,而是更少
//    因为前 20 个并发执行,后续依次进入
```

**运行:**

```powershell
node 03-test-pool.js
```

### 3.4 连接池调优参考

```
🎯 不同场景的推荐配置:

【低并发应用】(个人项目、内部工具)
{
    max: 5,
    min: 1,
    idleTimeoutMillis: 60000
}

【中等并发应用】(中小型 SaaS)
{
    max: 20,
    min: 2,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000
}

【高并发应用】(电商、社交)
{
    max: 50,
    min: 10,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 3000,
    statement_timeout: 5000
}

⚠️ 黄金法则:
   max 不要超过 PostgreSQL 的 max_connections × 70%
   留余地给监控、备份等系统连接
```

---

## 五、Step 4:CRUD 完整实战

把连接池用起来,封装一个用户操作模块。

### 4.1 创建用户操作模块

创建 `userRepository.js`:

```javascript
// 文件: userRepository.js
// 用户数据访问层 (Repository 模式)
// Repository 模式: 封装数据库操作,业务代码只调用方法,不写 SQL

const pool = require('./db');

// ========== 1. 查询所有用户 ==========
async function findAll() {
    const sql = `
        SELECT id, username, email, age, created_at 
        FROM users 
        ORDER BY created_at DESC
    `;
    const result = await pool.query(sql);
    return result.rows;  // 返回数组
}

// ========== 2. 根据 ID 查询单个用户 ==========
async function findById(id) {
    const sql = `
        SELECT id, username, email, age, created_at 
        FROM users 
        WHERE id = $1
    `;
    const result = await pool.query(sql, [id]);
    return result.rows[0] || null;  // 返回单个对象,没找到返回 null
}

// ========== 3. 根据 Email 查询用户(登录场景)==========
async function findByEmail(email) {
    const sql = 'SELECT * FROM users WHERE email = $1';
    const result = await pool.query(sql, [email]);
    return result.rows[0] || null;
}

// ========== 4. 创建新用户 ==========
async function create(userData) {
    const { username, email, password_hash, age } = userData;
    
    const sql = `
        INSERT INTO users (username, email, password_hash, age)
        VALUES ($1, $2, $3, $4)
        RETURNING id, username, email, age, created_at
    `;
    
    try {
        const result = await pool.query(sql, [username, email, password_hash, age]);
        return result.rows[0];
    } catch (error) {
        // 处理唯一约束冲突
        if (error.code === '23505') {
            // PostgreSQL 错误码 23505 = unique_violation
            throw new Error(`Email 或用户名已被注册`);
        }
        throw error;  // 其他错误向上抛
    }
}

// ========== 5. 更新用户 ==========
async function update(id, updates) {
    // 动态构建 UPDATE 语句(只更新提供的字段)
    const allowedFields = ['username', 'email', 'age'];
    const setClauses = [];
    const values = [];
    let paramIndex = 1;
    
    // 遍历允许更新的字段,只构造提供的字段
    for (const field of allowedFields) {
        if (updates[field] !== undefined) {
            setClauses.push(`${field} = $${paramIndex}`);
            values.push(updates[field]);
            paramIndex++;
        }
    }
    
    if (setClauses.length === 0) {
        throw new Error('没有提供要更新的字段');
    }
    
    // 自动更新 updated_at
    setClauses.push(`updated_at = NOW()`);
    
    // ID 是最后一个参数
    values.push(id);
    
    const sql = `
        UPDATE users 
        SET ${setClauses.join(', ')}
        WHERE id = $${paramIndex}
        RETURNING id, username, email, age, updated_at
    `;
    
    const result = await pool.query(sql, values);
    return result.rows[0] || null;
}

// ========== 6. 删除用户 ==========
async function deleteById(id) {
    const sql = 'DELETE FROM users WHERE id = $1 RETURNING id';
    const result = await pool.query(sql, [id]);
    return result.rowCount > 0;  // 返回是否删除成功
}

// ========== 7. 分页查询(实战必备)==========
async function findWithPagination({ page = 1, pageSize = 10, search = '' }) {
    // page: 第几页(从 1 开始)
    // pageSize: 每页多少条
    // search: 搜索关键词(可选)
    
    const offset = (page - 1) * pageSize;
    
    // 构造 WHERE 条件
    let whereClause = '';
    const params = [];
    
    if (search) {
        whereClause = 'WHERE username ILIKE $1 OR email ILIKE $1';
        // ILIKE: PostgreSQL 的大小写不敏感模糊匹配(LIKE 是大小写敏感)
        params.push(`%${search}%`);
    }
    
    // 查询总数
    const countSQL = `SELECT COUNT(*) FROM users ${whereClause}`;
    const countResult = await pool.query(countSQL, params);
    const total = parseInt(countResult.rows[0].count);
    
    // 查询数据
    const dataSQL = `
        SELECT id, username, email, age, created_at 
        FROM users 
        ${whereClause}
        ORDER BY created_at DESC
        LIMIT $${params.length + 1} OFFSET $${params.length + 2}
    `;
    const dataResult = await pool.query(dataSQL, [...params, pageSize, offset]);
    
    return {
        data: dataResult.rows,
        pagination: {
            page,
            pageSize,
            total,
            totalPages: Math.ceil(total / pageSize)
        }
    };
}

// 导出所有方法
module.exports = {
    findAll,
    findById,
    findByEmail,
    create,
    update,
    deleteById,
    findWithPagination
};
```

### 4.2 测试 CRUD

创建 `04-test-crud.js`:

```javascript
// 文件: 04-test-crud.js
// 测试 userRepository 的所有方法

const userRepo = require('./userRepository');
const pool = require('./db');

async function main() {
    try {
        console.log('========== 1. 查询所有用户 ==========');
        const users = await userRepo.findAll();
        console.table(users);
        
        console.log('\n========== 2. 根据 ID 查询 ==========');
        const user = await userRepo.findById(1);
        console.log(user);
        
        console.log('\n========== 3. 创建新用户 ==========');
        const newUser = await userRepo.create({
            username: `用户${Date.now()}`,
            email: `user${Date.now()}@example.com`,
            password_hash: 'hashed_xxx',
            age: 25
        });
        console.log('✅ 新用户:', newUser);
        
        console.log('\n========== 4. 更新用户 ==========');
        const updated = await userRepo.update(newUser.id, {
            age: 26,
            username: '更新后的名字'
        });
        console.log('✅ 更新后:', updated);
        
        console.log('\n========== 5. 分页查询 ==========');
        const paged = await userRepo.findWithPagination({
            page: 1,
            pageSize: 5
        });
        console.log('数据:', paged.data);
        console.log('分页信息:', paged.pagination);
        
        console.log('\n========== 6. 删除用户 ==========');
        const deleted = await userRepo.deleteById(newUser.id);
        console.log('删除成功:', deleted);
        
        console.log('\n========== 7. 测试唯一约束错误 ==========');
        try {
            await userRepo.create({
                username: '张三',  // 已存在的用户名
                email: 'duplicate@example.com',
                password_hash: 'xxx',
                age: 30
            });
        } catch (error) {
            console.log('✅ 正确捕获错误:', error.message);
        }
        
    } catch (error) {
        console.error('💥 错误:', error);
    } finally {
        await pool.end();
    }
}

main();
```

---

## 六、Step 5:事务处理(Transaction)

**事务是数据库的核心特性**,保证多个操作要么全部成功,要么全部失败。

### 5.1 经典场景:转账(必须事务!)

```
场景: 张三给李四转账 100 元
步骤 1: 张三账户 - 100
步骤 2: 李四账户 + 100

❌ 没有事务的灾难:
执行步骤 1 成功后,服务器突然宕机
→ 张三的钱扣了,李四的钱没加
→ 100 元凭空消失!

✅ 用事务保证:
所有操作要么全成功(commit),要么全回滚(rollback)
```

### 5.2 PostgreSQL 事务基础

```sql
-- 事务的三个关键命令:
BEGIN;           -- 开始事务
-- 中间执行各种 SQL...
COMMIT;          -- 提交事务(全部生效)
-- 或者
ROLLBACK;        -- 回滚事务(全部撤销)
```

### 5.3 Node.js 中的事务实现

创建 `05-transaction.js`:

```javascript
// 文件: 05-transaction.js
// 事务处理实战 - 电商下单场景

const pool = require('./db');

// ========== 场景: 用户下单 ==========
// 涉及操作:
// 1. 检查商品库存
// 2. 创建订单
// 3. 扣减库存
// 4. (可选)扣除用户余额
// 这些操作必须在同一个事务中!

async function createOrder(userId, productId, quantity) {
    // ⚠️ 关键!事务必须使用同一个 client,不能用 pool.query
    // 因为不同 query 可能从池中借不同连接,事务就不在同一个连接上了
    
    const client = await pool.connect();  // 从池中借一个专用连接
    
    try {
        // ===== 1. 开始事务 =====
        await client.query('BEGIN');
        console.log('🚀 事务开始');
        
        // ===== 2. 查询商品(加锁防止并发问题)=====
        // FOR UPDATE: 给查询的行加"行锁",其他事务必须等本事务结束才能修改
        // 这是处理"超卖"问题的关键!
        const productResult = await client.query(
            'SELECT id, name, price, stock FROM products WHERE id = $1 FOR UPDATE',
            [productId]
        );
        
        if (productResult.rows.length === 0) {
            throw new Error('商品不存在');
        }
        
        const product = productResult.rows[0];
        console.log(`📦 查询到商品: ${product.name}, 当前库存: ${product.stock}`);
        
        // ===== 3. 检查库存 =====
        if (product.stock < quantity) {
            throw new Error(`库存不足! 需要 ${quantity} 件,实际只有 ${product.stock} 件`);
        }
        
        // ===== 4. 计算总价 =====
        const total = (parseFloat(product.price) * quantity).toFixed(2);
        
        // ===== 5. 创建订单 =====
        const orderResult = await client.query(`
            INSERT INTO orders (user_id, product_id, quantity, total, status)
            VALUES ($1, $2, $3, $4, 'pending')
            RETURNING id, created_at
        `, [userId, productId, quantity, total]);
        
        const order = orderResult.rows[0];
        console.log(`📝 订单已创建: #${order.id}`);
        
        // ===== 6. 扣减库存 =====
        await client.query(
            'UPDATE products SET stock = stock - $1 WHERE id = $2',
            [quantity, productId]
        );
        console.log(`📉 库存已扣减 ${quantity} 件`);
        
        // ===== 7. 提交事务 =====
        await client.query('COMMIT');
        console.log('✅ 事务提交成功!');
        
        return {
            orderId: order.id,
            total: total,
            createdAt: order.created_at
        };
        
    } catch (error) {
        // ===== 出错时回滚 =====
        await client.query('ROLLBACK');
        console.error('❌ 事务回滚:', error.message);
        throw error;  // 把错误抛给上层
        
    } finally {
        // ===== 关键!必须归还连接到池子 =====
        client.release();
        console.log('🔌 连接已归还');
    }
}

// ========== 测试 ==========
async function testTransaction() {
    try {
        // 正常下单
        console.log('\n========== 测试 1: 正常下单 ==========');
        const order1 = await createOrder(1, 1, 2);
        console.log('成功:', order1);
        
        // 库存不足
        console.log('\n========== 测试 2: 库存不足 ==========');
        try {
            await createOrder(1, 1, 99999);
        } catch (error) {
            console.log('预期的错误:', error.message);
        }
        
        // 商品不存在
        console.log('\n========== 测试 3: 商品不存在 ==========');
        try {
            await createOrder(1, 99999, 1);
        } catch (error) {
            console.log('预期的错误:', error.message);
        }
        
    } finally {
        await pool.end();
    }
}

testTransaction();
```

### 5.4 封装事务工具函数

每次都写 BEGIN/COMMIT/ROLLBACK 很繁琐,封装一个工具:

创建 `transaction.js`:

```javascript
// 文件: transaction.js
// 事务工具函数 - 简化事务代码

const pool = require('./db');

/**
 * 在事务中执行回调函数
 * @param {Function} callback - 接收 client 参数的异步函数
 * @returns 回调函数的返回值
 * 
 * 使用示例:
 *   const result = await withTransaction(async (client) => {
 *       await client.query('INSERT ...');
 *       await client.query('UPDATE ...');
 *       return 'success';
 *   });
 */
async function withTransaction(callback) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        const result = await callback(client);  // 执行业务逻辑
        await client.query('COMMIT');
        return result;
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}

module.exports = { withTransaction };
```

**使用封装后的事务工具:**

```javascript
// 重写 createOrder,代码大幅简化
const { withTransaction } = require('./transaction');

async function createOrder(userId, productId, quantity) {
    return await withTransaction(async (client) => {
        // 直接写业务逻辑,事务处理自动完成
        
        const productResult = await client.query(
            'SELECT * FROM products WHERE id = $1 FOR UPDATE',
            [productId]
        );
        
        if (productResult.rows.length === 0) throw new Error('商品不存在');
        const product = productResult.rows[0];
        if (product.stock < quantity) throw new Error('库存不足');
        
        const total = (parseFloat(product.price) * quantity).toFixed(2);
        
        const orderResult = await client.query(`
            INSERT INTO orders (user_id, product_id, quantity, total, status)
            VALUES ($1, $2, $3, $4, 'pending')
            RETURNING *
        `, [userId, productId, quantity, total]);
        
        await client.query(
            'UPDATE products SET stock = stock - $1 WHERE id = $2',
            [quantity, productId]
        );
        
        return orderResult.rows[0];
    });
    // 自动 COMMIT(成功) 或 ROLLBACK(失败)
}
```

### 5.5 事务隔离级别(进阶)

```javascript
// PostgreSQL 支持 4 种隔离级别(从低到高):

// 1. READ UNCOMMITTED - 读未提交(PostgreSQL 实际等同于 READ COMMITTED)
await client.query('SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED');

// 2. READ COMMITTED - 读已提交(PostgreSQL 默认!)
// 只能读到已提交的数据,但同一事务中两次查询结果可能不同
await client.query('SET TRANSACTION ISOLATION LEVEL READ COMMITTED');

// 3. REPEATABLE READ - 可重复读
// 同一事务中多次查询结果保持一致
await client.query('SET TRANSACTION ISOLATION LEVEL REPEATABLE READ');

// 4. SERIALIZABLE - 串行化(最严格)
// 完全隔离,但性能最低,可能出现序列化冲突错误
await client.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');

// 💡 实战建议:
// - 默认 READ COMMITTED 满足 90% 场景
// - 涉及金钱/库存等关键业务,用 SERIALIZABLE 或 FOR UPDATE 行锁
```

---

## 七、Step 6:用 Express 搭建 RESTful API

把数据库操作做成 HTTP API,这才是真实的应用场景。

### 6.1 安装 Express

```powershell
npm install express
```

### 6.2 创建 API 服务器

创建 `server.js`:

```javascript
// 文件: server.js
// Express HTTP 服务器 - 提供 RESTful API

const express = require('express');
const userRepo = require('./userRepository');
const { withTransaction } = require('./transaction');
const pool = require('./db');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

// ========== 中间件 ==========
app.use(express.json());  // 解析 JSON 请求体

// 请求日志中间件
app.use((req, res, next) => {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}`);
    next();
});

// ========== 健康检查接口 ==========
app.get('/health', async (req, res) => {
    try {
        // 测试数据库连接
        const result = await pool.query('SELECT NOW()');
        res.json({
            status: 'ok',
            database: 'connected',
            time: result.rows[0].now,
            pool: {
                total: pool.totalCount,
                idle: pool.idleCount,
                waiting: pool.waitingCount
            }
        });
    } catch (error) {
        res.status(500).json({ status: 'error', message: error.message });
    }
});

// ========== 用户相关 API ==========

// GET /api/users - 获取用户列表(支持分页和搜索)
app.get('/api/users', async (req, res) => {
    try {
        const { page = 1, pageSize = 10, search = '' } = req.query;
        const result = await userRepo.findWithPagination({
            page: parseInt(page),
            pageSize: parseInt(pageSize),
            search
        });
        res.json(result);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// GET /api/users/:id - 获取单个用户
app.get('/api/users/:id', async (req, res) => {
    try {
        const user = await userRepo.findById(req.params.id);
        if (!user) {
            return res.status(404).json({ error: '用户不存在' });
        }
        res.json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// POST /api/users - 创建新用户
app.post('/api/users', async (req, res) => {
    try {
        const { username, email, password, age } = req.body;
        
        // 简单的输入验证
        if (!username || !email || !password) {
            return res.status(400).json({ 
                error: 'username, email, password 不能为空' 
            });
        }
        
        // 实际项目中应该用 bcrypt 加密密码
        const password_hash = `hashed_${password}`;
        
        const user = await userRepo.create({ username, email, password_hash, age });
        res.status(201).json(user);  // 201 Created
        
    } catch (error) {
        if (error.message.includes('已被注册')) {
            res.status(409).json({ error: error.message });  // 409 Conflict
        } else {
            res.status(500).json({ error: error.message });
        }
    }
});

// PUT /api/users/:id - 更新用户
app.put('/api/users/:id', async (req, res) => {
    try {
        const updated = await userRepo.update(req.params.id, req.body);
        if (!updated) {
            return res.status(404).json({ error: '用户不存在' });
        }
        res.json(updated);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// DELETE /api/users/:id - 删除用户
app.delete('/api/users/:id', async (req, res) => {
    try {
        const success = await userRepo.deleteById(req.params.id);
        if (!success) {
            return res.status(404).json({ error: '用户不存在' });
        }
        res.status(204).send();  // 204 No Content
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// ========== 订单 API(带事务)==========

// POST /api/orders - 下单
app.post('/api/orders', async (req, res) => {
    try {
        const { userId, productId, quantity } = req.body;
        
        // 在事务中执行下单逻辑
        const order = await withTransaction(async (client) => {
            // 1. 查询并锁定商品
            const productResult = await client.query(
                'SELECT * FROM products WHERE id = $1 FOR UPDATE',
                [productId]
            );
            
            if (productResult.rows.length === 0) {
                throw new Error('商品不存在');
            }
            
            const product = productResult.rows[0];
            
            // 2. 检查库存
            if (product.stock < quantity) {
                throw new Error('库存不足');
            }
            
            // 3. 计算总价
            const total = (parseFloat(product.price) * quantity).toFixed(2);
            
            // 4. 创建订单
            const orderResult = await client.query(`
                INSERT INTO orders (user_id, product_id, quantity, total, status)
                VALUES ($1, $2, $3, $4, 'pending')
                RETURNING *
            `, [userId, productId, quantity, total]);
            
            // 5. 扣减库存
            await client.query(
                'UPDATE products SET stock = stock - $1 WHERE id = $2',
                [quantity, productId]
            );
            
            return orderResult.rows[0];
        });
        
        res.status(201).json(order);
        
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

// ========== 404 处理 ==========
app.use((req, res) => {
    res.status(404).json({ error: '接口不存在' });
});

// ========== 全局错误处理 ==========
app.use((error, req, res, next) => {
    console.error('💥 未捕获的错误:', error);
    res.status(500).json({ error: '服务器内部错误' });
});

// ========== 启动服务器 ==========
app.listen(PORT, () => {
    console.log(`🚀 服务器启动: http://localhost:${PORT}`);
    console.log(`📍 健康检查: http://localhost:${PORT}/health`);
});
```

### 6.3 启动服务器并测试

```powershell
# 启动服务器
npm run dev

# 看到 "🚀 服务器启动: http://localhost:3000" 就成功了
```

**用浏览器测试:**

```
访问: http://localhost:3000/health
访问: http://localhost:3000/api/users
```

**用 curl 或 Postman 测试 POST 请求:**

```powershell
# 创建新用户
curl -X POST http://localhost:3000/api/users `
  -H "Content-Type: application/json" `
  -d '{\"username\":\"测试\",\"email\":\"test@test.com\",\"password\":\"123\",\"age\":25}'

# 下单(测试事务)
curl -X POST http://localhost:3000/api/orders `
  -H "Content-Type: application/json" `
  -d '{\"userId\":1,\"productId\":1,\"quantity\":1}'
```

---

## 八、Step 7:用 Prisma ORM 重写

体验过原生 SQL 的繁琐后,用 Prisma 简化代码。

### 7.1 安装 Prisma

```powershell
# 安装 Prisma
npm install --save-dev prisma
npm install @prisma/client

# 初始化(自动创建 prisma 文件夹和配置)
npx prisma init
```

### 7.2 配置 .env(Prisma 用的)

Prisma 默认从 `.env` 读取 DATABASE_URL,需要添加这一行:

```bash
# 添加到 .env 文件末尾
DATABASE_URL="postgresql://postgres:mysecret123@localhost:5432/shop_db?schema=public"
```

### 7.3 从现有数据库生成 Schema

由于我们的数据库已经存在,可以反向生成 Prisma Schema:

```powershell
# 自动从数据库读取表结构,生成 schema.prisma
npx prisma db pull

# 生成 Prisma Client(TypeScript 类型)
npx prisma generate
```

打开 `prisma/schema.prisma`,你会看到自动生成的内容,可以稍作美化:

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model users {
  id            BigInt    @id @default(autoincrement())
  username      String    @unique @db.VarChar(50)
  email         String    @unique @db.VarChar(100)
  password_hash String    @db.VarChar(255)
  age           Int?
  created_at    DateTime? @default(now()) @db.Timestamp(6)
  updated_at    DateTime? @default(now()) @db.Timestamp(6)
  orders        orders[]
}

model products {
  id          BigInt   @id @default(autoincrement())
  name        String   @db.VarChar(255)
  price       Decimal  @db.Decimal(10, 2)
  stock       Int      @default(0)
  description String?
  created_at  DateTime? @default(now()) @db.Timestamp(6)
  orders      orders[]
}

model orders {
  id         BigInt    @id @default(autoincrement())
  user_id    BigInt
  product_id BigInt
  quantity   Int
  total      Decimal   @db.Decimal(10, 2)
  status     String?   @default("pending") @db.VarChar(20)
  created_at DateTime? @default(now()) @db.Timestamp(6)
  users      users     @relation(fields: [user_id], references: [id])
  products   products  @relation(fields: [product_id], references: [id])
  
  @@index([user_id])
  @@index([product_id])
  @@index([status])
}
```

### 7.4 用 Prisma 重写用户操作

创建 `userRepository.prisma.js`:

```javascript
// 文件: userRepository.prisma.js
// 用 Prisma 重写用户操作 - 对比原生 SQL 的简洁性

const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient({
    log: ['query', 'info', 'warn', 'error']  // 打印所有 SQL,便于学习
});

// ========== 1. 查询所有用户 ==========
async function findAll() {
    // Prisma 写法:
    return await prisma.users.findMany({
        orderBy: { created_at: 'desc' }
    });
    
    // 对比原生 SQL 写法:
    // const result = await pool.query('SELECT * FROM users ORDER BY created_at DESC');
    // return result.rows;
}

// ========== 2. 根据 ID 查询 ==========
async function findById(id) {
    return await prisma.users.findUnique({
        where: { id: BigInt(id) }  // 注意:数据库是 BIGINT,需要转换
    });
}

// ========== 3. 创建用户 ==========
async function create(userData) {
    try {
        return await prisma.users.create({
            data: {
                username: userData.username,
                email: userData.email,
                password_hash: userData.password_hash,
                age: userData.age
            }
        });
    } catch (error) {
        if (error.code === 'P2002') {  // Prisma 的唯一约束错误码
            throw new Error('Email 或用户名已被注册');
        }
        throw error;
    }
}

// ========== 4. 更新用户 ==========
async function update(id, updates) {
    return await prisma.users.update({
        where: { id: BigInt(id) },
        data: updates  // Prisma 自动忽略 undefined 字段
    });
}

// ========== 5. 删除用户 ==========
async function deleteById(id) {
    try {
        await prisma.users.delete({
            where: { id: BigInt(id) }
        });
        return true;
    } catch (error) {
        if (error.code === 'P2025') return false;  // 记录不存在
        throw error;
    }
}

// ========== 6. 分页查询(对比一下,简洁多了!)==========
async function findWithPagination({ page = 1, pageSize = 10, search = '' }) {
    const where = search ? {
        OR: [
            { username: { contains: search, mode: 'insensitive' } },
            { email: { contains: search, mode: 'insensitive' } }
        ]
    } : {};
    
    // 并行查询总数和数据(性能优化!)
    const [total, data] = await Promise.all([
        prisma.users.count({ where }),
        prisma.users.findMany({
            where,
            skip: (page - 1) * pageSize,
            take: pageSize,
            orderBy: { created_at: 'desc' }
        })
    ]);
    
    return {
        data,
        pagination: {
            page,
            pageSize,
            total,
            totalPages: Math.ceil(total / pageSize)
        }
    };
}

// ========== 7. 复杂查询:用户 + 订单(关联查询)==========
async function findUserWithOrders(userId) {
    return await prisma.users.findUnique({
        where: { id: BigInt(userId) },
        include: {
            orders: {
                include: {
                    products: true  // 嵌套关联查询
                },
                orderBy: { created_at: 'desc' }
            }
        }
    });
    
    // 原生 SQL 写法需要 JOIN 多张表 + 在代码中重组数据,代码量是这个的 5 倍以上!
}

module.exports = {
    findAll,
    findById,
    create,
    update,
    deleteById,
    findWithPagination,
    findUserWithOrders,
    prisma  // 导出 prisma 实例,便于关闭连接
};
```

### 7.5 Prisma 事务

创建 `06-prisma-transaction.js`:

```javascript
// 文件: 06-prisma-transaction.js
// Prisma 的事务用法 - 对比原生 SQL 的简洁性

const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function createOrderWithPrisma(userId, productId, quantity) {
    // Prisma 的交互式事务(超简洁!)
    return await prisma.$transaction(async (tx) => {
        // tx 是事务上下文,所有操作在同一事务中
        
        // 1. 查询商品
        const product = await tx.products.findUnique({
            where: { id: BigInt(productId) }
        });
        
        if (!product) throw new Error('商品不存在');
        if (product.stock < quantity) throw new Error('库存不足');
        
        // 2. 计算总价
        const total = Number(product.price) * quantity;
        
        // 3. 创建订单
        const order = await tx.orders.create({
            data: {
                user_id: BigInt(userId),
                product_id: BigInt(productId),
                quantity,
                total,
                status: 'pending'
            }
        });
        
        // 4. 扣减库存
        await tx.products.update({
            where: { id: BigInt(productId) },
            data: { stock: { decrement: quantity } }
        });
        
        return order;
    }, {
        // 事务配置
        maxWait: 5000,         // 等待获取事务的最大时间
        timeout: 10000,        // 事务执行的最大时间
        isolationLevel: 'Serializable'  // 隔离级别(可选)
    });
    // 自动 COMMIT 或 ROLLBACK
}

// 测试
async function test() {
    try {
        const order = await createOrderWithPrisma(1, 1, 1);
        console.log('✅ 下单成功:', order);
    } catch (error) {
        console.error('❌ 下单失败:', error.message);
    } finally {
        await prisma.$disconnect();
    }
}

test();
```

### 7.6 Prisma Studio(可视化管理)

```powershell
# 启动 Prisma Studio
npx prisma studio

# 浏览器自动打开 http://localhost:5555
# 可以可视化查看、编辑、过滤所有表的数据
```

---

## 九、Step 8:对比与最佳实践

### 9.1 原生 SQL vs Prisma

```
┌──────────────────┬─────────────────┬──────────────────┐
│ 维度             │ 原生 pg         │ Prisma           │
├──────────────────┼─────────────────┼──────────────────┤
│ 学习曲线         │ 陡(要会 SQL)  │ 平缓(API 友好)│
│ 类型安全         │ 无              │ 完美(TS)        │
│ 代码量           │ 多              │ 少(约 1/3)       │
│ 性能             │ 最快            │ 几乎一样(优化好)│
│ 复杂查询         │ 灵活            │ 偶尔需要 raw SQL │
│ 学习深度         │ 深(懂底层)    │ 浅(高层抽象)     │
│ 适合场景         │ 高性能/复杂SQL  │ 业务系统/快速开发│
└──────────────────┴─────────────────┴──────────────────┘
```

### 9.2 最佳实践建议

**🎯 实际项目中的推荐方案:**

```javascript
// 80% 的业务代码用 Prisma
const users = await prisma.users.findMany({
    where: { age: { gte: 18 } },
    include: { orders: true }
});

// 20% 的复杂查询用原生 SQL(通过 Prisma 的 $queryRaw)
const stats = await prisma.$queryRaw`
    SELECT 
        DATE_TRUNC('day', created_at) AS day,
        COUNT(*) AS orders,
        SUM(total) AS revenue
    FROM orders
    WHERE created_at >= NOW() - INTERVAL '30 days'
    GROUP BY DATE_TRUNC('day', created_at)
    ORDER BY day DESC
`;
```

### 9.3 生产环境检查清单

```
✅ 使用连接池(不要直接用 Client)
✅ 使用参数化查询(防止 SQL 注入)
✅ 关键操作用事务包裹
✅ 数据库密码用环境变量(不要写在代码里)
✅ 用 .gitignore 排除 .env 文件
✅ 关键查询加索引
✅ 监听连接池错误事件
✅ 优雅关闭(SIGINT 信号处理)
✅ 慢查询日志(statement_timeout)
✅ 健康检查接口
```

### 9.4 性能优化要点

```javascript
// 1. 并行查询(代替串行)
// ❌ 慢:串行执行
const user = await prisma.users.findUnique({ where: { id: 1 } });
const orders = await prisma.orders.findMany({ where: { user_id: 1 } });

// ✅ 快:并行执行
const [user, orders] = await Promise.all([
    prisma.users.findUnique({ where: { id: 1 } }),
    prisma.orders.findMany({ where: { user_id: 1 } })
]);

// 2. 只查询需要的字段
// ❌ 浪费带宽
const users = await prisma.users.findMany();  // 查所有字段

// ✅ 只查需要的
const users = await prisma.users.findMany({
    select: { id: true, username: true }  // 只查这两个字段
});

// 3. 分页避免 OFFSET 大数据集
// ❌ 慢:OFFSET 越大越慢
LIMIT 10 OFFSET 100000

// ✅ 快:基于游标的分页
WHERE id > $last_id ORDER BY id LIMIT 10

// 4. 批量操作代替循环
// ❌ 慢:N 次数据库往返
for (const user of users) {
    await prisma.users.update({ where: { id: user.id }, data: { ... } });
}

// ✅ 快:一次批量操作
await prisma.users.updateMany({
    where: { id: { in: userIds } },
    data: { ... }
});
```

---

## 十、常见错误排查

### 10.1 连接错误

```
❌ Error: ECONNREFUSED 127.0.0.1:5432
原因: PostgreSQL 容器没启动
解决: docker compose up -d 然后 docker compose ps 检查状态

❌ Error: password authentication failed
原因: .env 中密码错误,或与 docker-compose.yml 不一致
解决: 检查两边密码是否相同

❌ Error: database "shop_db" does not exist
原因: 数据库没创建
解决: 检查 docker-compose.yml 的 POSTGRES_DB 配置,
     如果改了配置需要 docker compose down -v 清除数据卷后重启
```

### 10.2 并发错误

```
❌ Error: deadlock detected
原因: 两个事务互相等待对方的锁
解决: 
- 确保事务中加锁顺序一致
- 减小事务范围
- 用 SKIP LOCKED 跳过已锁的行

❌ Error: could not serialize access due to concurrent update
原因: SERIALIZABLE 隔离级别下检测到并发冲突
解决: 应用层捕获并重试该事务
```

### 10.3 BigInt 序列化错误

```javascript
// ❌ JSON.stringify(user) 报错: Do not know how to serialize a BigInt

// ✅ 解决方案 1: 全局 BigInt 转换
BigInt.prototype.toJSON = function() {
    return this.toString();
};

// ✅ 解决方案 2: 在 Prisma 中用 Number
const user = await prisma.users.findFirst();
res.json({
    ...user,
    id: Number(user.id)  // 显式转换
});
```

### 10.4 连接池耗尽

```
❌ Error: timeout exceeded when trying to connect
原因: 连接池满了,新请求等不到连接
解决:
- 检查是否有 client.release() 没调用(连接泄漏!)
- 增加 max 配置
- 检查慢查询,优化 SQL 或加索引
```

---

## 🎯 阶段 3 总结

完成本阶段后,你已经掌握:

```
✅ 用原生 pg 库连接 PostgreSQL
✅ 参数化查询防止 SQL 注入
✅ 连接池配置和调优
✅ 完整的 CRUD 操作
✅ 事务处理(BEGIN/COMMIT/ROLLBACK)
✅ 行锁(FOR UPDATE)避免超卖
✅ 用 Express 搭建 RESTful API
✅ Prisma ORM 的使用
✅ 原生 SQL vs ORM 的取舍
✅ 生产环境最佳实践
```

**下一阶段预告:**

```
🎯 阶段 4: 并发控制深入
- 悲观锁 vs 乐观锁的选型
- 死锁的产生和避免
- 高并发下的"超卖问题"实战
- 用 ab/wrk 工具进行压测

🎯 阶段 5: 备份与恢复
- pg_dump 逻辑备份
- pg_basebackup 物理备份
- WAL 归档 + 时间点恢复(PITR)
- 灾难恢复演练
```

加油!💪
