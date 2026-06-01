# 🔧 Git 零基础快速上手指南

> 📅 生成时间：`2026-06-01 10:08 (GMT+8)`
>
> 目标：面向完全没有版本控制经验的新手，从安装到第一次 Pull Request，覆盖 GitHub / GitLab / Gitee 三大平台的不同机制

---

## 一、先理解：Git 到底是什么、解决什么问题

### 1.1 没有 Git 的世界

```
论文_v1.doc
论文_v2_导师改.doc
论文_v3_最终版.doc
论文_v3_最终版_真的最终.doc
论文_v3_最终版_打死不改.doc
论文_v3_最终版_打死不改_这次真的不改了.ppt
```

手动管理版本：混乱、不可追溯、无法协作。

### 1.2 Git 做了什么

Git 是一个**分布式版本控制系统**。核心能力拆开看：

```
每一次 git commit  =  给你当前的所有文件拍一张完整快照
任何时候 git log    =  翻看整个项目的"相册"，知道谁在什么时候改了什么
git branch          =  在不影响主线的前提下开一条岔路做实验
git merge           =  实验成功，把岔路合回主线
```

| 能力 | 说明 |
|------|------|
| **版本快照** | 每次 `commit` 都是一张完整的项目快照，随时可以回溯到任意历史版本 |
| **分支开发** | 在不影响主线的前提下实验新功能，成熟后再合并 |
| **分布式协作** | 每个人本地都有完整仓库，不依赖中央服务器也能工作 |
| **变更追溯** | 谁、什么时候、改了什么、为什么改——全部可查（`git blame` / `git log`） |

### 1.3 Git ≠ GitHub / GitLab / Gitee

这是一个最常见的误区，一定要先分清：

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Git  (命令行工具，跑在你的电脑上)                        │
│   ├── 管理版本历史                                       │
│   ├── 分支、合并、回滚                                    │
│   ├── 纯本地工作，不联网也能用                             │
│   └── 本质：一个 .git 隐藏文件夹                          │
│                                                          │
│   GitHub / GitLab / Gitee  (网页服务平台)                  │
│   ├── 托管你的 Git 仓库，提供云端备份                      │
│   ├── Pull Request / Merge Request（代码审查 + 合并请求）  │
│   ├── Issue 追踪、CI/CD、Wiki、项目管理                    │
│   └── 本质：一个带 Web UI 的远程 Git 服务器               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

> **类比：** Git 是"发动机"，GitHub/GitLab/Gitee 是"4S 店"。发动机自己就能跑，但 4S 店提供了展示、交易、保养等附加服务。

---

## 二、安装 Git

### 2.1 Linux (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install git
```

### 2.2 Linux (RHEL/CentOS/Fedora)

```bash
sudo dnf install git      # Fedora
sudo yum install git      # CentOS 7
```

### 2.3 macOS

```bash
# 方式一：使用 Homebrew（推荐）
brew install git

# 方式二：安装 Xcode Command Line Tools
xcode-select --install
```

### 2.4 Windows

1. 前往 [git-scm.com](https://git-scm.com/download/win) 下载安装包
2. 安装过程中，以下选项建议：
   - **默认编辑器**：选你熟悉的（新手选 Nano 或 VSCode）
   - **PATH 环境**：选 "Git from the command line and also from 3rd-party software"
   - **换行符转换**：选 "Checkout Windows-style, commit Unix-style line endings"
   - **终端模拟器**：选 "Use MinTTY"
3. 其余默认即可

### 2.5 验证安装

```bash
git --version
# 输出示例：git version 2.44.0
```

---

## 三、首次配置（只需做一次，但很重要）

这些信息会**嵌入每一次 commit**，是协作追溯的依据：

```bash
# 设置用户名（建议用平台上的显示名）
git config --global user.name "张三"

# 设置邮箱（必须和托管平台账户邮箱一致！）
git config --global user.email "zhangsan@example.com"

# 设置默认分支名（推荐 main，行业标准）
git config --global init.defaultBranch main

# 设置换行符自动处理（Windows 用户务必设置）
git config --global core.autocrlf true       # Windows
git config --global core.autocrlf input      # macOS / Linux

# 查看当前配置
git config --global --list
```

> **为什么邮箱要一致？** 平台（GitHub/GitLab/Gitee）通过 commit 中的邮箱来关联你的账户。不一致会导致 commit 无法关联到你的头像和贡献统计，绿格子不亮。

---

## 四、核心概念：四个区域

理解这四个区域是理解 Git 一切操作的关键。不要死记命令，先把这个模型装进脑子里：

```
┌──────────────┐    git add    ┌──────────────┐   git commit   ┌──────────────┐   git push    ┌──────────────┐
│   工作目录    │ ────────────→ │   暂存区      │ ────────────→ │   本地仓库    │ ────────────→ │   远程仓库    │
│ Working Dir  │              │ Staging Area │              │ Local Repo   │              │ Remote Repo  │
│              │ ←──────────── │              │ ←──────────── │              │ ←──────────── │              │
│ 你正在编辑   │   git restore │ 准备提交的   │  git reset   │  已提交的    │   git pull   │ GitHub/      │
│ 的文件       │              │ 文件清单     │              │  版本历史    │              │ GitLab/Gitee │
└──────────────┘              └──────────────┘              └──────────────┘              └──────────────┘
```

### 4.1 四个区域类比

| 区域 | 类比 | 说明 |
|------|------|------|
| **工作目录** | 📝 你的书桌 | 你正在编辑、修改、新建的文件 |
| **暂存区** | 📋 待办篮 | 你决定"这次要提交"的文件清单 |
| **本地仓库** | 🗄️ 档案柜 | 所有已提交的版本历史（数据在 `.git` 目录里） |
| **远程仓库** | ☁️ 云端备份 | GitHub / GitLab / Gitee 上的副本 |

### 4.2 关键心智模型

```
git add    = 把文件放进"待办篮"（拍照登记，告诉 Git "这些文件我打算提交"）
git commit = 把篮子里的东西打包存档（正式入档，生成一个版本号）
git push   = 把存档同步到云端（备份 + 分享给别人）
```

两个容易踩的坑：

- `git add` 之后再改同一个文件，新改动不会自动进入这次 commit——需要**再次 `git add`**
- `git commit` 只提交暂存区里的内容，工作目录里没 add 的修改不受影响，也不会丢

---

## 五、单人项目基础流程

### 5.1 场景 A：从零开始（本地已有代码，想推到 GitHub）

```
第 1 步：初始化     git init        →  在当前目录创建 .git，Git 开始追踪
第 2 步：查看状态   git status      →  看哪些文件改了、哪些还没追踪
第 3 步：加入暂存   git add .       →  把所有改动登记到"待办篮"
第 4 步：提交       git commit -m   →  打包存档，写入版本历史
第 5 步：关联远程   git remote add  →  告诉本地仓库"云端地址在哪"
第 6 步：推送       git push        →  把存档同步到云端
```

```bash
cd my-project
git init                          # 1. 初始化
git status                        # 2. 查看状态
git add README.md src/            # 3. 添加文件到暂存区
git commit -m "初始化项目结构"      # 4. 提交
git remote add origin git@github.com:你的用户名/my-project.git  # 5. 关联远程
git push -u origin main           # 6. 推送（-u 建立追踪，以后 git push 即可）
```

### 5.2 场景 B：已有远程仓库（Clone 下来开发）

```bash
git clone git@github.com:用户名/仓库名.git   # 1. 下载到本地
cd 仓库名                                     # 2. 进入项目

# ... 写代码 ...

git status                        # 3. 看改了什么
git diff                          # 4. 看改动的具体内容
git add .                         # 5. 加入暂存区
git commit -m "修复了登录页样式"   # 6. 提交
git push                          # 7. 推送
```

### 5.3 日常开发循环（记住这一个循环就够了）

```bash
# 开始工作前：拉取最新代码
git pull

# 工作中：随时查看状态
git status

# 完成一个功能点：
git add .
git commit -m "做了什么改动"
git push

# 循环往复 ↑
```

### 5.4 提交信息（Commit Message）规范

```bash
# ❌ 这样写等于没写——半年后你自己都看不懂
git commit -m "修改"
git commit -m "fix"
git commit -m "update"
git commit -m "111"

# ✅ 能让人（包括未来的你）一眼看懂
git commit -m "修复首页轮播图在移动端不显示的问题"
git commit -m "新增用户登录接口，支持 JWT 认证"
git commit -m "重构数据库连接模块，改用连接池"
```

**推荐格式**（约定式提交，大厂和开源项目通用）：

```
<类型>: <简短描述>

类型包括：
  feat      — 新功能
  fix       — 修复 Bug
  docs      — 文档变更
  style     — 代码格式（不影响功能）
  refactor  — 重构（不新增功能也不修 Bug）
  test      — 添加测试
  chore     — 构建/工具变更

示例：
  feat: 新增用户注册页面的手机验证码功能
  fix: 修复订单状态在并发场景下的数据不一致
  docs: 更新 API 接口文档中的认证说明
```

---

## 六、分支操作：多人协作的基石

### 6.1 为什么需要分支？

```
                ○ ← 在分支上开发新面板功能
               /
main ──●──●──●──● ← 主线保持稳定
                  \
                   ○ ← 在另一个分支上修复紧急 Bug
```

分支让你在不同任务间切换而不互相干扰。常见分支命名习惯：

| 分支名 | 用途 |
|--------|------|
| `main` / `master` | 稳定版本，随时可发布 |
| `develop` | 开发主线（稍大一点的团队用） |
| `feature/xxx` | 新功能开发 |
| `fix/xxx` | Bug 修复 |
| `release/x.x` | 发布准备 |

### 6.2 分支操作命令

```bash
# 查看分支
git branch          # 本地分支
git branch -r       # 远程分支
git branch -a       # 所有分支

# 创建 + 切换分支
git checkout -b feature-login    # 创建 feature-login 并切过去（最常用）
git switch -c feature-login      # Git 2.23+ 推荐方式

# 在分支上开发
# ... 修改文件 ...
git add .
git commit -m "feat: 完成登录页面 UI"

# 推送分支到远程
git push -u origin feature-login

# 合并分支（回到 main，把 feature 合进来）
git checkout main                # 或 git switch main
git merge feature-login

# 删除分支
git branch -d feature-login          # 删除本地分支（已合并）
git branch -D feature-login          # 强制删除（未合并）
git push origin --delete feature-login  # 删除远程分支
```

### 6.3 分支策略（简化版 Git Flow）

对于个人或小团队，记住这个就够：

```
main ────────────────────●────●─── 稳定发布（只 merge，不直接在上面写代码）
                          \   /
develop ────●──●──●──●──●──●── 开发主线
              \
feature-xxx ──●──●── 功能分支（开发完合并回 develop，然后删掉）
```

操作顺序：

```bash
# 1. 从 main 创建开发分支
git checkout -b develop main

# 2. 每次做新功能从 develop 开分支
git checkout -b feature/xxx develop

# 3. 功能完成，合并回 develop
git checkout develop
git merge feature/xxx
git branch -d feature/xxx    # 用完就删，保持整洁

# 4. 版本稳定后发布
git checkout main
git merge develop
git tag v1.0.0               # 打版本标签
```

---

## 七、远程仓库与三大平台

### 7.1 SSH vs HTTPS：两种连接方式

| | HTTPS | SSH |
|------|-------|-----|
| **优点** | 简单，无需额外配置 | 安全，推送无需反复输密码 |
| **缺点** | 每次推送可能需输入密码/Token | 需要先生成并配置 SSH Key |
| **推荐场景** | 临时 clone、一次性操作 | 日常开发（一劳永逸） |

### 7.2 配置 SSH Key（推荐，三大平台通用）

```bash
# 1. 生成 SSH 密钥（一路回车即可）
ssh-keygen -t ed25519 -C "你的邮箱@example.com"

# 如果系统不支持 ed25519，用 RSA：
ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"

# 2. 查看公钥
cat ~/.ssh/id_ed25519.pub    # ed25519 的情况
cat ~/.ssh/id_rsa.pub        # RSA 的情况

# 3. 复制输出的全部内容，粘贴到下面各平台的设置页面
```

**各平台添加 SSH Key 的位置：**

| 平台 | 设置路径 |
|------|----------|
| **GitHub** | Settings → SSH and GPG keys → New SSH key |
| **GitLab** | Preferences → SSH Keys |
| **Gitee** | 设置 → SSH 公钥 |

```bash
# 4. 测试连接
ssh -T git@github.com       # GitHub
ssh -T git@gitlab.com       # GitLab
ssh -T git@gitee.com        # Gitee

# 成功会看到类似：Hi xxx! You've successfully authenticated.
```

### 7.3 远程仓库操作

```bash
# 查看已关联的远程仓库
git remote -v

# 添加远程仓库（origin 是约定俗成的名字）
git remote add origin git@github.com:用户名/仓库名.git

# 修改远程仓库地址（比如从 GitHub 迁移到 Gitee）
git remote set-url origin git@gitee.com:用户名/仓库名.git

# 添加多个远程仓库（一份代码同时推送到 GitHub 和 Gitee）
git remote add github git@github.com:用户名/仓库名.git
git remote add gitee git@gitee.com:用户名/仓库名.git

# 分别推送
git push github main
git push gitee main

# 拉取远程分支到本地
git fetch origin
git checkout -b feature-xxx origin/feature-xxx
```

---

## 八、协作实战：从 Clone 到 Pull Request

这是开源贡献的核心流程，也是面试常问的"PR 流程"。注意：三大平台的术语和 UI 略有不同，但本质流程一致。

### 8.1 核心流程（6 步走）

```
Fork → Clone → Branch → Commit → Push → PR/MR
 ①      ②        ③        ④       ⑤       ⑥
```

### 8.2 每一步在做什么

#### ① Fork：把别人的仓库"复制"到自己名下

```
原仓库 (原作者)                    你的 Fork (你的账户下)
┌─────────────────┐              ┌─────────────────┐
│ github.com/     │    Fork     │ github.com/     │
│   原作者/项目     │ ──────────→ │   你的用户名/项目  │
│                 │              │                 │
│ main ●──●──●    │              │ main ●──●──●   │
└─────────────────┘              └─────────────────┘
```

各平台操作：

| 平台 | 操作路径 |
|------|---------|
| **GitHub** | 进入目标仓库 → 右上角 **Fork** 按钮 → Create fork |
| **GitLab** | 进入目标仓库 → 右上角 **Fork** 按钮 → 选择命名空间 |
| **Gitee** | 进入目标仓库 → 右上角 **Fork** 按钮 → 确认 |

> **Fork 的本质：** 在你的账户下创建一个原仓库的副本。你有这个副本的完全写权限，可以自由修改而不影响原仓库。等改完了，通过 PR/MR 请原作者把你的改动"拉"回原仓库。

#### ② Clone：把 Fork 后的仓库下载到本地

```bash
git clone git@github.com:你的用户名/仓库名.git
cd 仓库名
```

#### ③ 添加上游仓库 + 创建分支

```bash
# 添加上游仓库（原作者的），方便同步更新
git remote add upstream git@github.com:原作者/仓库名.git

# 验证：应该看到两个远程仓库（origin=你的，upstream=原作者的）
git remote -v
# origin    git@github.com:你的用户名/仓库名.git (fetch)
# origin    git@github.com:你的用户名/仓库名.git (push)
# upstream  git@github.com:原作者/仓库名.git (fetch)
# upstream  git@github.com:原作者/仓库名.git (push)

# 创建功能分支（永远不要在 main 分支上直接修改！）
git fetch upstream
git checkout -b feature/my-contribution upstream/main
```

#### ④ 开发 + 提交

```bash
# ... 写代码 ...

git status
git add .
git commit -m "feat: 新增用户导出 CSV 的功能"

# 有多个 commit 时，保持原子性：一个 commit = 一个逻辑完整的最小变更
git commit -m "feat: 新增 CSV 导出的工具函数"
git commit -m "feat: 在用户列表页接入 CSV 导出按钮"
```

#### ⑤ Push 到自己的远程仓库

```bash
git push -u origin feature/my-contribution
```

#### ⑥ 发起 Pull Request / Merge Request

| 平台 | 术语 | 操作方式 |
|------|------|---------|
| **GitHub** | Pull Request (PR) | 你的 Fork 仓库 → **Compare & pull request** → 填标题和描述 → **Create pull request** |
| **GitLab** | Merge Request (MR) | 你的 Fork 仓库 → **Create merge request** → 填标题和描述 → **Create merge request** |
| **Gitee** | Pull Request (PR) | 你的 Fork 仓库 → **Pull Requests** 标签 → **新建 Pull Request** → 填写信息 → 提交 |

**PR/MR 描述的推荐模板：**

```markdown
## 变更概述
新增了用户管理页面的 CSV 导出功能。

## 变更内容
- [x] 新增 CSV 导出的工具函数（`utils/export-csv.ts`）
- [x] 在用户列表页接入导出按钮
- [x] 添加相关单元测试

## 测试方式
1. 进入用户列表页
2. 点击右上角"导出 CSV"按钮
3. 验证下载文件中包含正确的用户数据

## 关联 Issue
Closes #42
```

### 8.3 同步上游更新（保持 Fork 不落后）

当原仓库在你开发期间有了新 commit，需要同步：

```bash
# 1. 拉取上游更新
git fetch upstream

# 2. 切换到 main 分支
git checkout main

# 3. 合并上游的 main
git merge upstream/main

# 4. 推送到自己的远程仓库
git push origin main

# 5. 切回开发分支，同步最新代码
git checkout feature/my-contribution
git merge main          # 或 git rebase main（历史更整洁）
```

---

## 九、常见场景与急救手册

### 场景 1：改错了，想撤销

**完整的时间线，按紧急程度从小到大：**

```
改错了
  ├── 还没 git add        →  git restore 文件名         （最简单，直接丢弃修改）
  ├── 已经 git add        →  git restore --staged 文件名 （从暂存区撤回，修改保留）
  ├── 已经 git commit     →  git reset --soft HEAD~1    （撤销 commit，修改保留）
  ├── 已经 git push       →  git revert HEAD && git push（安全！生成反向 commit）
  └── 连 commit hash 都丢了 → git reflog                （救命稻草，操作历史全在）
```

具体命令：

```bash
# 情况 A：还没 add（只在工作目录改了）
git restore 文件名       # 撤销单个文件
git restore .            # 撤销所有修改

# 情况 B：已经 add 但还没 commit（在暂存区）
git restore --staged 文件名   # 从暂存区撤回（文件修改保留）
git restore --staged .        # 撤回所有

# 情况 C：已经 commit 但还没 push
git commit --amend -m "新的提交信息"   # 只改 commit 信息
git commit --amend --no-edit          # 追加忘记 add 的文件到上一次 commit
git reset --soft HEAD~1               # 撤销 commit，修改回到暂存区
git reset --mixed HEAD~1              # 撤销 commit，修改回到工作目录
git reset --hard HEAD~1               # ⚠️ 彻底丢弃！修改也没了

# 情况 D：已经 push 到远程
git revert HEAD           # 生成反向 commit（安全，历史可追溯）
git push
# ⚠️ git reset --hard + git push --force-with-lease 仅限你一个人的分支
```

### 场景 2：切错分支就开始写代码了

```bash
# 别慌！先别 commit，用 stash 暂存
git stash                   # 暂存当前所有修改
git checkout 正确的分支      # 切到正确分支
git stash pop               # 恢复修改

# 查看 stash 列表
git stash list

# 应用特定 stash（不删除）
git stash apply stash@{0}

# 删除 stash
git stash drop stash@{0}
```

### 场景 3：合并冲突（Merge Conflict）

当两个分支修改了同一个文件的同一行时，Git 无法自动决定保留哪个。

**完整解决链路：**

```
第 1 步：合并时冲突       git merge feature/xxx
         ├── Git 输出 CONFLICT 警告
         └── 告诉你哪些文件有冲突

第 2 步：定位冲突文件     git status
         └── 显示 "both modified: src/app.ts"

第 3 步：打开文件，会看到冲突标记：
         <<<<<<< HEAD       ← 当前分支的版本
         const port = 3000;
         =======            ← 分隔线
         const port = 8080;
         >>>>>>> feature/xxx ← 要合并进来的版本

第 4 步：手动编辑，决定保留哪个（或合并两者）
         → 删除 <<<<<<<、=======、>>>>>>> 这三行标记
         → 保留你想要的内容

第 5 步：标记为已解决      git add 冲突文件

第 6 步：完成合并          git commit -m "merge: 解决端口配置冲突"
```

```bash
# 整个流程的命令行版本
git merge feature/xxx               # → 冲突！
git status                          # → 看哪些文件冲突
# 编辑冲突文件，删除 <<< === >>> 标记
git add 冲突的文件                    # → 标记已解决
git commit -m "merge: 解决端口配置冲突"
```

> **提示：** VSCode、JetBrains 等现代编辑器对冲突解决有图形化支持（一键选择 Accept Current / Accept Incoming / Accept Both），强烈推荐新手使用，比手改标记快得多。

### 场景 4：代码被强制覆盖了，想找回

```bash
# 查看所有操作历史（包括已删除的分支和 commit）—— 真正的救命稻草
git reflog

# 输出类似：
# abc1234 HEAD@{0}: commit: 新增功能
# def5678 HEAD@{1}: reset: moving to HEAD~1
# 7890abc HEAD@{2}: commit: 之前的提交（已被 reset 覆盖）
# ...

# 恢复到某个 reflog 记录
git checkout -b recovery-branch abc1234
# 或直接：
git reset --hard abc1234
```

> **reflog 是 Git 的"后悔药"：** 只要你在本地做过操作，reflog 里就有记录，默认保留 90 天。

### 场景 5：.gitignore 不生效

```bash
# .gitignore 只能忽略"未追踪"的文件
# 如果文件已经被 Git 追踪，需要先移除追踪：

git rm --cached 文件名                 # 从 Git 追踪中移除（文件本身保留）
git commit -m "chore: 移除不应追踪的文件"

# 确保 .gitignore 里已添加该文件，之后就不会再被追踪了
```

### 常用 .gitignore 模板

```gitignore
# 依赖目录
node_modules/
vendor/
__pycache__/

# 构建产物
dist/
build/
*.o
*.class

# 环境变量（敏感信息，绝对不要提交！）
.env
.env.local
.env.*.local

# IDE 配置
.vscode/
.idea/
*.swp
*.swo
*~

# 操作系统
.DS_Store      # macOS
Thumbs.db      # Windows

# 日志
*.log
logs/
```

---

## 十、进阶技巧

### 10.1 查看历史（代码考古）

```bash
git log                              # 完整提交历史
git log --oneline                    # 简洁单行显示
git log --oneline --graph --all      # 图形化分支结构（超直观）
git log -p 文件名                    # 某个文件的修改历史（看每次改了什么）
git blame 文件名                     # 每一行代码是谁写的、什么时候写的
git log --grep="关键词"              # 搜索 commit 信息
git show <commit-hash>               # 查看某次 commit 的详细改动
```

### 10.2 标签（Tag）：给版本打标记

```bash
git tag v1.0.0                              # 轻量标签
git tag -a v1.0.0 -m "第一个正式版本发布"      # 附注标签（推荐，有作者和说明）
git push origin v1.0.0                      # 推送单个标签
git push origin --tags                      # 推送所有标签
git tag                                     # 查看所有标签
git checkout -b hotfix-v1.0 v1.0.0          # 基于标签创建分支
```

### 10.3 Rebase：保持历史整洁

> ⚠️ 黄金法则：**永远不要 rebase 已经推送的公共分支！** rebase 会改写历史，只应该用在你自己本地还没 push 的分支上。

```bash
# 场景：feature 分支落后于 main
# 方式一：merge（安全，但会多一个 merge commit）
git checkout feature/xxx
git merge main

# 方式二：rebase（历史线性整洁，适合个人分支）
git checkout feature/xxx
git rebase main
# 效果：把你 feature 分支的 commit "搬到" main 的最新 commit 之后
# 历史看起来像：你在最新 main 上一条一条写的，没有分叉

# 交互式 rebase：合并、修改、删除你本地的 commit
git rebase -i HEAD~3  # 整理最近 3 个 commit
```

**merge vs rebase 的区别（可视化）：**

```
merge 结果（有分叉 + merge commit）：     rebase 结果（线性历史）：
*   Merge branch 'feature'              * feat: 功能 B
|\                                      * feat: 功能 A
| * feat: 功能 B                        * main 最新 commit
| * feat: 功能 A                        * ...
* | main 最新 commit
|/
* ...
```

### 10.4 别名（Alias）：少敲键盘

```bash
# 一次设置，终身受益
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"

# 使用：
git co -b new-feature    # = git checkout -b new-feature
git st                   # = git status
git lg                   # = git log --oneline --graph --all
```

### 10.5 二分查找（Git Bisect）：定位 Bug 是什么时候引入的

```bash
# 当你知道 v1.0.0 没问题、当前版本有问题，但不清楚哪个 commit 引入的:
git bisect start
git bisect bad HEAD          # 当前版本是坏的
git bisect good v1.0.0       # v1.0.0 是好的

# Git 自动切到中间某个 commit，你测试后告知结果：
git bisect good              # 这个版本没问题
# 或
git bisect bad               # 这个版本有问题

# 重复几次，Git 自动定位到引入 Bug 的第一个 commit
git bisect reset             # 结束，恢复正常状态
```

---

## 十一、平台差异速查表

### 11.1 术语对照

不同平台对同一件事的叫法不一样，别搞混：

| 概念 | GitHub | GitLab | Gitee (码云) |
|------|--------|--------|-------------|
| 合并请求 | Pull Request (PR) | Merge Request (MR) | Pull Request (PR) |
| CI/CD | GitHub Actions | GitLab CI/CD | Gitee Go |
| 项目看板 | Projects | Issue Boards | 任务看板 |
| 文档站点 | GitHub Pages | GitLab Pages | Gitee Pages |
| 包管理 | GitHub Packages | GitLab Package Registry | — |
| 代码审查 | Review | Review | 代码审查 |
| 代码片段 | Gist | Snippets | 代码片段 |
| 组织/团队 | Organization | Group | 企业/组织 |

### 11.2 仓库地址格式

```bash
# GitHub
git@github.com:用户名/仓库名.git
https://github.com/用户名/仓库名.git

# GitLab（官方托管）
git@gitlab.com:用户名/仓库名.git
https://gitlab.com/用户名/仓库名.git

# Gitee（码云）
git@gitee.com:用户名/仓库名.git
https://gitee.com/用户名/仓库名.git

# 自建 GitLab（企业内部很常见！）
git@gitlab.公司域名.com:用户名/仓库名.git
```

### 11.3 平台鉴权差异

**这是最容易踩坑的地方，不同平台机制不一样：**

| 平台 | 支持的鉴权方式 | 推荐 |
|------|--------------|------|
| **GitHub** | SSH Key、Personal Access Token | SSH Key（日常）/ Token（CI） |
| **GitLab** | SSH Key、Personal Access Token | SSH Key |
| **Gitee** | SSH Key、HTTPS + 密码、私人令牌 | SSH Key |

> **⚠️ GitHub 的重要变化（2021年8月起）：** GitHub 已不再支持 HTTPS + 密码推送。如果你用 HTTPS 地址，必须在密码栏填入 **Personal Access Token（个人访问令牌）**。Token 生成路径：GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token → 勾选 `repo` 权限。

### 11.4 各平台的独特功能和适用场景

| 平台 | 独特优势 | 最适合 |
|------|---------|--------|
| **GitHub** | 🌍 全球最大开源社区、Copilot AI 编程助手、Actions 生态最丰富、Codespaces 云端开发环境 | 开源项目首选、个人作品展示、找工作（绿格子贡献图=简历） |
| **GitLab** | 🏢 自建部署最成熟、内置容器仓库和 K8s 集成、DevOps 全流程一体化 | 企业私有化部署、需要完整 CI/CD 流水线的团队 |
| **Gitee** | 🇨🇳 国内访问速度快（不用翻墙）、中文生态友好、企业版功能完善 | 国内项目、高校课程/比赛、不想折腾网络的用户 |
| **Gitea** | 🪶 极轻量（一个二进制文件）、一键自建、资源占用极小（树莓派都能跑） | 个人/小团队自建 Git 服务 |
| **Bitbucket** | 🔗 深度集成 Jira/Confluence、支持 Mercurial（已弃用） | Atlassian 全家桶用户 |

### 11.5 平台间"跨平台协作"技巧

用一份本地代码同时推送到多个平台：

```
你的本地仓库
    │
    ├── git push github main   → GitHub（给国际社区看）
    └── git push gitee main    → Gitee（给国内用户看）
```

```bash
# 设置
git remote add github git@github.com:用户名/仓库名.git
git remote add gitee git@gitee.com:用户名/仓库名.git

# 日常推送
git push github main
git push gitee main

# 或者写个脚本一键推送
# git push --all origin 会把所有远程当 origin 推，但这里用了不同 remote 名
```

---

## 十二、最佳实践 Checklist

### 12.1 日常操作

- [ ] **开始工作前先 `git pull`** — 避免冲突越积越多
- [ ] **一个分支只做一件事** — 方便审查和回滚，别把登录功能和数据库重构混在一个分支
- [ ] **Commit 要小且聚焦** — 一个 commit 做一件事，方便 `git bisect` 定位 Bug
- [ ] **Commit message 要能看懂** — 遵循 `类型: 描述` 格式，让半年后的自己还能理解
- [ ] **`git add .` 前先 `git status`** — 确认没把密钥、`.env` 等敏感文件加进去
- [ ] **推送前先 `git pull --rebase`** — 保持历史整洁，避免无意义的 merge commit
- [ ] **不推送敏感信息** — 密码、密钥、Token 放到 `.env` 文件并加入 `.gitignore`
- [ ] **不要在 main 分支上直接开发** — 永远用功能分支，main 只接受 merge

### 12.2 协作场景

- [ ] **先开 Issue 再写代码** — 让维护者知道你在做什么，避免跟别人撞车或方向跑偏
- [ ] **PR/MR 描述写清楚** — 做了什么、为什么这样做、怎么测试，附截图更好
- [ ] **保持 Fork 与上游同步** — 定期 `git fetch upstream && git merge upstream/main`
- [ ] **Review 别人的代码要友善** — 指出问题的同时给建设性建议，多用 "建议" 少用 "不对"
- [ ] **收到 Review 意见后及时修改** — 用 `git commit --amend` 或追加 commit，看项目规范

### 12.3 紧急情况速查

| 我想… | 命令 |
|-------|------|
| 撤消工作目录的修改 | `git restore .` |
| 撤消暂存区的 add | `git restore --staged .` |
| 撤消最近的 commit（保留修改） | `git reset --soft HEAD~1` |
| 回滚已推送的 commit（安全） | `git revert HEAD && git push` |
| 暂存当前修改去干别的事 | `git stash` |
| 找回误删的 commit | `git reflog` |
| 看是谁写的这行烂代码 | `git blame 文件名` |
| 把 commit 搬到另一个分支 | `git cherry-pick <commit-hash>` |

---

## 附录 A：完整命令速查表

```bash
# ========== 配置 ==========
git config --global user.name "名字"
git config --global user.email "邮箱"
git config --global init.defaultBranch main
git config --global --list

# ========== 仓库操作 ==========
git init                          # 初始化仓库
git clone <url>                   # 克隆远程仓库
git clone -b <分支名> <url>       # 克隆指定分支

# ========== 日常工作流 ==========
git status                        # 查看状态（最常用！）
git diff                          # 查看未暂存的修改内容
git diff --staged                 # 查看已暂存的修改内容
git add <文件>                    # 添加到暂存区
git add .                         # 添加所有修改
git commit -m "信息"              # 提交
git push                          # 推送到远程
git push -u origin <分支名>       # 推送并建立追踪关系
git pull                          # 拉取 + 自动合并
git pull --rebase                 # 拉取 + 变基合并
git fetch                         # 只拉取，不合并

# ========== 分支 ==========
git branch                        # 查看本地分支
git branch -a                     # 查看所有分支（含远程）
git branch <分支名>               # 创建分支（不切换）
git checkout -b <分支名>          # 创建 + 切换
git switch -c <分支名>            # 创建 + 切换（推荐）
git checkout <分支名>             # 切换分支
git switch <分支名>               # 切换分支（推荐）
git merge <分支名>                # 合并分支到当前分支
git rebase <分支名>               # 变基到目标分支
git branch -d <分支名>            # 删除本地分支
git branch -D <分支名>            # 强制删除本地分支
git push origin --delete <分支名> # 删除远程分支

# ========== 撤销 ==========
git restore <文件>                # 撤销工作目录的修改
git restore --staged <文件>       # 取消暂存（add 的反操作）
git reset --soft HEAD~1           # 撤销 commit，修改回到暂存区
git reset --hard HEAD~1           # ⚠️ 彻底丢弃最近一次 commit
git revert <commit-hash>          # 安全回滚（生成反向 commit）
git stash                         # 暂存当前修改
git stash pop                     # 恢复最近一次暂存
git reflog                        # 救命命令：查看所有操作历史

# ========== 远程仓库 ==========
git remote -v                     # 查看已关联的远程仓库
git remote add <别名> <url>       # 添加远程仓库
git remote remove <别名>          # 删除远程仓库
git remote set-url <别名> <url>   # 修改远程仓库地址

# ========== 历史查看 ==========
git log                           # 完整提交历史
git log --oneline                 # 简洁单行历史
git log --oneline --graph --all   # 图形化分支历史
git blame <文件>                  # 每行代码的作者和时间
git reflog                        # 操作历史（救命用）
git show <commit-hash>            # 查看某次 commit 的详细改动

# ========== 标签 ==========
git tag                           # 查看所有标签
git tag -a v1.0.0 -m "说明"       # 创建附注标签
git push origin --tags            # 推送所有标签到远程
git push origin --delete v1.0.0   # 删除远程标签
```

---

## 附录 B：推荐学习资源

### GUI 工具（新手友好，可以不用记命令）
- [GitHub Desktop](https://desktop.github.com/) — GitHub 官方出品，操作最简洁
- [Sourcetree](https://www.sourcetreeapp.com/) — 功能全面，免费
- [GitKraken](https://www.gitkraken.com/) — 界面精美，有免费版
- [VS Code 内置 Git](https://code.visualstudio.com/docs/sourcecontrol/overview) — 编辑器集成，日常最顺手

### 在线练习（强烈推荐）
- [Learn Git Branching](https://learngitbranching.js.org/) — ⭐ **交互式学习分支操作，可视化每一步的效果**
- [Oh My Git!](https://ohmygit.org/) — 游戏化学习，适合记不住命令的人

### 深入阅读
- [Pro Git（中文版）](https://git-scm.com/book/zh/v2) — Git 官方文档，免费，全面深入
- [GitHub Skills](https://skills.github.com/) — GitHub 官方互动教程
- [GitLab Learn](https://learn.gitlab.com/) — GitLab 官方学习路径

---

> **总结一句话：** Git 是"用"会的，不是"看"会的。找一个小项目，按本文第八节的流程实操一遍——从 `git init` 到 `git push`，再从 Fork 到 PR。踩几个坑、看几次 `reflog`、解决一两次冲突，你就真正掌握了。命令记不住没关系（附录 A 随时查），但第四章的"四个区域"模型一定要理解——那是 Git 一切操作的心智基础。
