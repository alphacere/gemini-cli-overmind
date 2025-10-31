# Fork 私有仓库和同步工作流指南

## 问题 1：如何把 Fork 变成私有仓库？

### ✅ 可以的！GitHub 支持私有 Fork

从 GitHub 的官方文档来说，**Fork 可以是私有的**。

#### 步骤 1：Fork 时设置为私有

**在 GitHub 网页操作：**

1. 访问 https://github.com/google-gemini/gemini-cli
2. 点击右上角 **Fork** 按钮
3. 在弹出的对话框中：
   - 选择你的账户
   - **勾选 "Copy the `main` branch only"**（可选，只复制 main 分支）
   - **勾选 "Private"** ← 关键步骤
4. 点击 **Create fork**

#### 步骤 2：如果已经 Fork 了，改为私有

**在你的 Fork 仓库设置：**

1. 访问 https://github.com/YOUR_USERNAME/gemini-cli
2. 点击 **Settings** 标签
3. 向下滚动到 **Danger Zone**
4. 点击 **Change repository visibility**
5. 选择 **Make private**
6. 输入仓库名称确认
7. 点击 **I understand the consequences, make this repository private.**

✅ **完成！**现在你的 Fork 是私有的了

---

## 问题 2：源头项目更新后，如何同步到你的 Fork？

### 方案 A：本地同步然后推送（⭐⭐⭐⭐⭐ 推荐）

这是最常用的方式，适合频繁更新的情况。

#### 初次设置（一次性）

```bash
cd /Users/dada/Development/Projects/research/gemini-cli

# 1️⃣ 查看当前 remote 配置
git remote -v
# 应该只显示：
# origin  https://github.com/YOUR_USERNAME/gemini-cli.git (fetch)
# origin  https://github.com/YOUR_USERNAME/gemini-cli.git (push)

# 2️⃣ 添加官方仓库作为 upstream
git remote add upstream https://github.com/google-gemini/gemini-cli.git

# 3️⃣ 验证
git remote -v
# 现在应该显示：
# origin    https://github.com/YOUR_USERNAME/gemini-cli.git (fetch)
# origin    https://github.com/YOUR_USERNAME/gemini-cli.git (push)
# upstream  https://github.com/google-gemini/gemini-cli.git (fetch)
# upstream  https://github.com/google-gemini/gemini-cli.git (push)
```

#### 日常同步（每次官方更新时）

```bash
# 1️⃣ 从官方仓库获取最新代码
git fetch upstream

# 2️⃣ 检查你当前在什么分支
git branch
# * feature/web-refactor
# main

# 3️⃣ 切换到 main 分支
git checkout main

# 4️⃣ 将官方的 main 合并到你的 main
git merge upstream/main
# 或者使用 rebase（保持历史干净）
git rebase upstream/main

# 5️⃣ 推送到你的 Fork（origin）
git push origin main

# 6️⃣ 返回你的开发分支
git checkout feature/web-refactor

# 7️⃣ 将更新的 main 同步到你的开发分支
git rebase main
# 或 git merge main

# 如果有冲突，解决冲突后：
git add .
git rebase --continue
# 或如果用了 merge，直接：
git add .
git commit -m "merge: sync with upstream"

# 8️⃣ 推送开发分支
git push origin feature/web-refactor
```

#### 冲突处理

```bash
# 如果 rebase 时遇到冲突
git rebase upstream/main

# Git 会告诉你哪些文件有冲突
# 编辑冲突文件，选择要保留的部分

# 解决所有冲突后
git add .
git rebase --continue

# 如果想放弃 rebase
git rebase --abort
```

---

### 方案 B：通过 GitHub Web 界面同步（⭐⭐⭐ 更简单）

GitHub 现在提供了一个非常方便的 Web 界面来同步 Fork。

#### 步骤：

1. 访问你的 Fork: https://github.com/YOUR_USERNAME/gemini-cli

2. 你会看到一个类似这样的提示：
   ```
   This branch is X commits behind google-gemini:main
   ```

3. 点击 **Sync fork** 按钮

4. 选择：
   - **Update branch** - 用官方的最新代码更新你的 main

   或者

   - **Compare** - 查看详细差异

5. ✅ 完成！你的 Fork 已同步

**优点**：
- 无需命令行
- 自动处理冲突（如果没有冲突的话）
- 图形化界面清晰

**缺点**：
- 只能同步整个分支
- 复杂冲突需要手动处理

---

### 方案 C：Merge 方式保留本地改动（⭐⭐⭐⭐ 推荐用这个）

如果你想保留你的改动但同时获取官方更新，用 `merge` 比 `rebase` 更安全：

```bash
# 1️⃣ 从官方获取最新
git fetch upstream

# 2️⃣ 切到 main
git checkout main

# 3️⃣ 合并官方 main（保留你的改动）
git merge upstream/main --no-ff
# --no-ff 会创建一个 merge commit，保留分支历史

# 4️⃣ 如果有冲突，解决冲突
# 编辑冲突文件...
git add .
git commit -m "merge: sync with upstream main"

# 5️⃣ 推回你的 Fork
git push origin main

# 6️⃣ 更新开发分支
git checkout feature/web-refactor
git merge main

# 7️⃣ 推送
git push origin feature/web-refactor
```

**区别：**
- `rebase` - 改写历史，线性提交（看起来更干净）
- `merge` - 保留分支历史，产生 merge commit（保留完整历史）

对于你的场景，**推荐用 merge**，因为：
- ✅ 更安全（不改写历史）
- ✅ 易于理解（清晰的合并点）
- ✅ 冲突处理更简单

---

## 工作流总结

### 你的本地仓库结构

```
你的 Fork (私有)
    ↑
    └─ upstream (官方仓库) - 只读
    └─ origin (你的 fork) - 读写

你的本地分支：
- main          ← 与官方同步的主分支
- feature/web-refactor ← 你的开发分支
```

### 完整的周期流程

```bash
# ════════════════════════════════════════════════════════
# 1️⃣ 开发阶段（每天）
# ════════════════════════════════════════════════════════

git checkout feature/web-refactor

# ... 编辑代码 ...

git add .
git commit -m "feat: add web-ui components"
git push origin feature/web-refactor


# ════════════════════════════════════════════════════════
# 2️⃣ 同步阶段（官方更新时，通常是每周）
# ════════════════════════════════════════════════════════

# 方法 A：命令行（推荐）
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 方法 B：GitHub Web 界面
# 访问 Fork 页面，点击 "Sync fork" → "Update branch"


# ════════════════════════════════════════════════════════
# 3️⃣ 更新开发分支
# ════════════════════════════════════════════════════════

git checkout feature/web-refactor
git merge main

# 如果有冲突
git add .
git commit -m "merge: sync web-refactor with latest main"
git push origin feature/web-refactor
```

---

## 私有 Fork 的重要事项

### ✅ 私有 Fork 的优点

1. **隐私保护** - 你的改动和计划不会暴露
2. **自由开发** - 可以进行任何实验性改动
3. **版本控制** - 完整的 Git 历史
4. **团队协作** - 可以邀请特定团队成员查看

### ⚠️ 私有 Fork 的限制

1. **不能 PR 回官方** - 官方看不到你的代码
   - 解决方案：如果要贡献，改为公开或手动创建 PR

2. **无法被发现** - 其他人搜索不到你的 Fork
   - 这正是你想要的（隐私）

3. **GitHub 免费额度** - 私有仓库有 CI/CD 时间限制
   - 公开仓库有无限额度
   - 如果超过限制，需要 GitHub Pro 或团队账户

### 如果想贡献回官方

```bash
# 创建新分支专门用于 PR
git checkout -b upstream-contribution

# 在这个分支上做最小化改动
# ... 编辑代码 ...

# 提交
git push origin upstream-contribution

# 去 GitHub：
# 1. 访问官方仓库：https://github.com/google-gemini/gemini-cli
# 2. 点击 "New pull request"
# 3. 选择：
#    - base: google-gemini/gemini-cli main
#    - compare: YOUR_USERNAME/gemini-cli upstream-contribution
# 4. 创建 PR
```

---

## 命令速查表

```bash
# 查看 remote 配置
git remote -v

# 从官方获取最新代码
git fetch upstream

# 查看官方最新的分支
git branch -r
# 应该显示 upstream/main 等

# 同步 main 分支
git checkout main
git merge upstream/main
git push origin main

# 同步开发分支
git checkout feature/web-refactor
git merge main
git push origin feature/web-refactor

# 查看你的改动相对官方的差异
git diff upstream/main

# 查看提交历史（显示分支关系）
git log --oneline --graph --all

# 清理本地已删除的远程分支
git remote prune origin
git remote prune upstream
```

---

## 常见问题 FAQ

### Q1: 私有 Fork 会被删除吗？

**不会**。GitHub 不会删除任何仓库（除非你手动删除）。

即使官方项目被删除，你的 Fork 也会继续存在。

### Q2: 能看到官方的最新代码吗？

**能**。虽然是私有，但你仍然可以：
- ✅ `git fetch upstream` 获取官方代码
- ✅ 查看官方的 GitHub 项目页面
- ✅ 看到官方的 issues 和 PRs

私有只是指**你的 Fork**不被公开，不是看不到官方。

### Q3: 多少时间同步一次比较好？

**建议**：
- 每周检查一次官方是否有更新
- 有重大更新时立即同步
- 你的开发分支最好一周合并一次 main

```bash
# 快速检查是否有更新
git fetch upstream
git log main..upstream/main --oneline
# 如果没输出，说明没有新更新
```

### Q4: 如果我的改动和官方冲突了怎么办？

**解决步骤**：

```bash
# 1. 尝试合并
git merge upstream/main

# 2. 查看冲突文件
git status

# 3. 打开文件，找到冲突部分（<<<<< ===== >>>>>）
vim packages/cli/src/gemini.tsx
# 选择要保留的部分，删除冲突标记

# 4. 标记为已解决
git add .

# 5. 完成合并
git commit -m "merge: resolve conflicts with upstream"

# 6. 推送
git push origin feature/web-refactor
```

### Q5: 怎样保持本地改动但同步官方代码？

这正是 `merge` 的作用！

```bash
git fetch upstream
git merge upstream/main
# 如果有冲突，手动选择保留哪些改动
```

`merge` 会**保留**你的改动，只是将官方的新代码整合进来。

---

## 推荐的操作频率

```
每周一次（定期维护）：
├─ git fetch upstream
├─ 检查 git log main..upstream/main
├─ 如果有更新，git merge upstream/main
└─ git push origin main

每次开发前：
├─ git checkout feature/web-refactor
├─ git merge main （如果上周同步过）
└─ 开始开发

每次完成一个功能：
├─ git add .
├─ git commit -m "..."
└─ git push origin feature/web-refactor
```

---

## 总结

### 你的设置应该是：

```
┌────────────────────────────────────────┐
│   官方仓库 (公开)                       │
│   github.com/google-gemini/gemini-cli  │
└────────────────┬──────────────────────┘
                 │ (fork)
                 ↓
┌────────────────────────────────────────┐
│   你的 Fork (私有)                      │
│   github.com/YOUR_USERNAME/gemini-cli  │
│   - main (与官方同步)                  │
│   - feature/web-refactor (你的改动)   │
└────────────────┬──────────────────────┘
                 │ (pull)
                 ↓
┌────────────────────────────────────────┐
│   你的本地仓库                          │
│   /Users/dada/.../gemini-cli           │
│   remotes:                             │
│   - origin (你的 Fork)                │
│   - upstream (官方仓库)                │
└────────────────────────────────────────┘
```

### 同步流程：

```
官方更新 → git fetch upstream → git merge upstream/main → git push origin main
```

就这么简单！

