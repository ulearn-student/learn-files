# Windows 数据库完整安装指南

> 包含 MySQL、PostgreSQL、Redis 三大数据库 + 免费图形化工具
> 面向 Windows 10/11 用户,新手友好,每一步都有详细说明

---

## 目录

- [安装前准备](#安装前准备)
- [一、MySQL 安装](#一mysql-安装)
- [二、PostgreSQL 安装](#二postgresql-安装)
- [三、Redis 安装](#三redis-安装)
- [四、图形化管理工具](#四图形化管理工具)
- [五、Node.js 连接测试](#五nodejs-连接测试)
- [六、常见问题排查](#六常见问题排查)
- [附录:卸载与重装](#附录卸载与重装)

---

## 安装前准备

### 系统要求
- Windows 10/11(64位)
- 至少 4GB 内存(三个数据库都跑建议 8GB+)
- 至少 10GB 可用磁盘空间

### 安装顺序建议
1. 先装 MySQL(最重要)
2. 再装 PostgreSQL(可选,但建议都装)
3. 最后装 Redis(缓存用)
4. 装图形化工具

### 推荐安装位置
**不要装在 C 盘**!建议统一放在 `D:\database\` 下:
```
D:\database\
├── mysql\
├── postgresql\
├── redis\
└── data\          ← 数据文件统一存放
```

理由:
- C 盘满了会导致系统问题
- 重装系统时数据不丢
- 便于备份管理

---

## 一、MySQL 安装

### 1.1 下载 MySQL Installer

1. 打开浏览器访问:https://dev.mysql.com/downloads/installer/
2. 选择 **Windows (x86, 32-bit), MSI Installer** 的**较大那个**(约 300+ MB,离线完整版,推荐)
3. 点击 Download
4. 下一页会提示登录 Oracle 账号,**直接点最下方的 "No thanks, just start my download"** 跳过登录

> 💡 小提示:Installer 是 32 位的,但安装的 MySQL 本体是 64 位,不用担心。

### 1.2 安装步骤

**步骤1:启动安装程序**

双击下载的 `.msi` 文件,如果弹出 "需要 .NET Framework",先去微软官网下载安装。

**步骤2:选择安装类型**

出现 "Choosing a Setup Type" 界面:
- ❌ Developer Default(全家桶,装一堆用不上的)
- ❌ Server only(只装服务端,没有客户端命令行工具)
- ✅ **Custom**(自定义,推荐!)

**步骤3:选择组件**

在 Custom 界面,展开左侧树形菜单,只勾选这几个:
```
MySQL Servers
  └── MySQL Server 8.0.x - X64    ← 必选,核心数据库

Applications
  └── MySQL Workbench 8.0.x - X64  ← 可选,官方图形工具
  └── MySQL Shell 8.0.x - X64      ← 可选,新版命令行

MySQL Connectors
  └── Connector/J 8.0.x            ← Java连接驱动(用Node.js可不装)
```

选中后点击中间的 **绿色右箭头 →** 把它们移到右侧。

**步骤4:修改安装路径(重要!)**

右侧选中 `MySQL Server 8.0.x`,下方会出现 **Advanced Options**,点击它:
- Install Directory: `D:\database\mysql\`
- Data Directory: `D:\database\data\mysql\`

> ⚠️ 路径不要有中文和空格,否则会出问题!

**步骤5:Installation**

点击 Execute,等待下载和安装完成(几分钟到十几分钟,取决于网速)。

全部 Complete 后点击 Next。

**步骤6:Product Configuration(配置 MySQL)**

点击 Next 进入配置环节。

**Type and Networking 页面:**
- Config Type: **Development Computer**(开发用)
- Connectivity: 保持默认
  - TCP/IP: ✅ 勾选
  - Port: `3306`(默认,记住这个端口)
  - X Protocol Port: `33060`
  - Open Windows Firewall ports: ✅ 勾选(防火墙放行)

**Authentication Method 页面:**
- ✅ **Use Strong Password Encryption (RECOMMENDED)**(推荐,更安全)
- 选这个的话,老版本驱动可能连不上,但 Node.js 的 mysql2 完全支持

**Accounts and Roles 页面(关键!):**

设置 **root 密码**!
- MySQL Root Password: 输入一个你能记住的密码
- Repeat Password: 再输一次

> 🔐 **密码强烈建议记在密码管理器里**,忘了非常麻烦!
> 新手可以先设简单点比如 `Root@123456`,生产环境再换强密码。

下方可以点 **Add User** 创建普通用户(可选):
- User Name: `dev`
- Host: `localhost`
- Role: `DB Admin`
- Password: 设置一个密码

> 💡 实际开发推荐:不要直接用 root,创建普通用户给项目用。

**Windows Service 页面:**
- ✅ Configure MySQL Server as a Windows Service(设为系统服务,开机自启)
- Windows Service Name: `MySQL80`(保持默认)
- ✅ Start the MySQL Server at System Startup(开机自动启动)
- Run Windows Service as: **Standard System Account**

> 💡 如果你不想开机自启(占内存),可以**取消勾选** Start at System Startup,后面手动启动。

**Apply Configuration:**

点击 Execute,等待全部 ✅,然后 Finish。

**步骤7:完成安装**

后面几个页面一路 Next、Finish 即可。最后会自动打开 MySQL Workbench(图形界面)。

### 1.3 验证安装

**方法1:命令行验证**

按 `Win + R` 输入 `cmd` 回车,在命令行输入:

```bash
# 注意:如果提示 'mysql' 不是内部或外部命令,说明环境变量没配,先看 1.4
mysql -u root -p
```

输入刚才设置的密码,看到:
```
Welcome to the MySQL monitor.
mysql>
```

恭喜!MySQL 安装成功。试几条命令:

```sql
-- 查看所有数据库
SHOW DATABASES;

-- 查看当前版本
SELECT VERSION();

-- 退出
EXIT;
```

**方法2:服务验证**

按 `Win + R` 输入 `services.msc` 回车,在服务列表里找到 `MySQL80`,状态应该是 **正在运行**。

### 1.4 配置环境变量(让 mysql 命令全局可用)

如果命令行输入 `mysql` 提示找不到命令:

1. 右键 **此电脑** → **属性** → **高级系统设置** → **环境变量**
2. 在 **系统变量** 中找到 **Path**,双击编辑
3. 点击 **新建**,添加:`D:\database\mysql\bin`
4. 一路点确定,**关闭所有命令行窗口**,重新打开命令行测试

### 1.5 启动/停止 MySQL 服务

**方法1:命令行(管理员身份)**
```bash
# 启动
net start MySQL80

# 停止
net stop MySQL80
```

**方法2:服务管理器**
`Win + R` → `services.msc` → 找到 `MySQL80` → 右键 → 启动/停止

---

## 二、PostgreSQL 安装

PostgreSQL 比 MySQL 功能更强大,安装也更简单。

### 2.1 下载安装包

1. 访问:https://www.postgresql.org/download/windows/
2. 点击 **Download the installer**(EDB 提供)
3. 选择最新稳定版(推荐 16.x 或 17.x)
4. 选择 **Windows x86-64**,下载

### 2.2 安装步骤

**步骤1:启动安装**

双击下载的 `.exe`,以管理员身份运行。

**步骤2:Installation Directory**

```
Installation Directory: D:\database\postgresql\
```

**步骤3:Select Components**

全部勾选(推荐):
- ✅ PostgreSQL Server(数据库本体)
- ✅ pgAdmin 4(官方图形管理工具,免费且强大!)
- ✅ Stack Builder(扩展包管理,可装可不装)
- ✅ Command Line Tools(命令行工具)

**步骤4:Data Directory**

```
D:\database\data\postgresql\
```

**步骤5:Password(关键!)**

设置 **postgres 超级管理员密码**:
- 输入一个你能记住的密码,比如 `Postgres@123456`
- 再次确认

> ⚠️ 这个密码就是 postgres 用户的密码,务必记牢!

**步骤6:Port**

```
Port: 5432
```
默认 5432,保持不变(MySQL 是 3306,Postgres 是 5432,别搞混)。

**步骤7:Advanced Options**

Locale: **Default locale** 或选 `Chinese (Simplified), China`(中文环境)

**步骤8:Pre Installation Summary → Ready to Install**

点击 Next 开始安装,等待完成。

**步骤9:完成**

最后会问要不要启动 Stack Builder,**不需要**,取消勾选,Finish。

### 2.3 验证安装

**方法1:命令行验证**

PostgreSQL 的命令行工具叫 `psql`,默认没加到 PATH,我们手动用:

```bash
# 进入 bin 目录
cd D:\database\postgresql\bin

# 连接到默认数据库(默认用户是 postgres)
psql -U postgres
```

输入密码后看到:
```
postgres=#
```

恭喜!试几条命令:

```sql
-- 查看所有数据库
\l

-- 查看版本
SELECT version();

-- 退出
\q
```

> 💡 PostgreSQL 的命令有两种:
> - 以 `\` 开头的是 psql 的元命令(如 `\l`、`\q`)
> - 以 `;` 结尾的是标准 SQL

**方法2:服务验证**

`Win + R` → `services.msc` → 找到 `postgresql-x64-16`(版本号根据你装的版本),应该是 **正在运行**。

### 2.4 配置环境变量

让 psql 全局可用:

1. **此电脑** → **属性** → **高级系统设置** → **环境变量**
2. 编辑 **Path**,新增:`D:\database\postgresql\bin`
3. 重新打开命令行,输入 `psql -U postgres` 测试

### 2.5 启动/停止 PostgreSQL

```bash
# 启动
net start postgresql-x64-16

# 停止
net stop postgresql-x64-16
```

(数字根据你装的版本变化)

---

## 三、Redis 安装

> ⚠️ **重要说明**:Redis 官方**不支持 Windows**!但有两种解决方案:
> 1. **方法A**:用社区维护的 Windows 版本(简单,适合学习)
> 2. **方法B**:用 WSL2 跑官方 Linux 版本(推荐,生产环境一致)

### 方法A:Memurai(免费 Redis Windows 替代,推荐新手)

**Memurai** 是 Redis 的 100% 兼容 Windows 实现,有免费的 Developer 版。

**步骤1:下载**

访问 https://www.memurai.com/get-memurai,填邮箱(可以随便填),下载 **Memurai Developer Edition**。

**步骤2:安装**

双击 `.msi`,一路 Next:
- 选择安装路径:`D:\database\memurai\`
- 端口:`6379`(Redis 默认端口)
- 勾选 Add to PATH(加入环境变量)
- 勾选 Install as Windows Service(作为服务)

**步骤3:验证**

```bash
# 连接 Redis
memurai-cli

# 测试命令
127.0.0.1:6379> ping
PONG

127.0.0.1:6379> set name "lrj"
OK

127.0.0.1:6379> get name
"lrj"

127.0.0.1:6379> exit
```

### 方法B:WSL2 + Redis(推荐进阶用户)

如果你以后要做后端开发,**强烈推荐这个方案**,因为生产环境都是 Linux。

**步骤1:启用 WSL2**

以**管理员身份**打开 PowerShell,执行:

```powershell
# 一键安装 WSL2 和默认 Ubuntu
wsl --install
```

重启电脑后会自动配置 Ubuntu,设置用户名和密码。

**步骤2:在 WSL2 中安装 Redis**

打开 Ubuntu(开始菜单搜索),执行:

```bash
# 更新软件包列表
sudo apt update

# 安装 Redis
sudo apt install redis-server -y

# 启动 Redis 服务
sudo service redis-server start

# 测试
redis-cli ping
# 输出 PONG 就成功了

# 简单测试
redis-cli
127.0.0.1:6379> set test "hello"
OK
127.0.0.1:6379> get test
"hello"
127.0.0.1:6379> exit
```

**步骤3:让 Windows 程序能访问**

WSL2 的 Redis 默认只监听 127.0.0.1,Windows 可以直接通过 `localhost:6379` 访问,无需额外配置(WSL2 自动转发)。

**步骤4:开机自启(可选)**

WSL2 默认不开机启动 Redis,每次要手动 `sudo service redis-server start`。

在 Windows 上创建一个启动脚本 `start-redis.bat`:
```batch
wsl sudo service redis-server start
```

把它放到启动文件夹(`Win + R` 输入 `shell:startup`)。

### 3.3 Redis 常用命令速查

```bash
# 字符串
SET key value           # 设置值
GET key                 # 获取值
DEL key                 # 删除key
EXPIRE key 60           # 设置过期时间(60秒)
TTL key                 # 查看剩余过期时间

# 列表
LPUSH list a b c        # 从左插入
RPUSH list x y z        # 从右插入
LRANGE list 0 -1        # 查看全部元素

# 哈希(类似对象)
HSET user:1 name lrj age 25
HGET user:1 name
HGETALL user:1

# 集合
SADD tags game frontend backend
SMEMBERS tags

# 通用
KEYS *                  # 查看所有key(生产环境禁用!)
FLUSHDB                 # 清空当前数据库(危险!)
INFO                    # 查看服务器信息
```

---

## 四、图形化管理工具

### 推荐工具对比

| 工具 | 支持数据库 | 优点 | 缺点 |
|------|----------|------|------|
| **DBeaver** ⭐推荐 | MySQL/PG/Redis 等几乎所有 | 全免费、支持几十种数据库、功能强大 | 启动稍慢(Java) |
| **MySQL Workbench** | 仅 MySQL | MySQL 官方、ER 图设计强 | 只支持 MySQL |
| **pgAdmin 4** | 仅 PostgreSQL | PG 官方、功能全 | 只支持 PG |
| **Another Redis Desktop Manager** | 仅 Redis | 免费、好看、好用 | 只支持 Redis |
| **HeidiSQL** | MySQL/PG | 轻量、快 | 界面相对旧 |

**新手推荐组合**:DBeaver(管所有 SQL 数据库)+ Another Redis Desktop Manager(管 Redis)

### 4.1 DBeaver Community 安装

**步骤1:下载**

访问 https://dbeaver.io/download/,选择 **Windows (installer)** 下载。

**步骤2:安装**

双击安装包,一路 Next:
- 安装路径建议:`D:\tools\DBeaver\`
- 其他保持默认

**步骤3:连接 MySQL**

1. 打开 DBeaver,左上角点击 **新建数据库连接** 图标(插头形状)
2. 选择 **MySQL** → Next
3. 填写连接信息:
   - Server Host: `localhost`
   - Port: `3306`
   - Database: 留空或填 `mysql`
   - Username: `root`
   - Password: 你设置的 MySQL root 密码
4. 点击 **Test Connection** 测试
5. 首次连接会提示下载驱动,点击 **Download** 即可
6. 测试成功后点击 **Finish**

左侧导航栏会出现你的连接,展开就能看到所有数据库、表。

**步骤4:连接 PostgreSQL**

同样的操作,新建连接选 **PostgreSQL**:
- Host: `localhost`
- Port: `5432`
- Database: `postgres`
- Username: `postgres`
- Password: 你设置的密码

**DBeaver 常用操作**:
- 双击表名打开数据浏览
- 右键表 → **View Data**:查看数据
- 右键表 → **Edit Table**:修改结构
- `Ctrl + Enter`:执行选中的 SQL
- `Ctrl + Space`:SQL 自动补全
- 右键数据库 → **SQL Editor**:打开 SQL 编辑器

### 4.2 Another Redis Desktop Manager 安装

**步骤1:下载**

访问 https://github.com/qishibo/AnotherRedisDesktopManager/releases

下载最新版的 `.exe`(Windows 版)。

**步骤2:安装并连接**

1. 双击安装,完成后打开
2. 点击 **+ New Connection**
3. 填写:
   - Name: `本地Redis`
   - Host: `127.0.0.1`
   - Port: `6379`
   - Password: 默认无密码(留空)
4. 点击 **Test Connection** → **OK**

左侧会出现连接,点击展开能看到所有 key,支持可视化的增删改查。

### 4.3 MySQL Workbench(可选,安装 MySQL 时已经装了)

如果你装 MySQL 时勾选了 Workbench,开始菜单可以找到。

- 打开后会自动显示本地连接(`Local instance MySQL80`)
- 双击连接,输入 root 密码
- 左侧 SCHEMAS 显示所有数据库
- 顶部点击 **+ 号图标** 新建 SQL 标签页执行查询

> 💡 Workbench 的 **ER 图设计**功能很强,适合可视化设计数据库表结构。

### 4.4 pgAdmin 4(可选,安装 PostgreSQL 时已经装了)

开始菜单找到 **pgAdmin 4**,首次打开:
1. 设置 master 密码(用于保护连接信息)
2. 左侧 Servers 下默认有 PostgreSQL,双击展开
3. 输入安装时设置的 postgres 密码
4. 展开 → Databases,可以看到所有数据库

---

## 五、Node.js 连接测试

装完数据库,写个小脚本测试 Node.js 能不能连上(为后续全栈开发铺路)。

### 5.1 安装 Node.js

如果还没装,从 https://nodejs.org/ 下载 **LTS 版本**安装。

验证:
```bash
node -v
npm -v
```

### 5.2 创建测试项目

```bash
# 创建测试目录
mkdir db-test
cd db-test

# 初始化 npm 项目
npm init -y

# 安装三个数据库的驱动
npm install mysql2 pg redis
```

### 5.3 MySQL 连接测试

创建文件 `test-mysql.js`:

```javascript
// 引入 mysql2 的 Promise 版本,支持 async/await
const mysql = require('mysql2/promise');

async function testMySQL() {
    try {
        // 创建连接,把密码改成你自己的
        const conn = await mysql.createConnection({
            host: 'localhost',
            port: 3306,
            user: 'root',
            password: '你的MySQL密码',   // ⚠️ 改成你的密码
            database: 'mysql'           // 先连默认的mysql数据库测试
        });

        console.log('✅ MySQL 连接成功!');

        // 执行一条简单查询:获取版本号
        const [rows] = await conn.execute('SELECT VERSION() AS version');
        console.log('MySQL 版本:', rows[0].version);

        // 关闭连接
        await conn.end();
    } catch (error) {
        console.error('❌ MySQL 连接失败:', error.message);
    }
}

testMySQL();
```

运行:
```bash
node test-mysql.js
```

应该看到:
```
✅ MySQL 连接成功!
MySQL 版本: 8.0.x
```

### 5.4 PostgreSQL 连接测试

创建文件 `test-pg.js`:

```javascript
// 引入 pg 的 Client 类(单连接)
const { Client } = require('pg');

async function testPG() {
    // 创建客户端,改成你的密码
    const client = new Client({
        host: 'localhost',
        port: 5432,
        user: 'postgres',
        password: '你的PostgreSQL密码',   // ⚠️ 改成你的密码
        database: 'postgres'              // 默认数据库
    });

    try {
        await client.connect();
        console.log('✅ PostgreSQL 连接成功!');

        // 查询版本
        const result = await client.query('SELECT version()');
        console.log('PostgreSQL 版本:', result.rows[0].version);

        await client.end();
    } catch (error) {
        console.error('❌ PostgreSQL 连接失败:', error.message);
    }
}

testPG();
```

运行:
```bash
node test-pg.js
```

### 5.5 Redis 连接测试

创建文件 `test-redis.js`:

```javascript
// 引入 redis 客户端
const redis = require('redis');

async function testRedis() {
    // 创建客户端,默认连 localhost:6379
    const client = redis.createClient({
        url: 'redis://localhost:6379'
        // 如果有密码:url: 'redis://:你的密码@localhost:6379'
    });

    // 监听错误事件
    client.on('error', (err) => console.error('Redis 错误:', err));

    try {
        await client.connect();
        console.log('✅ Redis 连接成功!');

        // 测试写入和读取
        await client.set('test_key', 'Hello Redis!');
        const value = await client.get('test_key');
        console.log('读取的值:', value);

        // 设置带过期时间的key(60秒后自动删除)
        await client.set('temp_key', 'will expire', { EX: 60 });
        const ttl = await client.ttl('temp_key');
        console.log('temp_key 剩余生存时间:', ttl, '秒');

        // 删除测试 key
        await client.del('test_key');

        await client.quit();
    } catch (error) {
        console.error('❌ Redis 连接失败:', error.message);
    }
}

testRedis();
```

运行:
```bash
node test-redis.js
```

三个测试都通过,说明你的数据库环境完全可用了!🎉

---

## 六、常见问题排查

### 6.1 MySQL 启动失败 / 连不上

**问题1:`Can't connect to MySQL server on 'localhost'`**

```bash
# 检查服务是否运行
net start MySQL80
# 如果提示 "服务已启动",但还是连不上,看下面

# 检查端口是否被占用
netstat -ano | findstr 3306
# 如果有别的进程占用,改 MySQL 端口或关掉那个进程
```

**问题2:忘记 root 密码**

```bash
# 1. 停止 MySQL 服务(管理员命令行)
net stop MySQL80

# 2. 在 my.ini 配置文件中加入(找到 [mysqld] 下加一行)
#    文件位置一般在:C:\ProgramData\MySQL\MySQL Server 8.0\my.ini
skip-grant-tables

# 3. 启动 MySQL
net start MySQL80

# 4. 无密码连接
mysql -u root

# 5. 重置密码(在 mysql> 中执行)
USE mysql;
UPDATE user SET authentication_string='' WHERE user='root';
FLUSH PRIVILEGES;
EXIT;

# 6. 删除 my.ini 中的 skip-grant-tables 这一行,重启服务
net stop MySQL80
net start MySQL80

# 7. 设置新密码
mysql -u root
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
```

**问题3:中文乱码**

确保数据库、表、字段都用 `utf8mb4` 字符集:
```sql
-- 查看当前字符集
SHOW VARIABLES LIKE 'character%';

-- 创建数据库时指定
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 6.2 PostgreSQL 启动失败

**问题1:服务无法启动**

```bash
# 查看服务状态
sc query postgresql-x64-16

# 查看日志(位置示例)
# D:\database\data\postgresql\log\
# 看最新的日志文件,通常能找到具体错误原因
```

**问题2:数据目录权限问题**

右键数据目录(如 `D:\database\data\postgresql`)→ 属性 → 安全 → 编辑 → 添加用户 `NETWORK SERVICE`,给完全控制权限。

### 6.3 Redis 连接失败

**Memurai 方案**:
```bash
# 检查服务
sc query Memurai
# 没运行就启动
net start Memurai
```

**WSL2 方案**:
```bash
# 进入 WSL
wsl

# 检查 Redis 是否运行
sudo service redis-server status

# 启动
sudo service redis-server start

# 看监听地址(应该是 127.0.0.1:6379)
ss -tlnp | grep 6379
```

### 6.4 端口冲突

三个数据库默认端口:
| 数据库 | 端口 |
|--------|------|
| MySQL | 3306 |
| PostgreSQL | 5432 |
| Redis | 6379 |

如果冲突,查看占用进程并处理:
```bash
# 查看 3306 端口占用
netstat -ano | findstr 3306
# 最后一列是 PID,用任务管理器或 taskkill 处理
taskkill /PID 进程ID /F
```

### 6.5 防火墙问题

如果 Node.js 连不上,可能是防火墙拦截:

1. **Windows 安全中心** → **防火墙和网络保护** → **允许应用通过防火墙**
2. 添加 `mysqld.exe`、`postgres.exe`、`memurai.exe`
3. 勾选 **专用** 和 **公用**

### 6.6 安装路径有中文/空格导致的问题

如果数据库行为异常,检查:
- 安装路径是否有中文字符
- 数据目录路径是否有空格
- 用户名是否有中文(如 `C:\Users\李某某\`)

最稳妥的做法:**安装到 `D:\database\` 这种纯英文路径**。

---

## 附录:卸载与重装

### 卸载 MySQL

1. **控制面板** → **程序和功能** → 卸载所有 `MySQL` 开头的程序
2. 删除目录:
   - `D:\database\mysql\`
   - `D:\database\data\mysql\`
   - `C:\ProgramData\MySQL\`(隐藏文件夹,需要显示隐藏项目)
3. 用注册表编辑器删除残留(可选):
   - `Win + R` 输入 `regedit`
   - 删除 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\MySQL80`

### 卸载 PostgreSQL

1. 开始菜单找到 PostgreSQL → **Uninstall** 运行卸载程序
2. 删除目录:
   - `D:\database\postgresql\`
   - `D:\database\data\postgresql\`

### 卸载 Memurai / Redis

1. 控制面板卸载 Memurai
2. 或在 WSL 中:`sudo apt remove redis-server`

### 数据备份(重装前必做!)

**MySQL 备份**:
```bash
# 导出所有数据库到 backup.sql
mysqldump -u root -p --all-databases > backup.sql

# 导出指定数据库
mysqldump -u root -p mydb > mydb_backup.sql

# 恢复
mysql -u root -p mydb < mydb_backup.sql
```

**PostgreSQL 备份**:
```bash
# 导出所有数据库
pg_dumpall -U postgres > backup.sql

# 导出指定数据库
pg_dump -U postgres mydb > mydb_backup.sql

# 恢复
psql -U postgres mydb < mydb_backup.sql
```

**Redis 备份**:
Redis 默认每隔一段时间自动保存 `dump.rdb` 文件,在数据目录下,直接复制保存即可。

---

## 学习路径建议

装完之后的下一步:

1. ✅ **第1周**:用 DBeaver 在本地 MySQL 创建表,练习 CRUD
2. ✅ **第2周**:用 Node.js 连数据库,写简单的接口(配合 Express)
3. ✅ **第3周**:学 Redis 缓存,做"接口缓存"练习
4. ✅ **第4周**:尝试 ORM(Prisma)做个全栈项目

把上次给你的 [《关系型数据库完整学习指南》](#) 配合着练,理论 + 实战双线推进效果最好。

---

**安装顺利的话,你现在应该有:**
- ✅ MySQL 跑在 3306 端口
- ✅ PostgreSQL 跑在 5432 端口
- ✅ Redis 跑在 6379 端口
- ✅ DBeaver 管理 SQL 数据库
- ✅ Another Redis Desktop Manager 管理 Redis
- ✅ Node.js 能成功连接三个数据库

遇到任何报错可以截图发过来,我帮你排查 👍
