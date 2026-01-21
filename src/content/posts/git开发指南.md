---
title: git开发指南
published: 2026-01-21
description: ''
image: ''
tags: ['学习','Git']
category: 'Git'
draft: false 
lang: ''
---

> 本指南专为开发人员设计，涵盖日常开发、团队协作、CI/CD 流程中必备的 Git 技能。

## 目录

1. [基础概念](#1-基础概念)
2. [日常开发命令](#2-日常开发命令)
3. [分支管理策略](#3-分支管理策略)
4. [团队协作工作流](#4-团队协作工作流)
5. [代码审查 (Code Review)](#5-代码审查-code-review)
6. [冲突解决](#6-冲突解决)
7. [高级操作](#7-高级操作)
8. [Git Hooks 与自动化](#8-git-hooks-与自动化)
9. [大仓库 (Monorepo) 技巧](#9-大仓库-monorepo-技巧)
10. [故障排查与恢复](#10-故障排查与恢复)
11. [最佳实践](#11-最佳实践)
12. [面试高频问题](#12-面试高频问题)

## 1.基础概念

### 1.1 Git 三大区域
![Git三大区域](./image/Git/Git三大区域.png)

### 1.2 核心对象模型
![Gemini_2](./image/Git/Git核心对象模型.png)

### 1.3 引用与指针
![Gemini_3](./image/Git/Git引用与指针.png)

## 2. 日常开发命令

### 2.1 配置设置

```bash
# 全局配置
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"

# 设置默认编辑器
git config --global core.editor "code --wait"

# 设置默认分支名
git config --global init.defaultBranch main

# 配置别名 (提高效率)
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"

# 查看所有配置
git config --list
```

### 2.2 基础工作流

```bash
# 克隆仓库
git clone git@github.com:company/project.git
git clone --depth 1 git@github.com:company/project.git  # 浅克隆，只获取最近一次提交

# 查看状态
git status
git status -s  # 简洁模式

# 添加更改
git add <file>           # 添加单个文件
git add .                # 添加所有更改
git add -p               # 交互式添加 (选择性暂存)
git add -u               # 只添加已跟踪文件的更改

# 提交更改
git commit -m "feat: add login feature"
git commit -am "fix: resolve null pointer"  # 跳过 add，直接提交已跟踪文件
git commit --amend       # 修改最后一次提交
git commit --amend --no-edit  # 修改提交但不改消息

# 查看历史
git log
git log --oneline -10                    # 简洁模式，最近10条
git log --graph --all                    # 图形化显示所有分支
git log --author="name" --since="2024-01-01"  # 按作者和时间过滤
git log -p <file>                        # 查看文件的修改历史
git log --stat                           # 显示每次提交的文件变更统计
```

### 2.3 同步操作

```bash
# 拉取远程更新
git fetch origin                 # 获取远程更新但不合并
git pull                         # fetch + merge
git pull --rebase               # fetch + rebase (推荐，保持线性历史)
git pull --rebase --autostash   # 自动暂存本地修改，rebase 后恢复

# 推送更改
git push origin <branch>
git push -u origin <branch>     # 首次推送并设置上游分支
git push --force-with-lease     # 安全的强制推送 (比 --force 更安全)
git push origin --delete <branch>  # 删除远程分支
```

## 3. 分支管理策略

### 3.1 Git Flow (经典模式)
![Gemini_4](./image/Git/GitFlow经典模式.png)

### 3.2 Trunk-Based Development (大厂主流)
![Gemini_5](./image/Git/大厂主流模式.png)


### 3.3 分支操作命令

```bash
# 创建与切换
git branch <name>                    # 创建分支
git checkout <name>                  # 切换分支
git checkout -b <name>               # 创建并切换
git switch <name>                    # 切换 (新命令)
git switch -c <name>                 # 创建并切换 (新命令)

# 查看分支
git branch                           # 本地分支
git branch -r                        # 远程分支
git branch -a                        # 所有分支
git branch -v                        # 显示最后一次提交
git branch --merged                  # 已合并到当前分支的分支
git branch --no-merged               # 未合并的分支

# 删除分支
git branch -d <name>                 # 删除已合并的分支
git branch -D <name>                 # 强制删除
git push origin --delete <name>      # 删除远程分支

# 重命名分支
git branch -m <old> <new>            # 本地重命名
```



## 4. 团队协作工作流

### 4.1 标准 PR/MR 流程
![Gemini_6](./image/Git/Git标准流程.png)

### 4.2 保持分支最新

```bash
# 方法一：Rebase (推荐，保持线性历史)
git checkout feature/xxx
git fetch origin
git rebase origin/main
# 如有冲突，解决后：
git add .
git rebase --continue
git push --force-with-lease

# 方法二：Merge
git checkout feature/xxx
git merge origin/main
# 解决冲突后正常提交
```

### 4.3 Commit 规范 (Conventional Commits)

```
<type>(<scope>): <subject>

<body>

<footer>
```

| Type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档更新 |
| `style` | 代码格式 (不影响逻辑) |
| `refactor` | 重构 (既不是新功能也不是修复) |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `chore` | 构建/工具相关 |
| `ci` | CI 配置更改 |

**示例**:
```bash
git commit -m "feat(auth): add OAuth2 login support"
git commit -m "fix(api): handle null response from payment service"
git commit -m "perf(db): optimize user query with index"
git commit -m "refactor(order): extract order validation logic"
```

---


## 5. 代码审查 (Code Review)

### 5.1 Review 前的自查清单

```bash
# 查看自己的更改
git diff main...HEAD                 # 与 main 分支的差异
git log main..HEAD --oneline         # 相比 main 新增的提交
git diff --stat main...HEAD          # 变更文件统计

# 自我 Review
git show <commit>                    # 查看单个提交详情
git diff <commit>^..<commit>         # 查看某个提交的更改
```

### 5.2 常见 Review 工具命令

```bash
# 查看文件的每一行最后由谁修改
git blame <file>
git blame -L 10,20 <file>            # 只看第 10-20 行

# 查找引入 bug 的提交
git bisect start
git bisect bad                       # 当前版本有 bug
git bisect good <commit>             # 某个已知没问题的版本
# Git 会自动二分查找，测试后标记 good/bad
git bisect reset                     # 结束 bisect

# 搜索代码
git grep "function_name"             # 在整个仓库搜索
git grep -n "TODO"                   # 显示行号
```

---

## 6. 冲突解决

### 6.1 冲突类型与解决

```bash
# 合并时发生冲突
git merge feature/xxx
# CONFLICT (content): Merge conflict in file.py

# 查看冲突文件
git status
git diff                             # 查看冲突详情

# 解决冲突的标记
<<<<<<< HEAD
当前分支的代码
=======
要合并进来的代码
>>>>>>> feature/xxx
```
```bash
# 解决后
git add <resolved-files>
git merge --continue                 # 或 git commit

# 放弃合并
git merge --abort
```

### 6.2 Rebase 冲突

```bash
git rebase main
# 发生冲突
# 1. 编辑文件解决冲突
# 2. git add <files>
# 3. git rebase --continue
# 重复直到完成

# 放弃 rebase
git rebase --abort

# 跳过当前提交 (慎用)
git rebase --skip
```

### 6.3 使用工具解决冲突

```bash
# 使用合并工具
git mergetool

# VS Code 用户
# 直接在 VS Code 中使用可视化冲突解决器

# 配置合并工具
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

---

## 7. 高级操作

### 7.1 Rebase 操作

```bash
# 变基到目标分支
git rebase main

# 交互式 Rebase (整理提交历史)
git rebase -i HEAD~5                 # 操作最近 5 个提交

# 交互式 Rebase 选项
# pick   = 保留提交
# reword = 保留提交，但修改提交信息
# edit   = 保留提交，但停止以便修改
# squash = 与上一个提交合并，保留两个提交信息
# fixup  = 与上一个提交合并，丢弃这个提交信息
# drop   = 删除提交
```

### 7.2 Cherry-pick

```bash
# 选择性地将某个提交应用到当前分支
git cherry-pick <commit-hash>

# 应用多个提交
git cherry-pick <hash1> <hash2> <hash3>

# 应用一个范围的提交
git cherry-pick <start>..<end>

# 只应用更改，不提交
git cherry-pick -n <commit>

# 解决冲突后继续
git cherry-pick --continue

# 放弃
git cherry-pick --abort
```

### 7.3 Stash 暂存

```bash
# 临时保存工作现场
git stash
git stash save "work in progress"

# 保存包括未跟踪文件
git stash -u

# 查看 stash 列表
git stash list

# 恢复最近的 stash
git stash pop                        # 恢复并删除
git stash apply                      # 恢复但保留

# 恢复指定的 stash
git stash apply stash@{2}

# 删除 stash
git stash drop stash@{0}
git stash clear                      # 清空所有
```

### 7.4 Reset vs Revert

```bash
# Reset - 修改历史 (谨慎使用)
git reset --soft HEAD~1              # 回退提交，保留更改在暂存区
git reset --mixed HEAD~1             # 回退提交，保留更改在工作区 (默认)
git reset --hard HEAD~1              # 回退提交，丢弃所有更改 (危险!)

# Revert - 创建新提交来撤销 (安全)
git revert <commit>                  # 撤销某个提交
git revert HEAD                      # 撤销最近一次提交
git revert <start>..<end>            # 撤销一个范围
git revert -m 1 <merge-commit>       # 撤销合并提交
```

### 7.5 Reflog - 后悔药

```bash
# 查看所有操作历史 (包括已删除的提交)
git reflog

# 恢复误删的提交
git reflog
# 找到要恢复的 commit hash
git checkout <hash>
git checkout -b recovery-branch

# 恢复误删的分支
git reflog
git checkout -b <branch-name> <hash>
```

---

## 8. Git Hooks 与自动化

### 8.1 常用 Hooks

| Hook | 触发时机 | 用途 |
|------|---------|------|
| `pre-commit` | commit 前 | 代码检查、格式化 |
| `commit-msg` | 提交信息编辑后 | 校验提交信息格式 |
| `pre-push` | push 前 | 运行测试 |
| `post-merge` | merge 后 | 安装依赖 |

### 8.2 使用 Husky (推荐)

```bash
# 安装
npm install husky --save-dev
npx husky install

# 添加 pre-commit hook
npx husky add .husky/pre-commit "npm run lint"

# 添加 commit-msg hook
npx husky add .husky/commit-msg 'npx commitlint --edit "$1"'
```

### 8.3 配置 commitlint

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'chore', 'ci']
    ],
    'subject-max-length': [2, 'always', 72]
  }
};
```

---

## 9. 大仓库 (Monorepo) 技巧

### 9.1 Sparse Checkout (稀疏检出)

```bash
# 只检出需要的目录
git clone --filter=blob:none --sparse <url>
cd <repo>
git sparse-checkout set path/to/dir1 path/to/dir2

# 添加更多目录
git sparse-checkout add path/to/dir3
```

### 9.2 Shallow Clone (浅克隆)

```bash
# 只克隆最近的历史
git clone --depth 1 <url>           # 只克隆最近 1 次提交
git clone --depth 100 <url>         # 克隆最近 100 次提交

# 之后需要更多历史
git fetch --unshallow               # 获取完整历史
git fetch --depth 100               # 获取更多历史
```

### 9.3 Git LFS (大文件存储)

```bash
# 安装 Git LFS
git lfs install

# 追踪大文件类型
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "models/*.bin"

# 查看追踪规则
cat .gitattributes

# 查看 LFS 文件
git lfs ls-files
```

### 9.4 Submodules (子模块)

```bash
# 添加子模块
git submodule add <url> path/to/submodule

# 克隆包含子模块的仓库
git clone --recurse-submodules <url>

# 初始化已克隆仓库的子模块
git submodule init
git submodule update

# 更新所有子模块
git submodule update --remote --merge
```

---

## 10. 故障排查与恢复

### 10.1 常见问题

```bash
# 撤销 git add
git reset HEAD <file>
git restore --staged <file>          # 新命令

# 撤销工作区修改
git checkout -- <file>
git restore <file>                   # 新命令

# 修改最后一次提交
git commit --amend

# 找回删除的文件
git checkout HEAD~1 -- <file>

# 找回删除的分支
git reflog
git checkout -b <branch> <sha>

# 清理本地分支
git fetch -p                         # 删除远程已删除的分支引用
git branch -vv | grep 'gone]' | awk '{print $1}' | xargs git branch -d
```

### 10.2 紧急回滚

```bash
# 生产环境回滚 (安全方式)
git revert <bad-commit>
git push origin main

# 如果需要回滚多个提交
git revert --no-commit HEAD~3..HEAD
git commit -m "Revert last 3 commits"
git push

# 强制回滚 (危险，会丢失历史)
git reset --hard <good-commit>
git push --force-with-lease origin main
```

### 10.3 仓库维护

```bash
# 清理不需要的文件
git clean -fd                        # 删除未跟踪的文件和目录
git clean -fxd                       # 包括 .gitignore 忽略的文件

# 垃圾回收
git gc                               # 清理不必要的文件
git gc --aggressive                  # 更彻底的清理

# 查看仓库大小
git count-objects -vH

# 查找大文件
git rev-list --objects --all | git cat-file --batch-check='%(objectname) %(objecttype) %(objectsize) %(rest)' | sort -rnk3 | head -20
```

---

## 11. 最佳实践

### 11.1 提交原则

1. **原子提交**: 每个提交只做一件事
2. **频繁提交**: 小步快跑，避免大提交
3. **有意义的提交信息**: 遵循 Conventional Commits
4. **不提交编译产物**: 使用 `.gitignore`
5. **不提交敏感信息**: 密码、密钥使用环境变量

### 11.2 分支原则

1. **main 分支永远可部署**
2. **从最新的 main 创建特性分支**
3. **特性分支生命周期要短** (1-2 天)
4. **合并前先 rebase/merge main**
5. **使用 Squash Merge 保持历史整洁**

### 11.3 协作原则

1. **提交前拉取最新代码**
2. **使用 `--force-with-lease` 而非 `--force`**
3. **不要修改已推送的历史**
4. **提 PR 前自我 Review**
5. **及时处理 Review 意见**

### 11.4 推荐的 .gitignore

```gitignore
# 编译产物
*.pyc
__pycache__/
*.class
*.o
build/
dist/

# 依赖
node_modules/
venv/
.venv/
vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo

# 系统文件
.DS_Store
Thumbs.db

# 环境配置
.env
.env.local
*.local

# 日志
*.log
logs/

# 测试覆盖率
coverage/
.coverage
htmlcov/
```

---

## 12. 面试高频问题

### Q1: git merge 和 git rebase 的区别？

**Merge**:
- 创建一个新的合并提交
- 保留完整的分支历史
- 更安全，不改变历史

**Rebase**:
- 将提交"移动"到目标分支顶端
- 创建线性历史
- 改变提交历史，需要 force push

```
Merge:                              Rebase:
  main                                main
    │                                   │
    ●───●───●───●                       ●───●───●───●───○───○
        │       │                               (feature commits
        └───○───┘                                rebased on top)
     (merge commit)
```

### Q2: git reset 和 git revert 的区别？

| 特性 | reset | revert |
|-----|-------|--------|
| 操作 | 移动 HEAD | 创建新提交 |
| 历史 | 改变历史 | 保留历史 |
| 适用 | 本地未推送 | 已推送的提交 |
| 安全性 | 低 | 高 |

### Q3: 如何撤销一个已经 push 的提交？

```bash
# 安全方式 - revert
git revert <commit-hash>
git push origin main

# 危险方式 - reset + force push (不推荐)
git reset --hard HEAD~1
git push --force-with-lease origin main
```

### Q4: 如何找到引入 bug 的提交？

```bash
# 使用 git bisect
git bisect start
git bisect bad                  # 当前有 bug
git bisect good v1.0.0          # v1.0.0 没有 bug
# 测试并标记，Git 会二分查找
git bisect good/bad
# 直到找到引入 bug 的提交
git bisect reset
```

### Q5: 解释 HEAD~1 和 HEAD^1 的区别

- `HEAD~n`: 回退 n 代祖先 (沿第一父提交)
- `HEAD^n`: 第 n 个父提交 (用于合并提交)

```
对于普通提交: HEAD~1 = HEAD^1
对于合并提交: HEAD^1 是主分支父提交，HEAD^2 是被合并分支的父提交
```

### Q6: git fetch 和 git pull 的区别？

```bash
git fetch  # 只下载远程数据，不修改工作目录
git pull   # fetch + merge (或 --rebase)
```

### Q7: 如何恢复误删的分支？

```bash
git reflog                      # 查看操作历史
# 找到分支最后一次提交的 hash
git checkout -b <branch-name> <hash>
```

---

## 附录: 命令速查表

```bash
# 配置
git config --global user.name "name"
git config --global user.email "email"

# 基础
git init / clone / status / add / commit / log

# 分支
git branch / checkout / switch / merge / rebase

# 远程
git remote / fetch / pull / push

# 撤销
git reset / revert / restore / checkout --

# 暂存
git stash / stash pop / stash list

# 查看
git diff / show / blame / log --graph

# 高级
git cherry-pick / bisect / reflog
```

---

> 💡 **学习建议**: 
> 1. 先掌握日常开发命令 (Section 2)
> 2. 理解分支策略和团队协作 (Section 3-4)
> 3. 熟练冲突解决 (Section 6)
> 4. 逐步学习高级操作 (Section 7)
> 5. 面试前重点复习 Section 12
