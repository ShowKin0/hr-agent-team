# New API 自建站点：部署 / Git / 维护 全流程学习手册

> 本文档由实战整理，面向想彻底学会的你，把每个命令、每个概念都讲透。
> 读完你能独立完成：改代码、同步到服务器、部署、备份、恢复、回滚。

---

# 第一章 整体架构

## 1.1 你的系统由哪几部分组成

你的 newapi 站点不是单一程序，而是一套"容器化"服务，通过 **Docker Compose** 一次管理多个服务：

| 服务 | 镜像 | 作用 | 端口 |
| --- | --- | --- | --- |
| new-api | （自己构建的 custom）| API 网关主程序（前端+后端）| 3000→3000 |
| postgres | postgres:15 | 数据库（存用户/渠道/配置/充值）| 内部5432 |
| redis | redis:latest | 缓存（加速请求）| 内部6379 |

**关键理解**：
- new-api = "门面"，提供网站和 API
- postgres = "账本"，所有永久数据（要备份/迁移的核心）
- redis = "临时记忆"，只是缓存，丢了会自建，不用备份

## 1.2 开发与部署链路

```
本地电脑 --push--> GitHub私有仓库 --pull--> 服务器源码目录 --build--> docker镜像 --up--> 容器(运行) --连--> postgres(pg_data卷)
```

- 本地电脑：写代码的地方（VS Code）
- GitHub 私有仓库：代码中转站 + 历史保险箱
- 服务器源码目录：从 GitHub 拉来，用于构建镜像
- Docker 容器：实际运行的程序
- postgres 数据卷：数据存储（与代码分离，重建容器不丢）

---

# 第二章 Git 详解

## 2.1 什么是 Git？为什么需要它？

**Git** 是**版本控制工具**，帮你记录每次改动，随时回退历史版本。

一个人也要用 Git 的原因：
- 改坏代码能一键回到可用版本（回滚）
- 本地与服务器之间同步代码（今天的核心价值）
- 每笔改动有历史，知道哪天改了什么
- GitHub 私有仓库 = 云端保险箱

## 2.2 关键概念：仓库、分支、remote

| 概念 | 解释 |
| --- | --- |
| 仓库 (Repository) | 一个项目所有代码+历史，通常是一个目录 |
| commit | 一次"存档点"，把改动固定下来 |
| 分支 (branch) | 代码的"时间线"，默认 main |
| remote | "远程仓库地址的别名"，不用每次打长网址 |
| origin/upstream/mine | 都是 remote 名字（别名），指向不同远程仓库 |

## 2.3 深度讲解：remote

### 为什么一个仓库有多个 remote？

今天的关键策略：

```bash
upstream → https://github.com/QuantumNous/new-api.git   # 官方
mine     → https://github.com/ShowKin0/new-api.git      # 你的私有
```

- upstream（上游）= 官方来源，以后官方更新可 pull upstream 拉新代码
- mine（我的）= 你自己掌控，改动 push 到这，服务器也从这拉
- 既保留官方历史（能升级），又用自己仓库（能同步、能回滚）

### remote 命令逐个解释

```bash
# ① 查看当前 remote
git remote -v
# -v = --verbose 详细模式，显示每个 remote 的 fetch(取) 和 push(推) 地址

# ② 添加 remote
git remote add mine https://github.com/ShowKin0/new-api.git
# add = 添加新远程地址，命名为 mine

# ③ 改名 remote
git remote rename origin upstream
# rename = 把 origin 改名为 upstream（clone时官方默认叫 origin）

# ④ 删除 remote
git remote remove mine
```

### remote 的删除与恢复（想删官方 remote？看这里）

**为什么可以放心删 remote？**

remote 只是「远程仓库地址的快捷方式」，删除它：
- **不影响代码** — 代码、提交历史都在
- **不影响你的 push/pull** — 只要当前分支跟踪的 remote 还在

**delete remote 命令：**

    git remote remove <名字>
    # 例：删除官方 remote
    git remote remove upstream
    # 删完只剩 mine，本地照常工作

**恢复被删的 remote（后悔了再加回来）：**

    # 想重新关联官方，随时可加回
    git remote add upstream https://github.com/QuantumNous/new-api.git
    # 或恢复自己的
    git remote add mine https://github.com/ShowKin0/new-api.git

**判断该不该删 upstream：**
- 想更干净、只留自己的仓库 → 可删（不影响 main，它跟踪 mine）
- 想继续跟随官方升级 → 建议保留 upstream（不碍事，还能拉官方新代码）
- 删了以后想要 → git remote add 随时加回，无损失

### 重点：直接 git pull 到底拉哪个 remote？

**git pull（不带参数）拉的是「当前分支跟踪的远程仓库」**，不是随机的。

你的 main 分支因为执行过：

    git push -u mine main
    # -u = 把 main 分支绑定(跟踪)到 mine 远程

所以：

| 命令 | 拉谁 | 说明 |
| --- | --- | --- |
| git pull | **mine（你的私有）** | 因为 main 跟踪 mine |
| git pull mine main | mine（私有） | 显式指定 |
| git pull upstream main | **官方** | 显式指定官方才拉官方 |
| git fetch upstream + merge | 官方 | 拉官方代码再合并 |

**查看 main 跟踪哪个 remote：**

    git branch -vv
    # 看到 [mine/main] 就说明 main 跟的是 mine
    git status -sb

> 所以直接 git pull 是安全的：拉的是你自己的私有仓库，不会把官方覆盖过来。

---
## 2.4 私有仓库的认证：PAT 与 SSH Key（很重要！）

### 为什么私有仓库需要 密钥 认证？

GitHub **私有仓库**是私密的，所以 push / clone / pull 都要证明你是主人。
GitHub 2021 年后**不再支持账号密码验证**，改用两种认证：

1. **Personal Access Token（PAT）** —— 类似临时密码的令牌
2. **SSH Key** —— 一对加密钥匙（更安全、免输密码）

---

### 方式一：Personal Access Token（PAT） 最常用

PAT 是一长串字符串（形如 ghp_xxxxxxxx），相当于专用密码，只授权特定权限。

**生成步骤：**
1. GitHub 网页 -> 右上头像 -> **Settings**
2. 底部左边 -> **Developer settings** -> **Personal access tokens** -> **Tokens (classic)**
3. **Generate new token (classic)**
4. 填名字（如 server-deploy）、选有效期
5. 勾权限：常用勾 **repo**（私有库读写）；更精确可只勾 public_repo
6. Generate，**立即复制保存**（只显示一次！）

**用 PAT clone 私有仓库：**

    git clone https://USERNAME:PAT@github.com/ShowKin0/new-api.git
    # 或先 clone，提示 Username 填 GitHub 用户名，Password 填 PAT

**避免每次输 PAT**（缓存凭据）：

    git config --global credential.helper store

---

### 方式二：SSH Key（更安全，推荐长期用）

一对钥匙：私钥留在本机，公钥填到 GitHub，认证后免输密码。

**① 生成密钥对：**

    ssh-keygen -t ed25519 -C "你的邮箱"
    # 一路回车，生成 ~/.ssh/id_ed25519(私钥) 和 .pub(公钥)

**② 复制公钥：**

    cat ~/.ssh/id_ed25519.pub

**③ 把公钥加进 GitHub：** Settings -> **SSH and GPG keys** -> New SSH key -> 粘贴 -> Save

**④ 用 SSH 方式 clone/remote：**

    git clone git@github.com:ShowKin0/new-api.git
    git remote add mine git@github.com:ShowKin0/new-api.git

**看地址判断方式：** https:// 开头=HTTPS用PAT；git@github.com 开头=SSH用Key

---

### 两种方式对比

| 方式 | 地址 | 优点 | 缺点 |
| --- | --- | --- | --- |
| HTTPS+PAT | https://用户名:PAT@... | 简单、可设权限/有效期 | 可能每次输PAT |
| SSH Key | git@github.com:... | 安全、免输密码 | 配置稍繁琐 |

建议：长期维护用 SSH Key 一劳永逸；临时用 PAT。

### 密钥安全红线

- PAT / SSH 私钥 = 密码，泄露=别人能读写你的私有仓库
- 绝不把 PAT / 私钥提交进代码、仓库、截图公开
- 泄露时立刻去 GitHub 撤销（Revoke / Delete）
- PAT 设短有效期（如90天），定期换

---
## 2.5 核心命令逐个解释

### ① clone - 复制远程仓库到本地
```bash
git clone https://github.com/QuantumNous/new-api.git
```
- 把远程仓库全部代码+历史复制到本地
- 克隆后自动生成叫 origin 的 remote

### ② status - 当前状态
```bash
git status
```
- 显示改了什么、当前分支。红色=已改未暂存，绿色=已暂存待提交

### ③ add - 加入待提交区
```bash
git add .
```
- . = 当前目录所有改动；也可 git add web/src/xx.tsx 只加某文件
- 分 add/commit 两步，方便选择性提交

### ④ commit - 存档
```bash
git commit -m "初始化：new-api 官方源码 + 私有仓库同步"
```
- -m "说明" = 附描述。好说明是"人话"：如 修复登录页白屏

### ⑤ push - 推送到远程
```bash
git push -u mine main
```
- push = 把本地 commit 推送到指定远程
- mine = 你的私有仓库；main = main 分支
- -u = --set-upstream 跟踪；第一次 -u 后，以后 git push 默认推 mine/main

### ⑥ pull - 拉取最新
```bash
git pull
git pull mine main   # 指定来源
```
- 把远程新代码拉下来合并。服务器靠它拿本地 push 的新代码

### ⑦ log - 历史
```bash
git log --oneline -5
```
- --oneline 简洁每行；-5 看最近5条

### ⑧ branch - 分支
```bash
git branch          # 列出所有分支
git branch -M main  # 强制改名当前分支为 main
```

## 2.6 .gitignore - 让 Git 忽略某些文件

```gitignore
node_modules/
dist/
.env
.env.*
*.log
*.db
*.sql
data/
```

原因：
- node_modules/：前端依赖几十万文件，提交会撑爆仓库，应在服务器重 npm install
- dist/：构建产物，可再生，不提交
- *.sql、*.db：数据库备份，**含密码/密钥，绝不能进仓库**
- .env：环境变量，可能含密钥

判断原则：如果某文件"别人重新生成就能得到"，通常不提交。

## 2.7 Git 安全红线

- 密钥/密码不进仓库（SQL_DSN密码、epay密钥、SMTP密码、sql备份）
- 私有仓库别公开
- 即使删了密钥文件，历史里还有，公开会泄露