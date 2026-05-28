# Postman 使用完整指南

> Postman 是后端开发必备的 API 测试工具,可视化界面,比命令行敲 curl 舒服 100 倍。本文档基于你的 node-app 认证项目实战讲解。

---

## 📚 目录

- [一、Postman 是什么,为什么用它](#一postman-是什么为什么用它)
- [二、界面认识](#二界面认识)
- [三、发送第一个请求](#三发送第一个请求)
- [四、HTTP 方法详解](#四http-方法详解)
- [五、Body:发送数据](#五body发送数据)
- [六、Authorization:带 Token 认证](#六authorization带-token-认证)
- [七、Headers:请求头](#七headers请求头)
- [八、看懂返回结果](#八看懂返回结果)
- [九、Collection:管理你的接口](#九collection管理你的接口)
- [十、环境变量:告别复制粘贴 token](#十环境变量告别复制粘贴-token)
- [十一、常见问题排查](#十一常见问题排查)

---

## 一、Postman 是什么,为什么用它

### 1.1 一句话定义

> Postman 是一个测试 API 接口的工具。你不用写前端页面,就能直接调用后端接口,看返回结果。

### 1.2 对比命令行 curl

```
用 PowerShell curl:
curl -Method POST http://localhost:3001/api/auth/login `
  -ContentType "application/json" `
  -Body '{"email":"jack@test.com","password":"123456"}'
→ 命令长、易写错、看返回结果还要 $res.Content

用 Postman:
点几下鼠标,填几个框,点 Send
→ 直观、不易错、返回结果自动格式化高亮
```

### 1.3 后端开发为什么离不开它

```
✅ 写完一个 API,先用 Postman 测通,再接前端
✅ 调试接口:哪里返回错了,一眼看出
✅ 保存接口:整理成 Collection,团队共享
✅ 自动化测试:可以写测试脚本批量验证
```

---

## 二、界面认识

打开 Postman,主要区域:

```
┌─────────────────────────────────────────────────────┐
│  [顶部标签页] Overview | 请求1 | 请求2 | +           │ ← 多个请求切换
├──────────────┬──────────────────────────────────────┤
│              │  [方法▼] [地址栏............] [Send]   │ ← 核心操作区
│  左侧         │  ─────────────────────────────────   │
│  Collections │  Params | Auth | Headers | Body | ... │ ← 请求配置标签
│  (接口列表)   │  ─────────────────────────────────   │
│              │  [配置内容区]                          │
│              │  ─────────────────────────────────   │
│              │  [返回结果区] Body | Headers | ...     │ ← 看结果的地方
└──────────────┴──────────────────────────────────────┘
```

记住三个最常用的区域:
1. **地址栏 + 方法 + Send** —— 发请求
2. **Body / Authorization 标签** —— 配置请求内容
3. **下方返回结果区** —— 看后端返回了什么

---

## 三、发送第一个请求

### 3.1 新建请求

```
方式 A: 点顶部标签栏的 "+" 号 → 新建空白请求
方式 B: 左侧 Collection 上右键 → Add Request
```

### 3.2 一个最简单的 GET 请求

测试一下你的公开接口(不需要认证):

```
1. 方法选: GET(默认就是)
2. 地址栏输入: http://localhost:3001/api/public/info
3. 点蓝色 "Send"
4. 下方看到返回:
   {
     "message": "这是公开信息,谁都能看"
   }
```

⚠️ 前提:`auth-server.js` 必须在运行。

---

## 四、HTTP 方法详解

地址栏左边的下拉框,选择请求方法:

| 方法 | 用途 | 例子 |
|---|---|---|
| GET | 获取数据 | 查询用户列表、获取个人信息 |
| POST | 新建/提交 | 注册、登录、下单 |
| PUT | 完整更新 | 更新整个用户资料 |
| PATCH | 部分更新 | 只改用户的某个字段 |
| DELETE | 删除 | 删除用户、删除订单 |

```
记忆口诀:
查 → GET
增 → POST
改 → PUT / PATCH
删 → DELETE
```

你的认证接口对应:

```
注册:   POST /api/auth/register
登录:   POST /api/auth/login
查资料: GET  /api/auth/me
```

---

## 五、Body:发送数据

当你要给后端**发送数据**(比如注册时的账号密码),就用 Body。

### 5.1 操作步骤

```
1. 方法选 POST
2. 点地址栏下面的 "Body" 标签
3. 选 "raw"(原始数据)
4. 右边格式下拉框选 "JSON"
5. 在输入框里写 JSON 数据
```

### 5.2 Body 的几种类型

| 类型 | 用途 |
|---|---|
| none | 不发送数据(GET 常用) |
| form-data | 表单数据,可传文件 |
| x-www-form-urlencoded | 传统表单 |
| **raw** | **原始数据(最常用,配合 JSON)** |
| binary | 二进制文件 |

90% 的情况用 **raw + JSON**。

### 5.3 实战:注册接口

```
方法: POST
地址: http://localhost:3001/api/auth/register
Body → raw → JSON:
{
  "username": "jack",
  "email": "jack@test.com",
  "password": "123456"
}
Send
```

⚠️ **JSON 格式要点:**

```
✅ 正确:
{
  "username": "jack",     ← key 必须用双引号
  "password": "123456"    ← 最后一项后面不要逗号
}

❌ 错误:
{
  username: "jack",       ← key 没引号(这是 JS 写法,JSON 不行)
  "password": "123456",   ← 最后一项多了逗号
}
```

---

## 六、Authorization:带 Token 认证

访问**受保护接口**时,需要带上登录拿到的 token。

### 6.1 操作步骤

```
1. 点地址栏下面的 "Authorization" 标签
2. Auth Type 下拉框选 "Bearer Token"
3. 右边出现 Token 输入框
4. 粘贴你登录/注册返回的 token
5. Send
```

### 6.2 实战:访问 /me 接口

```
第一步:先登录拿 token
   POST http://localhost:3001/api/auth/login
   Body → raw → JSON:
   { "email": "jack@test.com", "password": "123456" }
   Send → 复制返回的 token

第二步:用 token 访问受保护接口
   GET http://localhost:3001/api/auth/me
   Authorization → Bearer Token → 粘贴 token
   Send → 返回你的用户信息
```

### 6.3 Bearer Token 是什么

```
Postman 帮你做的事:
   你粘贴 token 到 Bearer Token 框
        ↓
   Postman 自动在请求头加上:
   Authorization: Bearer <你的token>
        ↓
   后端中间件就能取到 token 验证

如果不用这个功能,你得自己去 Headers 手动加这一行
所以 Bearer Token 选项是个便利功能
```

### 6.4 复制 token 的技巧

```
在返回结果里,token 是一长串字符
双击 token 那串字符 → Postman 自动选中整个 token(不带引号)
Ctrl + C 复制

⚠️ 千万别手动框选,容易少复制或多带引号
```

---

## 七、Headers:请求头

Headers 是请求的"附加说明",告诉服务器一些元信息。

### 7.1 常见 Headers

| Header | 作用 |
|---|---|
| Content-Type | 告诉服务器发的是什么格式(application/json) |
| Authorization | 认证信息(Bearer token) |
| User-Agent | 客户端信息 |

### 7.2 注意:Postman 经常自动帮你加

```
当你用 Body → raw → JSON 时:
   Postman 自动加 Content-Type: application/json

当你用 Authorization → Bearer Token 时:
   Postman 自动加 Authorization: Bearer xxx

所以大部分时候你不用手动管 Headers
```

### 7.3 一个常见坑

```
⚠️ 如果你在 Headers 里手动加了 Authorization
   又在 Authorization 标签里设了 Bearer Token
   → 两者可能冲突,Headers 里的会覆盖

解决:二选一,别同时设。推荐用 Authorization 标签
```

---

## 八、看懂返回结果

发送请求后,下方显示返回结果。

### 8.1 结果区的几个标签

```
Body      ← 返回的数据(最常看)
Cookies   ← 返回的 cookie
Headers   ← 返回的响应头
Test Results ← 测试脚本结果
```

### 8.2 状态码(最重要)

返回结果区会显示一个**状态码**,比如 `200 OK` 或 `401 Unauthorized`:

| 状态码 | 颜色 | 含义 |
|---|---|---|
| 200 | 绿色 | 成功 |
| 201 | 绿色 | 创建成功(注册) |
| 400 | 橙/红 | 请求参数错误 |
| 401 | 橙/红 | 未认证(没登录/token失效) |
| 404 | 橙/红 | 接口不存在 |
| 409 | 橙/红 | 冲突(邮箱已注册) |
| 500 | 红色 | 服务器错误 |

```
看到 2xx → 成功
看到 4xx → 你这边错了(参数、认证)
看到 5xx → 服务器代码错了
```

### 8.3 实战例子

```
注册成功:
状态码 201 Created
Body:
{
  "message": "注册成功",
  "user": {...},
  "token": "eyJhbGci..."
}

token 失效:
状态码 401 Unauthorized
Body:
{
  "error": "登录已过期,请重新登录"
}

邮箱已注册:
状态码 409 Conflict
Body:
{
  "error": "用户名或邮箱已被注册"
}
```

---

## 九、Collection:管理你的接口

Collection(集合)= 把相关接口归类保存,方便复用。

### 9.1 为什么用 Collection

```
没有 Collection:
   每次测试都要重新输地址、填 Body
   关掉 Postman 就没了

用 Collection:
   把"注册""登录""获取信息"保存成一组
   下次直接点开就能用,不用重输
```

### 9.2 创建 Collection

```
1. 左侧 Collections 区域,点 "+" 或 "New Collection"
2. 命名,比如 "三角洲认证接口"
3. 在 Collection 上右键 → Add Request
4. 配置好请求后,点 Save 保存进 Collection
```

### 9.3 建议的接口组织方式

```
📁 三角洲认证接口 (Collection)
├── 注册     POST /api/auth/register
├── 登录     POST /api/auth/login
├── 获取信息  GET  /api/auth/me
└── 公开信息  GET  /api/public/info
```

保存后,这些接口永久存在,随时点开就能测。

---

## 十、环境变量:告别复制粘贴 token

这是 Postman 的**高级技巧**,能省去反复复制 token 的麻烦。

### 10.1 痛点

```
每次测受保护接口,都要:
1. 登录拿 token
2. 复制 token
3. 粘贴到 Authorization
   → 麻烦,token 还经常过期要重来
```

### 10.2 解决方案:用变量自动保存 token

**第一步:创建环境变量**

```
1. 右上角 "No Environment" 下拉 → 点 "+" 新建环境
2. 命名 "本地开发"
3. 加一个变量:
   Variable: token
   Value: (留空)
4. 保存,然后在右上角选中"本地开发"环境
```

**第二步:登录后自动把 token 存进变量**

```
在【登录请求】里:
1. 点 "Scripts" 标签(老版本叫 "Tests")
2. 填入这段脚本:
   const data = pm.response.json();
   pm.environment.set("token", data.token);
3. 这样每次登录,token 自动存进变量
```

**第三步:受保护接口用变量代替 token**

```
在【/me 请求】里:
Authorization → Bearer Token
Token 框里填: {{token}}
            ↑ 用双大括号引用变量

这样它会自动用最新的 token,不用手动复制
```

### 10.3 效果

```
配置一次后:
1. 点"登录" Send → token 自动存进变量
2. 点"获取信息" Send → 自动带上最新 token
   → 不用任何复制粘贴!
```

---

## 十一、常见问题排查

### 11.1 连不上服务器

```
❌ Error: connect ECONNREFUSED 127.0.0.1:3001

原因: 服务器没启动
解决: 
cd node-app
node auth-server.js
确认看到 "🚀 认证服务器启动: http://localhost:3001"
```

### 11.2 JSON 格式错误

```
❌ 返回: SyntaxError: Unexpected token

原因: Body 里的 JSON 写错了
检查:
- key 是否用双引号
- 最后一项后面有没有多余逗号
- 大括号是否配对
```

### 11.3 Invalid character in header

```
❌ Invalid character in header content ["Authorization"]

原因: token 里有非法字符(比如中文)
解决: 见 JWT 文档第九章,payload 只放 userId
```

### 11.4 401 未认证

```
❌ 返回 401: "登录已过期,请重新登录"

可能原因:
1. token 没复制完整 → 重新完整复制
2. token 过期了 → 重新登录拿新 token
3. Authorization 没选 Bearer Token → 检查设置
4. token 框里有多余空格/引号 → 清空重填
```

### 11.5 Body 没生效

```
❌ 后端收到的数据是空的

检查:
1. Body 是否选了 "raw"
2. 格式是否选了 "JSON"
3. 方法是否是 POST(GET 不带 Body)
4. 后端是否有 app.use(express.json())
```

---

## 🏆 你掌握了什么

```
✅ Postman 的界面和基本操作
✅ 发送 GET / POST 请求
✅ Body 传 JSON 数据
✅ Authorization 带 Token 认证
✅ 看懂状态码和返回结果
✅ 用 Collection 管理接口
✅ 用环境变量自动管理 token
✅ 常见问题排查
```

---

## 💡 日常工作流总结

```
开发一个新接口的标准流程:

1. 后端写好接口(比如新增一个 POST /api/orders)
2. 启动服务器 node xxx-server.js
3. 打开 Postman
4. 新建请求,选方法、填地址、配 Body
5. Send 测试
6. 看返回结果:
   - 2xx → 接口正常,可以接前端了
   - 4xx/5xx → 看错误信息,回去改代码
7. 测通后保存进 Collection,方便以后复用

→ 这是每个后端工程师每天都在做的事
```

Postman 是你后端开发路上的常用工具,以后写的每个接口都靠它来测试和调试 💪
