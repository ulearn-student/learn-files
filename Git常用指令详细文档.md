# Git 常用指令详细文档

> Git 是代码版本管理工具，记录每一次代码改动，可以随时回退、对比、多人协作。

---

## 一、基本概念

```
工作区（Working Directory）
→ 你在电脑上直接编辑的文件夹

暂存区（Stage / Index）
→ 准备提交的"临时存放区"，执行 git add 后文件进入这里

本地仓库（Local Repository）
→ 执行 git commit 后，代码正式保存在本地的历史记录里

远程仓库（Remote Repository）
→ GitHub / Gitee 上的仓库，执行 git push 后同步上去
```

**文件流转过程：**

```
修改文件 → git add → git commit → git push
 工作区       暂存区     本地仓库     远程仓库
```

---

## 二、初始化配置（第一次用 Git 必做）

```bash
# 设置用户名（显示在提交记录里）
git config --global user.name "你的名字"

# 设置邮箱
git config --global user.email "你的邮箱"

# 查看当前配置
git config --list

# 设置默认分支名为 main（GitHub 默认是 main）
git config --global init.defaultBranch main
```

---

## 三、创建仓库

```bash
# 在当前文件夹初始化一个新的 Git 仓库
git init
# 执行后当前目录会出现一个 .git 隐藏文件夹，里面存储所有历史记录

# 克隆远程仓库到本地（把别人的或自己的项目下载下来）
git clone https://github.com/用户名/仓库名.git

# 克隆到指定文件夹名
git clone https://github.com/用户名/仓库名.git 自定义文件夹名
```

---

## 四、查看状态

```bash
# 查看当前状态（最常用！提交前一定要看）
git status
# 显示哪些文件被修改、哪些在暂存区、哪些未被追踪

# 查看文件具体改了什么内容
git diff
# 显示工作区和暂存区的差异（还没 git add 的改动）

git diff --staged
# 显示暂存区和上次提交的差异（已 git add 但还没 git commit 的改动）

# 查看提交历史
git log
# 显示所有提交记录（时间、作者、提交信息）

git log --oneline
# 简洁版，每条记录只显示一行

git log --oneline --graph
# 图形化显示分支合并历史
```

---

## 五、添加文件到暂存区

```bash
# 添加单个文件
git add 文件名.js

# 添加某个文件夹下所有文件
git add src/

# 添加所有改动的文件（最常用）
git add .

# 添加所有 .js 文件
git add *.js

# 交互式选择要添加的内容
git add -p
```

---

## 六、提交

```bash
# 提交暂存区的内容（最常用）
git commit -m "提交说明"

# 提交说明的规范写法：
# feat:     新增功能
# fix:      修复 bug
# docs:     修改文档
# chore:    删除/整理文件，不影响功能
# refactor: 重构代码
# style:    格式调整（空格、换行等）
# test:     新增测试

# 示例：
git commit -m "feat: 新增用户登录功能"
git commit -m "fix: 修复订单金额计算错误"
git commit -m "docs: 更新 README"

# 跳过 git add，直接提交所有已追踪文件的改动
git commit -am "提交说明"
# 注意：新建的文件（Untracked）不会被包含，还是要先 git add

# 修改最近一次提交的说明（还没 push 时才能用）
git commit --amend -m "修改后的说明"
```

---

## 七、远程仓库操作

```bash
# 查看当前关联的远程仓库
git remote -v

# 关联远程仓库
git remote add origin https://github.com/用户名/仓库名.git
# origin 是远程仓库的别名，可以自定义，一般用 origin

# 修改远程仓库地址
git remote set-url origin https://github.com/用户名/新仓库名.git

# 删除远程仓库关联
git remote remove origin

# 推送到远程仓库（第一次推送用 -u，以后直接 git push）
git push -u origin main
# -u 设置默认上游，以后直接 git push 就行

# 推送（日常使用）
git push

# 强制推送（⚠️ 危险！会覆盖远程记录，一般不用）
git push --force

# 拉取远程最新代码（下载 + 合并）
git pull

# 只下载不合并
git fetch
```

---

## 八、分支操作

```bash
# 查看所有分支
git branch
# 带 * 的是当前所在分支

# 查看所有分支（包括远程）
git branch -a

# 创建新分支
git branch 分支名

# 切换到某个分支
git checkout 分支名

# 创建并切换（常用）
git checkout -b 分支名

# 新版写法（Git 2.23+）
git switch 分支名
git switch -c 分支名   # 创建并切换

# 删除分支（已合并的）
git branch -d 分支名

# 强制删除分支（未合并的）
git branch -D 分支名

# 合并分支（把 dev 合并到当前分支）
git merge dev

# 推送新分支到远程
git push origin 分支名

# 删除远程分支
git push origin --delete 分支名
```

**分支使用场景：**

```
main（主分支）     → 稳定版本，不直接在这里开发
dev（开发分支）    → 日常开发
feature/xxx       → 开发某个新功能
fix/xxx           → 修复某个 bug

开发流程：
1. git checkout -b feature/登录功能   # 新建功能分支
2. 写代码...
3. git add . && git commit -m "feat: 完成登录功能"
4. git checkout main                  # 切回主分支
5. git merge feature/登录功能          # 合并
6. git branch -d feature/登录功能      # 删除功能分支
```

---

## 九、版本标签

```bash
# 查看所有标签
git tag

# 打一个轻量标签（只是个书签）
git tag v1.0.0

# 打一个附注标签（推荐，包含说明信息）
git tag -a v1.0.0 -m "第一版：基础功能完成"

# 给历史某次提交打标签
git tag -a v0.9.0 提交的hash值 -m "测试版"

# 推送单个标签到远程
git push origin v1.0.0

# 推送所有标签到远程
git push origin --tags

# 删除本地标签
git tag -d v1.0.0

# 删除远程标签
git push origin --delete v1.0.0
```

---

## 十、撤销操作

```bash
# ===== 还没 git add =====

# 撤销单个文件的修改（恢复到上次提交的状态）
git checkout -- 文件名.js

# 撤销所有未暂存的修改
git checkout -- .


# ===== 已经 git add，还没 git commit =====

# 把文件从暂存区移出（改动还在，只是取消 add）
git restore --staged 文件名.js

# 把所有文件从暂存区移出
git restore --staged .


# ===== 已经 git commit，还没 git push =====

# 撤销最近一次提交，保留代码改动（代码还在，只是取消提交）
git reset --soft HEAD~1

# 撤销最近一次提交，代码改动也丢弃（⚠️ 谨慎）
git reset --hard HEAD~1

# 撤销最近 3 次提交
git reset --soft HEAD~3


# ===== 已经 git push 到远程 =====

# 用新提交来撤销某次提交（安全，推荐）
git revert 提交的hash值
# 不会删除历史记录，而是新增一条"撤销"记录
```

---

## 十一、.gitignore 文件

`.gitignore` 告诉 Git 哪些文件不要追踪，不会被提交到仓库。

```bash
# Node.js 项目常用的 .gitignore 内容：

node_modules/        # 依赖包（体积大，从 package.json 重新安装就行）
.env                 # 环境变量（含密码，不能提交）
.env.local
*.log                # 日志文件
.DS_Store            # macOS 系统文件
Thumbs.db            # Windows 系统文件
dist/                # 构建输出目录
build/
coverage/            # 测试覆盖率报告
```

```bash
# 如果已经提交了不该提交的文件，用这个从 Git 移除（文件本身不删除）
git rm --cached 文件名
git rm --cached -r 文件夹名/

# 然后提交
git commit -m "chore: 从 Git 移除敏感文件"
```

---

## 十二、日常工作流（完整示例）

```bash
# ===== 场景1：日常开发提交 =====

cd 项目目录
git status                          # 看看改了什么
git add .                           # 添加所有改动
git status                          # 再确认一遍
git commit -m "feat: 新增职位搜索功能"
git push                            # 推送到远程


# ===== 场景2：拉取别人的最新代码 =====

git pull                            # 拉取并合并
# 如果有冲突，手动解决冲突后：
git add .
git commit -m "fix: 解决合并冲突"
git push


# ===== 场景3：开发新功能（分支工作流）=====

git checkout -b feature/用户登录     # 新建功能分支
# 写代码...
git add .
git commit -m "feat: 完成用户登录"
git push origin feature/用户登录     # 推送功能分支
# 在 GitHub 上发起 Pull Request
# 审核通过后合并到 main


# ===== 场景4：紧急修复 bug =====

git checkout main                   # 切到主分支
git checkout -b fix/登录密码错误      # 新建修复分支
# 修复代码...
git add .
git commit -m "fix: 修复登录密码验证错误"
git checkout main
git merge fix/登录密码错误
git push
git branch -d fix/登录密码错误        # 删除修复分支
```

---

## 十三、查看和对比

```bash
# 查看某次提交的详细内容
git show 提交hash值

# 对比两次提交之间的差异
git diff 提交hash1 提交hash2

# 查看某个文件的修改历史
git log --follow 文件名.js

# 查看是谁修改了某一行（追责神器）
git blame 文件名.js

# 搜索代码中的某个关键词
git grep "关键词"
```

---

## 十四、快捷参考

```
常用命令速查：

git init                    初始化仓库
git clone <url>             克隆项目
git status                  查看状态
git add .                   添加所有文件
git commit -m "说明"        提交
git push                    推送到远程
git pull                    拉取最新代码
git log --oneline           查看提交历史
git branch -b <名字>        新建并切换分支
git merge <分支名>          合并分支
git tag -a v1.0.0 -m "说明" 打版本标签
git reset --soft HEAD~1     撤销最近一次提交
```

---

## 十五、常见错误处理

```bash
# 错误：fatal: not a git repository
# 原因：当前目录没有初始化 Git
# 解决：cd 到正确的项目目录，或执行 git init

# 错误：fatal: remote origin already exists
# 原因：已经关联过远程仓库
# 解决：git remote remove origin 然后重新 git remote add

# 错误：rejected - non-fast-forward
# 原因：远程有别人提交的新代码，你的本地落后了
# 解决：先 git pull，解决冲突后再 git push

# 错误：Please tell me who you are
# 原因：没有配置用户名和邮箱
# 解决：git config --global user.name "名字"
#       git config --global user.email "邮箱"

# 错误：The authenticity of host can't be established
# 原因：第一次用 SSH 连接 GitHub
# 解决：输入 yes 确认即可

# 错误：Could not resolve host: github.com
# 原因：网络无法访问 GitHub
# 解决：开启代理，或配置 Git 代理：
#       git config --global http.proxy http://127.0.0.1:7890
```
