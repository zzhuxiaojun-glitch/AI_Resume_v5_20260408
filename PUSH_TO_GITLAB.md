# 推送代码到 GitLab 指南

> 目标仓库：`https://git.keiten-jp.com/hr_bot/hr_frontend`  
> 当前已绑定：GitHub `origin`（不受影响，两个远端并存）

---

## 核心概念：Git 支持多个远端（remote）

```
本地仓库
  ├── origin  → GitHub（已有）
  └── gitlab  → git.keiten-jp.com（新增）
```

两个远端互不干扰，可以分别 push。

---

## 完整步骤

### 第一步：在 GitLab 上创建仓库

**方法 A：网页操作（推荐）**

1. 浏览器打开 `https://git.keiten-jp.com`，登录账号
2. 点击左上角 **New project**
3. 选择 **Create blank project**
4. 填写：
   - Project name: `hr_frontend`
   - Namespace（群组）: `hr_bot`
   - Visibility: `Private`
   - **取消勾选** "Initialize repository with a README"（因为本地已有代码）
5. 点击 **Create project**

**方法 B：API 创建（命令行）**

```bash
curl -X POST \
  -H "PRIVATE-TOKEN: <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"hr_frontend","namespace_id":247,"visibility":"private","initialize_with_readme":false}' \
  "https://git.keiten-jp.com/api/v4/projects"
```

---

### 第二步：添加 GitLab 为第二个远端

```bash
# 添加名为 gitlab 的远端（用 token 认证，避免每次输密码）
git remote add gitlab https://<your-token>@git.keiten-jp.com/hr_bot/hr_frontend.git

# 确认两个远端都存在
git remote -v
```

预期输出：
```
gitlab  https://<token>@git.keiten-jp.com/hr_bot/hr_frontend.git (fetch)
gitlab  https://<token>@git.keiten-jp.com/hr_bot/hr_frontend.git (push)
origin  https://github.com/zzhuxiaojun-glitch/AI_Resume_v5_20260408.git (fetch)
origin  https://github.com/zzhuxiaojun-glitch/AI_Resume_v5_20260408.git (push)
```

---

### 第三步：推送代码

```bash
# 首次推送，用 -u 绑定追踪分支
git push -u gitlab main
```

---

### 之后日常推送

```bash
# 推到 GitHub（原有流程不变）
git push origin main

# 推到 GitLab
git push gitlab main

# 同时推两个（可设置默认行为，见下方进阶）
```

---

## 进阶：一条命令同时推两个远端（可选）

如果希望 `git push` 默认同时推 GitHub 和 GitLab：

```bash
# 给 origin 额外添加一个 push URL
git remote set-url --add --push origin https://<token>@git.keiten-jp.com/hr_bot/hr_frontend.git

# 验证
git remote -v
```

此后 `git push` 会同时推送到两个地址。  
（注意：fetch 仍然只从 GitHub 拉取）

---

## 常见问题

**Q：已绑定 GitHub 会冲突吗？**  
A：不会。`origin` 和 `gitlab` 是两个独立的远端，互不影响。

**Q：Token 放在 URL 里安全吗？**  
A：Token 会存在 `.git/config` 里（本地文件，不会被提交）。如果介意，可以不放在 URL 里，每次 push 时手动输入用户名和密码/Token。

**Q：GitLab 上的仓库需要是空的吗？**  
A：首次推送时需要是空仓库（不能有 README 等文件），否则会有冲突。如果不小心初始化了，用 `git push --force` 强制覆盖（仅限第一次设置时）。
