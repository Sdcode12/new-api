# 项目长期记忆 — new-api fork

## 仓库与上游关系

| remote | URL | 说明 |
|---|---|---|
| `origin` | `git@github.com:Sdcode12/new-api.git` | 本次二次开发的 fork |
| `upstream` | `https://github.com/Calcium-Ion/new-api.git` | 上游源项目 |

- 根模块：`github.com/QuantumNous/new-api`；`relaykit/` 是独立的第二模块。
- `AGENTS.md` 明确要求：所有自定义实现必须能**低成本合并未来上游变更**。因此 fork 与上游**必须共享提交历史**（同一批 commit 对象），否则 `git merge upstream/main` 会退化成大面积冲突。
- 分支命名约定：工作分支直接用特性名（如 `groupbuy`）。**避免 `feature/xxx` 这类嵌套名**——见下方环境约束。

## 环境约束（cnb.cool 云开发环境，重要）

### 1. GitHub 访问不稳定（重试即可）

**结论修正**：GitHub 不是不可达，而是**时通时断**。首次 `git push` 失败（`Connection reset by 20.205.243.160 port 443`），原样重试一次即成功。所以遇到失败**先重试，不要急着换方案**。

已知情况：
- `origin` 用 SSH，`~/.ssh/config` 把 github.com 指向 `ssh.github.com:443`
- 环境有本地代理 `http_proxy=http://127.0.0.1:65482`；HTTPS 走代理访问 GitHub 曾返回 `CONNECT tunnel failed, response 502`（同样可能是瞬时的）
- **cnb.cool**（HTTP 200，~0.25s）与 **gitee.com** 稳定可达
- 凭据助手只为 `cnb.cool`、`gitee.com` 配了 provider，没有 github——但 SSH 走密钥，不受影响

**推送后的验证方法**（不要只看本地记录）：`git ls-remote origin refs/heads/<branch>`，直接问服务器。本地 remote-tracking ref 可能因下方 quirk 缺失，此时 `git status` 会显示 `[gone]`，属正常现象。

### 2. 嵌套 git ref 无法持久化

git 自己无法在 `.git/refs/` 下创建子目录：`refs/heads/feature/groupbuy` 这类嵌套引用写入后会被撤销，导致 `git commit` 成功（对象与 reflog 都写入）但分支不前进、HEAD 悬空、`git status` 报 "No commits yet"。

- **扁平名正常**（`refs/heads/zz-probe` 可持久）。
- **解法**：手动建目录写松散 ref 再打包（打包后的引用不会再被撤销）。注意**每一层嵌套都要 `mkdir -p`**，例如 `refs/remotes/origin/feature/groupbuy` 需要先建 `origin/feature/`：
  ```bash
  mkdir -p .git/refs/<完整父路径>
  printf '<sha>\n' > .git/refs/<...>/<name>
  git pack-refs --all
  ```
- **每次在该分支 commit/push 后都要检查并重复此步骤**。用 reflog 尾部或 `git ls-remote` 取新 sha。
- **强烈建议改用扁平分支名**（`groupbuy` 而非 `feature/groupbuy`），可彻底规避这类问题。

### 3. 改写提交历史要用安全方法

`git rebase --exec` 在此环境**危险**：它会签出基底提交，导致基底中不存在的目录（如 `.trellis/`、`custom/`）被从磁盘删除且无法写回。

安全替代：**只构造新对象、不碰工作区**——`git cat-file commit` 读原始对象 → 替换 `author`/`committer`/`parent` 行 → `git hash-object -t commit -w --stdin`。之后手动写 ref + `pack-refs`。

## 身份

- `user.name = ssd`，`user.email = lkFdYV6UMOGn6nxWSAD8iB+cnb.c06P9u2VAFA@noreply.cnb.cool`（全局配置在 `C:/Users/lishuaijie/.gitconfig`）。
- 按 `AGENTS.md` 规则，`ssd` 不属于历史核心开发者，**开 PR 时须在正文声明代码由 AI 生成/辅助**。

## 历史遗留的 fork 特有文件

边界建立前，自定义代码直接落在业务包里：`controller/channel_upstream_update.go`、`model/vendor_meta.go`、`service/codex_*.go`。新增自定义特性应进 `custom/`，但不要顺手搬移这些旧文件（搬移本身是巨大的上游 diff）。
