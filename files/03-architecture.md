# 03-architecture.md · 技术架构

> 三角洲战友匹配平台
> 不写代码,只画框和箭头

---

## 整体架构图

```
┌──────────────────────────────────────────────┐
│         玩家手机/电脑浏览器                  │
│         (微信内 / 浏览器 / 加桌面)            │
└──────────────────┬───────────────────────────┘
                   │ HTTPS
                   ↓
┌──────────────────────────────────────────────┐
│  前端 H5 网页(响应式 + PWA)                 │
│  ┌────────────────────────────────────────┐  │
│  │ ▶ 注册/登录页                          │  │
│  │ ▶ 玩法选择页(跑刀/猛攻)              │  │
│  │ ▶ 匹配推荐页(3 个候选人)             │  │
│  │ ▶ 候选人详情页                         │  │
│  │ ▶ 评分弹窗(强制结算页)               │  │
│  │ ▶ 个人主页                             │  │
│  └────────────────────────────────────────┘  │
└──────────────────┬───────────────────────────┘
                   │ REST API (JSON)
                   ↓
┌──────────────────────────────────────────────┐
│  后端 Node.js + Express                      │
│  ┌────────────────────────────────────────┐  │
│  │ ▶ JWT 用户认证中间件                   │  │
│  │ ▶ /api/auth/*      认证接口            │  │
│  │ ▶ /api/user/*      用户管理            │  │
│  │ ▶ /api/match/*     匹配推荐            │  │
│  │ ▶ /api/rating/*    评分提交            │  │
│  │ ▶ 错误处理 / 日志中间件                │  │
│  └────────────────────────────────────────┘  │
└──────────────────┬───────────────────────────┘
                   │ SQL
                   ↓
┌──────────────────────────────────────────────┐
│  PostgreSQL 数据库                           │
│  ┌────────────────────────────────────────┐  │
│  │ ▶ users    用户表                      │  │
│  │ ▶ matches  匹配记录表                  │  │
│  │ ▶ ratings  评分记录表                  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘

部署: 阿里云轻量服务器(¥18/月) + 域名(¥30/年)
```

---

## 技术栈选择

| 层级 | 技术 | 选择理由 | 学习成本 |
|---|---|---|---|
| 前端框架 | HTML + CSS + JS | 阶段 3 学过,复用职位看板 | 0 |
| 前端样式 | 响应式 + PWA | 手机适配 + 加桌面像 App | 半天 |
| 后端运行时 | Node.js | 阶段 3 学过 | 0 |
| 后端框架 | Express | 阶段 3 学过 | 0 |
| 数据库 | PostgreSQL | 阶段 1-5 学过 | 0 |
| ORM | Prisma | 阶段 3 学过 | 0 |
| 用户认证 | JWT | 标准方案,1 天学会 | 1 天 |
| 部署 | 阿里云轻量服务器 | 个人最便宜 | 1 天 |
| 容器化 | Docker | 阶段 1 学过 | 0 |
| 版本管理 | Git + GitHub | 已熟练 | 0 |

**总体技术学习成本:约 2-3 天**(主要是 JWT 和部署)

---

## 数据库设计

### 表 1: users(用户表)

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PRIMARY KEY | 用户 ID |
| username | VARCHAR(50) UNIQUE NOT NULL | 用户名 |
| email | VARCHAR(100) UNIQUE | 邮箱(可选) |
| phone | VARCHAR(20) UNIQUE | 手机号(可选) |
| password_hash | VARCHAR(255) NOT NULL | 密码哈希(bcrypt) |
| play_style | VARCHAR(20) | 跑刀 / 猛攻 |
| game_id | VARCHAR(50) | 三角洲游戏 ID |
| game_rank | VARCHAR(30) | 段位(黑鹰等) |
| credit_score | DECIMAL(3,2) DEFAULT 5.00 | 信用分(满分 5) |
| total_ratings | INT DEFAULT 0 | 总评分次数 |
| created_at | TIMESTAMPTZ DEFAULT NOW() | 注册时间 |
| updated_at | TIMESTAMPTZ DEFAULT NOW() | 更新时间 |

**索引:**
- `idx_users_play_style` ON play_style(匹配查询用)
- `idx_users_credit_score` ON credit_score DESC(信用分排序用)

---

### 表 2: matches(匹配记录表)

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PRIMARY KEY | 匹配 ID |
| user_a_id | BIGINT NOT NULL | 发起方用户 ID |
| user_b_id | BIGINT NOT NULL | 匹配对象 ID |
| status | VARCHAR(20) | pending / accepted / played / rated |
| matched_at | TIMESTAMPTZ DEFAULT NOW() | 匹配时间 |
| accepted_at | TIMESTAMPTZ | 双方接受时间 |
| played_at | TIMESTAMPTZ | 实际游戏时间 |

**外键:**
- user_a_id → users.id
- user_b_id → users.id

**索引:**
- `idx_matches_user_a` ON user_a_id
- `idx_matches_user_b` ON user_b_id
- `idx_matches_status` ON status

---

### 表 3: ratings(评分表)

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGSERIAL PRIMARY KEY | 评分 ID |
| match_id | BIGINT NOT NULL | 关联哪局 |
| rater_id | BIGINT NOT NULL | 谁打的分 |
| rated_id | BIGINT NOT NULL | 打给谁 |
| rating | INT NOT NULL CHECK (rating BETWEEN 1 AND 5) | 1-5 星 |
| tags | TEXT[] | 标签数组(开朗/沉稳/战术派等) |
| created_at | TIMESTAMPTZ DEFAULT NOW() | 评分时间 |

**外键:**
- match_id → matches.id
- rater_id → users.id
- rated_id → users.id

**唯一约束:**
- UNIQUE (match_id, rater_id, rated_id)(防止重复评分)

**索引:**
- `idx_ratings_rated_id` ON rated_id(查某人的所有评分)

---

## API 接口设计

### 用户相关

```
POST   /api/auth/register   注册账号
       Body: { username, email, password }
       Return: { user, token }

POST   /api/auth/login      登录
       Body: { email, password }
       Return: { user, token }

GET    /api/user/profile    获取个人资料
       Header: Authorization: Bearer <token>

PUT    /api/user/profile    更新资料(改玩法等)
       Body: { play_style, game_id, game_rank }
```

### 匹配相关

```
GET    /api/match/recommend  获取 3 个推荐候选人
       Query: ?play_style=跑刀
       Return: { candidates: [...] }

POST   /api/match/select     选择想一起玩
       Body: { target_user_id }
       Return: { match_id, status }

GET    /api/match/history    历史匹配记录
       Query: ?status=played
```

### 评分相关

```
POST   /api/rating/submit    提交评分
       Body: { match_id, rated_id, rating, tags }
       Return: { success }

GET    /api/rating/pending   待评分列表(强制弹出用)
```

---

## 核心匹配算法(MVP 版本)

```javascript
// MVP 版:同标签随机推荐
function recommendCandidates(userId, playStyle) {
    return db.users
        .where({ play_style: playStyle })
        .whereNot({ id: userId })       // 排除自己
        .whereNotIn(已经匹配过的用户)    // 排除已匹配
        .orderByRaw('RANDOM()')          // 随机
        .limit(3);
}

// V1.5 版:按信用分排序
function recommendCandidates_v15(userId, playStyle) {
    return db.users
        .where({ play_style: playStyle })
        .whereNot({ id: userId })
        .whereNotIn(已经匹配过的用户)
        .orderBy('credit_score', 'desc')  // 信用分高的优先
        .limit(3);
}
```

**MVP 阶段不复杂,5 行代码搞定。**

---

## 部署方案

| 服务 | 服务商 | 配置 | 月费 |
|---|---|---|---|
| 服务器 | 阿里云轻量应用 | 2 核 2G 40G SSD | ¥18/月 |
| 域名 | 阿里云 | .cn 域名 | ¥30/年 |
| 备案 | 工信部 | 个人备案(免费,但 7-15 天) | 免费 |
| SSL 证书 | 阿里云免费版 | DV 证书 | 免费 |
| 数据库 | 服务器自建 PostgreSQL | Docker 部署 | 免费 |
| **总计** | | | **约 ¥250/年** |

### 部署架构

```
用户浏览器
   ↓ HTTPS
[阿里云轻量服务器]
   ├─ Nginx(反向代理 + SSL)
   ├─ Node.js 应用(PM2 守护)
   └─ Docker
       └─ PostgreSQL 容器
```

---

## 不用的第三方服务(MVP 阶段)

| 服务 | 是否用 | 理由 |
|---|---|---|
| 短信验证 | ❌ | 用邮箱注册就行 |
| 微信登录 | ❌ | 用邮箱密码,V2 再加 |
| 推送通知 | ❌ | 网页无法推送,V2 上小程序再说 |
| CDN | ❌ | 用户少,V2 再加 |
| 对象存储 | ❌ | 没有图片上传需求 |
| 邮件服务 | ❌ | V1 不发邮件验证,V2 再加 |

---

## 安全考虑(MVP 最小集)

| 风险 | 应对 |
|---|---|
| 密码明文存储 | bcrypt 加密 |
| SQL 注入 | Prisma ORM 参数化 |
| XSS 攻击 | 前端转义用户输入 |
| 暴力登录 | rate limit(每分钟最多 5 次) |
| Token 泄漏 | JWT 24 小时过期 |
| HTTPS | 部署 SSL |

---

## V2 才考虑的事

```
V2 路线图(MVP 验证成功后):

第一波(用户 0 → 100):
├─ PWA 完善
├─ 邀请裂变机制
└─ 用户运营(贴吧投放)

第二波(用户 100 → 1000):
├─ 微信小程序开发
├─ 互选机制
└─ 作风测试 5 题

第三波(用户 1000+):
├─ 战队系统
├─ 战绩同步(对接游戏 API)
└─ 商业化探索
```

---

## 关键决策记录

| 决策 | 选择 | Critic 建议 | 最终方案 |
|---|---|---|---|
| 前端形态 | 小程序 vs 网页 | 网页 | ✅ 网页 |
| 前端框架 | 原生 vs React | 原生 | ✅ 原生 HTML+CSS+JS |
| 匹配算法 | 随机 vs 算法 | 随机 | ✅ 随机(V1.5 改算法) |
| 评分机制 | 自愿 vs 强制 | 强制 | ✅ 强制弹出 |
| 是否做开麦 | 是 vs 否 | 否 | ✅ 不做(用游戏自带) |

---

_文档版本: v1.0_
_定稿日期: 2026-05-27_
