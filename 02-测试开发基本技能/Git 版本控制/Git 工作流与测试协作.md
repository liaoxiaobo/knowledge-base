---
tags:
  - 实战
  - Git
---
# Git 工作流与测试协作

> 来源：`source/_posts/git_tool_note.md`  
> 整理日期：2026-06-16  
> 适用场景：测试代码版本管理、与开发协作、CI/CD 触发、测试分支策略

---

## 一、Git 核心工作流

```text
工作区 → git add → 暂存区 → git commit → 本地仓库 → git push → 远程仓库
```

| 命令 | 作用 | 测试场景 |
|---|---|---|
| `git clone` | 克隆远程仓库 | 初始化测试环境 |
| `git pull` | 拉取并合并远程更新 | 同步最新代码 |
| `git fetch` | 获取远程分支信息，不合并 | 查看远程状态 |
| `git add` | 添加改动到暂存区 | 准备提交测试代码 |
| `git commit` | 提交到本地仓库 | 保存测试用例变更 |
| `git push` | 推送到远程仓库 | 共享测试代码 |
| `git status` | 查看当前状态 | 确认改动范围 |
| `git log` | 查看提交历史 | 追溯变更来源 |

---

## 二、分支管理

### 2.1 常用分支操作

```bash
# 创建并切换分支
git checkout -b feature/add-login-tests

# 推送本地分支到远端
git push -u origin feature/add-login-tests

# 切换分支
git checkout main

# 删除本地分支
git branch -d feature/add-login-tests

# 删除远端分支
git push origin --delete feature/add-login-tests
```

### 2.2 测试开发推荐分支策略

| 分支 | 用途 | 生命周期 |
|---|---|---|
| `main` / `master` | 稳定主干 | 长期 |
| `feature/*` | 新功能测试用例开发 | 随功能开发周期 |
| `bugfix/*` | 缺陷修复测试 | 随缺陷修复周期 |
| `release/*` | 回归测试、发布验证 | 发布期间 |
| `hotfix/*` | 线上紧急修复验证 | 紧急，快速合并 |

**建议**：测试代码与产品代码使用相同分支模型，测试分支随对应功能分支创建和合并。

---

## 三、合并代码

### 3.1 git merge

```bash
git checkout main
git pull origin main
git merge feature/add-login-tests
git push origin main
```

### 3.2 git cherry-pick

当只需要把某个提交应用到当前分支，而不想合并整个分支时：

```bash
git checkout release/v1.2
git cherry-pick abc1234  # 某次测试修复的 commit id
git push origin release/v1.2
```

---

## 四、冲突处理

### 4.1 保留本地修改并拉取远端

```bash
git stash
git pull
git stash pop
# 手动解决冲突后
git add .
git commit -m "merge: resolve conflict"
```

### 4.2 放弃本地修改，使用远端版本

```bash
git reset --hard HEAD
git pull
```

### 4.3 cherry-pick 冲突

```bash
# 解决冲突后
git add .
git cherry-pick --continue

# 或放弃
git cherry-pick --abort
```

---

## 五、提交规范

### 5.1 Commit Message 格式

```text
<类型>(<范围>): <简短描述>

<详细说明>
```

| 类型 | 说明 | 示例 |
|---|---|---|
| `feat` | 新功能/新用例 | `feat(test): 添加登录接口异常场景用例` |
| `fix` | 修复 | `fix(test): 修复订单断言失败` |
| `refactor` | 重构 | `refactor(utils): 提取公共请求封装` |
| `docs` | 文档 | `docs(readme): 更新测试执行说明` |
| `chore` | 杂项 | `chore(ci): 调整 pytest 触发条件` |

### 5.2 原子提交原则

- 一个 commit 只做一件事。
- 不要把测试代码修改和配置文件调整混在一个 commit。
- 提交前检查是否包含调试代码、临时文件。

---

## 六、Git Hooks 与 CI 联动

### 6.1 常用 Hooks

| Hook | 触发时机 | 测试应用 |
|---|---|---|
| `pre-commit` | 提交前 | 运行代码格式检查、静态分析 |
| `pre-push` | 推送前 | 运行单元测试、冒烟测试 |
| `post-merge` | 合并后 | 安装依赖、清理缓存 |

### 6.2 pre-commit 示例

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 运行格式化检查
black --check tests/

# 运行静态检查
flake8 tests/

# 运行快速冒烟测试
pytest tests/smoke/ -q
```

---

## 七、版本回滚

### 7.1 回滚到指定提交

```bash
# 本地回滚（丢弃之后所有提交）
git reset --hard abc1234

# 强制推送到远端（谨慎使用）
git push --force
```

### 7.2 撤销某次提交但保留改动

```bash
git revert abc1234
git push
```

---

## 八、测试协作实践

1. **测试代码与产品代码同仓库**：便于版本对应和 CI 触发。
2. **提交前跑本地快速测试**：避免把失败用例推送到 CI。
3. **PR/MR 模板化**：要求填写测试范围、影响点、执行结果。
4. **标签管理**：用 `git tag` 标记每次发布的测试基线。
5. **定期清理废弃分支**：避免分支过多造成混乱。

---

## 九、关联文档

- [[Shell 脚本编程]]
- 3.4 CI/CD 集成
- 7.1 测试流程规范