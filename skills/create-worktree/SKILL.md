---
name: create-worktree
description: Use when the user wants to create a new worktree for development, feature work, bug fixes, or any isolated workspace. Triggers on phrases like "create worktree", "start worktree", "new branch", "隔离开发", "创建工作区". Accepts an optional branch name as argument.
---

# Create Worktree

## 角色

本地并发开发的工作区管理 Agent。利用 `git worktree` 实现物理级别的代码和环境隔离。**绝对禁止**篡改或污染主仓库的未提交文件、暂存区或当前状态。

**绝对纪律**：必须在项目根目录下的 `.worktree/` 文件夹内集中管理新工作区，且**必须使用软链接（Symlink）复用主库的依赖环境**，绝对禁止重新全量拉取或安装依赖！

## Usage

```
/create-worktree <target-branch> [base-branch]
```

不带参数时，询问目标分支名和基准底座（默认 main）。

---

## 状态机流程（严格按步骤执行）

### 状态 0：主库防污染准备（极其重要）

1. 检查主仓库根目录的 `.gitignore` 文件
2. **安全卡点**：如果 `.gitignore` 中没有 `.worktree/`，必须自动追加该行。绝对禁止在未忽略该目录的情况下创建嵌套工作树！
3. 规划新工作区路径：`./.worktree/<Target_Branch的无斜杠名称>`

### 状态 1：执行 Worktree 创建

#### 👉 分支 A：目标分支已在远端或本地存在

```bash
git fetch origin <Target_Branch>
git worktree add .worktree/<Target_Branch> <Target_Branch>
```

#### 👉 分支 B：需要基于底座创建全新分支

```bash
git worktree add -b <Target_Branch> .worktree/<Target_Branch> <Base_Branch>
```

### 状态 2：极速环境复用（软链接与配置拷贝）

自动跳转到新目录 `cd .worktree/<Target_Branch>`，执行：

1. **私有配置拷贝**：将主库（`../../` 目录）的 `.env`、`.env.local`、`.cursorrules`、`CLAUDE.md` 等非追踪文件精准复制到当前工作区

2. **环境软链接（核心）**：
   - 如果主库存在 Python 虚拟环境（`.venv`）：`ln -s ../../.venv .venv`
   - 如果主库存在前端依赖（`node_modules`）：`ln -s ../../node_modules node_modules`
   - 其他可复用的依赖目录同理

3. **安全拦截**：绝对禁止在此状态下主动运行 `npm install`、`pip install` 等耗时装包命令。完全依赖主库的软链接环境。

---

## 最终输出

执行期间保持静默。完全就绪后输出：

```
### 📂 极速分工作区 (Worktree) 创建就绪
- **项目与分支**：<Project> / <Target_Branch>
- **隔离路径**：`.worktree/<Target_Branch>`
- **复用环境情况**：(成功软链接了哪些虚拟环境或 node_modules？复制了哪些配置？)
- **快速跳转**：cd .worktree/<Target_Branch>
```

## 清理规则

| 场景 | 行为 |
|---|---|
| `.worktree/` 不在 `.gitignore` | 自动追加 `.worktree/` 后继续 |
| git worktree add 失败 | 报错并中止 |
| 主库无 `.venv` 或 `node_modules` | 跳过对应软链接，报告缺失 |
| Target_Branch 含 `/` | 路径中替换为 `-`