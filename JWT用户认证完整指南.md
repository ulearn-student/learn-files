# JWT 用户认证完整指南

> 从零搭建一套生产级的用户认证系统:密码加密、注册登录、Token 验证、接口保护。所有代码带详细中文注释,基于你的 node-app 项目实战。

---

## 📚 目录

- [一、为什么需要 JWT](#一为什么需要-jwt)
- [二、核心概念扫盲](#二核心概念扫盲)
- [三、Step 1:安装依赖](#三step-1安装依赖)
- [四、Step 2:密码为什么不能明文存](#四step-2密码为什么不能明文存)
- [五、Step 3:JWT 工具模块](#五step-3jwt-工具模块)
- [六、Step 4:认证中间件(门卫)](#六step-4认证中间件门卫)
- [七、Step 5:认证 API 服务器](#七step-5认证-api-服务器)
- [八、Step 6:用 Postman 测试](#八step-6用-postman-测试)
- [九、踩坑实录:中文用户名 bug](#九踩坑实录中文用户名-bug)
- [十、完整流程复盘](#十完整流程复盘)
- [十一、最佳实践与安全要点](#十一最佳实践与安全要点)

---

## 一、为什么需要 JWT

### 1.1 没有认证的接口有多危险

回想最早做的 API,有个致命问题:

```javascript
// 这些接口谁都能调,完全没有"你是谁"的检查
app.get('/api/users', ...)         // 任何人都能看所有用户
app.delete('/api/users/:id', ...)  // 任何人都能删除用户!!!
```

任何知道你接口地址的人,都能删库、改数据。真实项目里这是灾难。

### 1.2 JWT 是什么(一句话)

> JWT 是用户登录后,服务器发给用户的一张"电子身份证"。

用游乐园类比:

```
1. 买票              → 登录(输入账号密码)
2. 拿到手环           → 服务器发 JWT
3. 玩项目刷手环        → 每次请求带上 JWT
4. 工作人员看手环放行   → 服务器验证 JWT,确认是你
```

手环 = JWT。有了它,玩每个项目不用重新买票(每次请求不用重新登录)。

---

## 二、核心概念扫盲

### 2.1 JWT 的三段式结构

JWT 是一串用两个点分隔的字符串:

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiI3In0.dQw4w9abc123...
└──── 头部 ────┘ └───── 数据 ─────┘ └── 签名 ──┘
```

| 部分 | 英文名 | 内容 | 能否解开看 |
|---|---|---|---|
| 头部 | Header | 用什么加密算法 | 能 |
| 数据 | Payload | 用户信息,如 `{userId: 7}` | 能(所以别存密码!) |
| 签名 | Signature | 防伪标记,用密钥生成 | 不能伪造 |

**关键理解:** 数据段能被任何人解开看,所以**绝不能放密码、敏感信息**。但签名段别人伪造不了,因为没有你服务器的密钥。

### 2.2 完整工作流程

```
① 用户登录
   前端 → 账号密码 → 后端

② 后端验证
   核对密码正确 → 生成 JWT → 返回给前端

③ 前端保存
   把 JWT 存起来(浏览器里)

④ 后续请求
   每次调 API,请求头带上 JWT:
   Authorization: Bearer eyJhbGci...

⑤ 后端验证
   检查 JWT 有效 → 放行;无效 → 拒绝
```

---

## 三、Step 1:安装依赖

```bash
cd node-app

npm install jsonwebtoken bcrypt
```

```
jsonwebtoken : 生成和验证 JWT
bcrypt       : 密码加密(绝不能明文存密码!)
```

---

## 四、Step 2:密码为什么不能明文存

### 4.1 铁律

```
❌ 绝对禁止: 数据库存明文密码
   password: "123456"
   → 数据库一旦泄露,所有用户密码暴露
   → 用户在其他网站的同款密码也跟着遭殃

✅ 正确做法: 存加密后的哈希值
   password_hash: "$2b$10$N9qo8uLOickgx2ZMRZoMye..."
   → 这串东西无法反推出原密码
```

### 4.2 bcrypt 工作原理

```
注册时:
  "123456" → bcrypt 加密 → "$2b$10$N9qo..." → 存数据库

登录时:
  用户输入 "123456" → bcrypt 对比 → 和数据库的哈希匹配吗?
  匹配   → 登录成功
  不匹配 → 密码错误
```

**关键点:** bcrypt 是**单向加密**,只能"加密对比",不能"解密还原"。即使黑客拿到哈希值,也算不出原密码。

---

## 五、Step 3:JWT 工具模块

创建 `utils-auth.js`,封装四个核心函数:

```javascript
// 文件: utils-auth.js
// JWT 认证工具模块 - 封装密码加密和 token 生成/验证

const jwt = require('jsonwebtoken');   // JWT 库
const bcrypt = require('bcrypt');       // 密码加密库

// ===== 密钥配置 =====
// 这个密钥用来给 JWT 签名,泄露了别人就能伪造 token
// ⚠️ 生产环境必须放在 .env 里,不能写死在代码中
const JWT_SECRET = 'delta-buddy-secret-key-2026';

// Token 有效期(7天后用户需要重新登录)
const JWT_EXPIRES = '7d';

// ===== 1. 密码加密 =====
// 注册时调用,把明文密码变成无法反推的哈希值
async function hashPassword(plainPassword) {
    // saltRounds = 10: 加密强度(越高越安全但越慢,10 是推荐值)
    const saltRounds = 10;
    const hash = await bcrypt.hash(plainPassword, saltRounds);
    return hash;
    // 返回类似: "$2b$10$N9qo8uLOickgx2ZMRZoMye..."
}

// ===== 2. 密码验证 =====
// 登录时调用,对比用户输入的密码和数据库里的哈希值
async function verifyPassword(plainPassword, hashedPassword) {
    // bcrypt 会自动处理对比逻辑,返回 true/false
    const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
    return isMatch;
}

// ===== 3. 生成 JWT =====
// 登录成功后调用,给用户发一张"电子身份证"
function generateToken(payload) {
    // payload: 要存进 token 的用户信息
    // ⚠️ 绝不要把密码放进 payload!因为 payload 可以被解开看
    // ⚠️ 也不要放中文(会导致 HTTP 头编码错误,见第九章)
    const token = jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRES });
    return token;
}

// ===== 4. 验证 JWT =====
// 每次请求受保护接口时调用,检查 token 是否有效
function verifyToken(token) {
    try {
        // 验证签名 + 是否过期,通过则返回 payload
        const decoded = jwt.verify(token, JWT_SECRET);
        return { valid: true, data: decoded };
    } catch (error) {
        // token 无效、被篡改、过期都会进这里
        return { valid: false, error: error.message };
    }
}

// 导出所有函数
module.exports = {
    hashPassword,
    verifyPassword,
    generateToken,
    verifyToken
};
```

### 逐函数解释

| 函数 | 什么时候用 | 输入 | 输出 |
|---|---|---|---|
| `hashPassword` | 注册时 | 明文密码 | 加密哈希 |
| `verifyPassword` | 登录时 | 明文密码 + 哈希 | true/false |
| `generateToken` | 登录成功后 | 用户信息 | JWT 字符串 |
| `verifyToken` | 访问受保护接口时 | JWT 字符串 | 是否有效 + 用户信息 |

---

## 六、Step 4:认证中间件(门卫)

中间件就是"门卫",在请求到达接口前先检查 token。

创建 `middleware-auth.js`:

```javascript
// 文件: middleware-auth.js
// 认证中间件 - 保护需要登录才能访问的接口

const { verifyToken } = require('./utils-auth');

// 这个函数会在受保护的接口前执行,检查请求是否带了有效 token
function authMiddleware(req, res, next) {
    // 1. 从请求头取出 token
    // 标准格式: Authorization: Bearer eyJhbGci...
    const authHeader = req.headers.authorization;
    
    if (!authHeader) {
        return res.status(401).json({ error: '请先登录(缺少认证信息)' });
    }
    
    // 2. 分离出 token(去掉前面的 "Bearer ")
    // "Bearer eyJhbGci..." → split 后取 [1] → "eyJhbGci..."
    const token = authHeader.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'token 格式错误' });
    }
    
    // 3. 验证 token
    const result = verifyToken(token);
    
    if (!result.valid) {
        return res.status(401).json({ error: '登录已过期,请重新登录' });
    }
    
    // 4. 验证通过!把用户信息挂到 req 上,后续接口可以用
    req.user = result.data;
    // 现在接口里可以通过 req.user.userId 知道是谁在操作
    
    // 5. 放行,继续执行下一步
    next();
}

module.exports = authMiddleware;
```

### 中间件的核心机制

```
请求进来
    ↓
authMiddleware 拦截(门卫检查)
    ↓
有 token 且有效?
   ├─ 是 → next() 放行 → 进入真正的接口处理
   └─ 否 → 返回 401 错误 → 请求被拦下,接口代码根本不执行
```

`next()` 是关键:调用它 = 放行;不调用 = 拦截。

---

## 七、Step 5:认证 API 服务器

创建 `auth-server.js`,包含注册、登录、受保护接口三个核心功能:

```javascript
// 文件: auth-server.js
// 带 JWT 认证的完整服务器

const express = require('express');
const pool = require('./db');
const { hashPassword, verifyPassword, generateToken } = require('./utils-auth');
const authMiddleware = require('./middleware-auth');

const app = express();
const PORT = 3001;  // 用 3001 避免和之前的 server.js 冲突

app.use(express.json());

// ============ 1. 注册接口(公开,不需要登录)============
app.post('/api/auth/register', async (req, res) => {
    try {
        const { username, email, password } = req.body;
        
        // 基础校验
        if (!username || !email || !password) {
            return res.status(400).json({ error: '用户名、邮箱、密码都不能为空' });
        }
        if (password.length < 6) {
            return res.status(400).json({ error: '密码至少 6 位' });
        }
        
        // 关键!把明文密码加密成哈希值
        const passwordHash = await hashPassword(password);
        
        // 存入数据库(存的是加密后的哈希,不是明文)
        const result = await pool.query(`
            INSERT INTO users (username, email, password_hash)
            VALUES ($1, $2, $3)
            RETURNING id, username, email, created_at
        `, [username, email, passwordHash]);
        
        const user = result.rows[0];
        
        // 注册成功后直接生成 token,让用户免去再登录一次
        // ⚠️ 注意:payload 里只放 userId,不放 username(避免中文编码问题)
        const token = generateToken({ userId: user.id.toString() });
        
        res.status(201).json({
            message: '注册成功',
            user: { id: user.id.toString(), username: user.username, email: user.email },
            token  // 把 token 返回给前端
        });
        
    } catch (error) {
        if (error.code === '23505') {  // 唯一约束冲突(邮箱/用户名重复)
            return res.status(409).json({ error: '用户名或邮箱已被注册' });
        }
        res.status(500).json({ error: error.message });
    }
});

// ============ 2. 登录接口(公开)============
app.post('/api/auth/login', async (req, res) => {
    try {
        const { email, password } = req.body;
        
        if (!email || !password) {
            return res.status(400).json({ error: '邮箱和密码不能为空' });
        }
        
        // 根据邮箱找用户
        const result = await pool.query(
            'SELECT id, username, email, password_hash FROM users WHERE email = $1',
            [email]
        );
        
        if (result.rows.length === 0) {
            // 安全要点:不要提示"用户不存在",统一说"账号或密码错误"
            // 否则黑客能用这个接口探测哪些邮箱注册过
            return res.status(401).json({ error: '账号或密码错误' });
        }
        
        const user = result.rows[0];
        
        // 关键!用 bcrypt 对比密码
        const isMatch = await verifyPassword(password, user.password_hash);
        
        if (!isMatch) {
            return res.status(401).json({ error: '账号或密码错误' });
        }
        
        // 密码正确,生成 token
        const token = generateToken({ userId: user.id.toString() });
        
        res.json({
            message: '登录成功',
            user: { id: user.id.toString(), username: user.username, email: user.email },
            token
        });
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// ============ 3. 受保护接口(需要登录)============
// 注意第二个参数 authMiddleware,这就是"门卫"
app.get('/api/auth/me', authMiddleware, async (req, res) => {
    try {
        // req.user 是中间件挂上去的,包含 token 里的用户信息
        const userId = req.user.userId;
        
        const result = await pool.query(
            'SELECT id, username, email, created_at FROM users WHERE id = $1',
            [userId]
        );
        
        if (result.rows.length === 0) {
            return res.status(404).json({ error: '用户不存在' });
        }
        
        const user = result.rows[0];
        res.json({
            id: user.id.toString(),
            username: user.username,
            email: user.email,
            created_at: user.created_at
        });
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// ============ 4. 对比:不受保护的接口 ============
app.get('/api/public/info', (req, res) => {
    res.json({ message: '这是公开信息,谁都能看' });
});

app.listen(PORT, () => {
    console.log(`🚀 认证服务器启动: http://localhost:${PORT}`);
    console.log(`📝 注册: POST /api/auth/register`);
    console.log(`🔑 登录: POST /api/auth/login`);
    console.log(`🔒 受保护: GET /api/auth/me (需要 token)`);
});
```

### 受保护接口的关键写法

```javascript
// 公开接口:只有一个处理函数
app.get('/api/public/info', (req, res) => {...})

// 受保护接口:中间件 + 处理函数
app.get('/api/auth/me', authMiddleware, (req, res) => {...})
//                       ^^^^^^^^^^^^^^
//                       门卫先检查,通过才执行后面的处理函数
```

启动:

```bash
node auth-server.js
```

---

## 八、Step 6:用 Postman 测试

### 8.1 测试注册

```
方法:   POST
地址:   http://localhost:3001/api/auth/register
Body:   raw → JSON
内容:
{
  "username": "jack",
  "email": "jack@test.com",
  "password": "123456"
}
点 Send
```

成功返回:

```json
{
  "message": "注册成功",
  "user": { "id": "7", "username": "jack", "email": "jack@test.com" },
  "token": "eyJhbGci..."
}
```

### 8.2 测试登录

```
方法:   POST
地址:   http://localhost:3001/api/auth/login
Body:   raw → JSON
内容:
{
  "email": "jack@test.com",
  "password": "123456"
}
点 Send
```

### 8.3 测试受保护接口(重点)

```
方法:          GET
地址:          http://localhost:3001/api/auth/me
Authorization: 标签 → Type 选 "Bearer Token"
Token:         粘贴登录返回的 token
点 Send
```

成功返回用户信息 = JWT 验证通过!

### 8.4 测试拦截(验证门卫)

```
同样访问 /api/auth/me
但 Authorization 改成 "No Auth"
点 Send
→ 返回 {"error":"请先登录(缺少认证信息)"}
→ 证明门卫起作用了
```

---

## 九、踩坑实录:中文用户名 bug

### 9.1 现象

用中文用户名(如 "postman用户")注册后,拿 token 访问受保护接口,Postman 报错:

```
Invalid character in header content ["Authorization"]
```

### 9.2 根因分析

```
用户名 = "postman用户"(带中文)
    ↓
生成 token 时,中文被编码进 token 的 payload 段
    ↓
token 里出现了 HTTP 请求头不允许的字符
    ↓
Postman 想把 token 放进 Authorization 头
    ↓
HTTP 协议规定:请求头只能用 ASCII 字符
    ↓
中文编码后的字符不合法 → 请求根本发不出去
```

### 9.3 解决方案

**核心原则:token 的 payload 里只放最少的必要信息(通常就是 userId)。**

```javascript
// ❌ 旧做法:payload 里塞 username(可能是中文)
const token = generateToken({ 
    userId: user.id.toString(), 
    username: user.username   // 中文会出问题!
});

// ✅ 新做法:只放 userId(纯数字,绝对安全)
const token = generateToken({ 
    userId: user.id.toString() 
});
```

注册接口和登录接口里**各有一处**生成 token 的代码,两处都要改。

### 9.4 为什么这是最佳实践

```
token 应该只放【最少的必要信息】:
- 通常就是 userId(纯数字,无编码风险)
- 需要用户名等信息时,用 userId 去数据库查

好处:
1. 避免编码问题(纯 ASCII)
2. 更安全(token 泄露也只暴露一个 ID)
3. 信息始终最新(从数据库实时查,不会用到过期的旧数据)
```

---

## 十、完整流程复盘

### 10.1 注册流程

```
前端发送 {username, email, password}
    ↓
后端:bcrypt 加密密码
    ↓
存数据库(存哈希,不存明文)
    ↓
生成 JWT(只放 userId)
    ↓
返回 {user, token}
```

### 10.2 登录流程

```
前端发送 {email, password}
    ↓
后端:根据 email 查用户
    ↓
bcrypt 对比密码
    ├─ 不匹配 → 返回"账号或密码错误"
    └─ 匹配   → 生成 JWT → 返回 {user, token}
```

### 10.3 访问受保护接口流程

```
前端请求,头里带 Authorization: Bearer <token>
    ↓
authMiddleware 拦截
    ↓
取出 token → verifyToken 验证
    ├─ 无效 → 返回 401
    └─ 有效 → 解出 userId → 挂到 req.user → next() 放行
    ↓
接口处理函数:用 req.user.userId 查数据库
    ↓
返回数据
```

---

## 十一、最佳实践与安全要点

### 11.1 安全铁律

```
✅ 密码必须 bcrypt 加密,绝不明文存储
✅ JWT_SECRET 密钥必须保密,放 .env 不放代码
✅ payload 里绝不放密码、敏感信息
✅ payload 里只放 userId,不放中文/复杂数据
✅ 登录失败统一提示"账号或密码错误"(防探测)
✅ 生产环境必须配 HTTPS(否则 token 会被截获)
✅ token 设置合理的过期时间(如 7 天)
```

### 11.2 HTTP 状态码规范

| 状态码 | 含义 | 什么时候用 |
|---|---|---|
| 200 | 成功 | 登录成功、查询成功 |
| 201 | 创建成功 | 注册成功 |
| 400 | 请求参数错误 | 缺少必填字段 |
| 401 | 未认证 | 没登录、token 失效 |
| 403 | 无权限 | 登录了但权限不够 |
| 409 | 冲突 | 邮箱/用户名已存在 |
| 500 | 服务器错误 | 代码异常、数据库错误 |

### 11.3 把密钥移到 .env(生产级做法)

```bash
# .env 文件里加一行
JWT_SECRET=delta-buddy-secret-key-2026
```

```javascript
// utils-auth.js 里改成从环境变量读取
require('dotenv').config();
const JWT_SECRET = process.env.JWT_SECRET;
```

这样密钥就不会暴露在代码仓库里(配合 .gitignore 排除 .env)。

### 11.4 常见错误处理

```
❌ "Invalid character in header content"
   原因: token 里有中文/非法字符
   解决: payload 只放 userId,不放中文

❌ "登录已过期,请重新登录"
   原因: token 无效/过期/被篡改/复制不完整
   解决: 重新登录拿新 token,确保复制完整

❌ "jwt malformed"
   原因: token 格式不对(不是三段式)
   解决: 检查是否完整复制了 token

❌ "jwt expired"
   原因: token 真的过期了
   解决: 重新登录

❌ "invalid signature"
   原因: 生成和验证用了不同的 JWT_SECRET
   解决: 确保两处密钥一致
```

---

## 🏆 你掌握了什么

```
✅ JWT 的原理(电子身份证 + 三段式结构)
✅ bcrypt 密码加密(安全铁律)
✅ 完整的注册/登录流程
✅ 认证中间件(保护接口的门卫机制)
✅ 用 Postman 测试 API
✅ 排查并修复了中文编码 bug(真实工程问题)
✅ HTTP 状态码规范
✅ 认证系统的安全最佳实践
```

JWT 认证是全栈开发的**核心必备技能**,几乎每个有登录功能的项目都用它。

---

## 🚀 下一步建议

```
A. 把 JWT 接到真实前端登录页面
   → 做一个能点击登录的网页
   → 学前端怎么保存和使用 token(localStorage)

B. 学 React(全栈最大短板)
   → 现代前端框架,招聘需求最大

C. 把认证系统整合进三角洲搭子项目
   → 用真实项目串联所学,做成简历作品
```
