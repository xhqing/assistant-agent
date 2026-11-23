# Leader / Worker 职责边界方案

> 状态：**方案稿**（2026-11-15 拟），尚未实施。正式规则生效后应写入 `~/.pi/agent/AGENTS.md` 的「Leader / Worker 职责边界」节（并在 `docs/agents-registry.md` 第 5 条引用）；本文档保留为设计依据与工具强制清单。

## 一、为什么需要这份方案

2026-11-12 的定义已经解决了「谁是 Leader、谁是 Worker」——cwd 落点决定角色：Agent 项目 → Leader，子项目 → Worker。但**没有定义「谁可以写什么」**，也没有任何守卫校验「这个仓库归不归你」。于是越界只能靠自觉，一旦发生，代价落在别人的项目上：改错别人的代码、替别人关闭 Issue、替别人合并 PR 与发版。

本方案补两件事：

1. **写权边界**：用「本家（home）」这个唯一变量界定可写范围。
2. **工具强制**：新增与改造守卫，让越界在命令执行前失败，并直接给出正确通道。

**非目标**：这不是安全边界。同一机器、同一用户下，脚本内部的 `os.remove`、`cd` 后的相对路径、子进程写文件都能绕过文本分析。工具强制要达成的是「**无意越界当场失败**」，不是「防住有意绕过」——那由分支保护、凭证隔离、事后审计去管。

## 二、角色与「本家」

| cwd 落点 | 角色 | 本家 |
|---|---|---|
| 某个 Agent 项目根，或它的任何 worktree | 该 Agent 的 **Leader** | 该 Agent 项目 |
| 某个已登记子项目根，或它的任何 worktree | 该子项目的 **Worker**（归属其 Leader） | 该子项目 |
| 未登记目录、家目录、`/tmp` | **无项目身份** | 无（只有共享区写权） |

**本家是写权的唯一来源**——不看你「想做什么」，只看你「站在哪」。判断顺序固定：

1. 取会话 cwd，若是 git worktree，用 `.git` 里的 `gitdir:` 反查出主仓库；
2. 主仓库名命中 `~/.pi/agent/docs/agents-registry.md` 的「智能体命名注册表·项目列」→ Leader；
3. 命中「子项目映射」表 → Worker，归属列即它的 Leader；
4. 都不命中 → 无项目身份。

## 三、边界矩阵

对象分四类：① 本家 ② 本 Agent 名下子项目 ③ 别人的项目（任何 Agent 项目或其子项目）④ 共享区（`~/.pi/agent`、`~/.dev/private`、`/tmp`、`~/Backups`）。

| 动作 | Leader @ ① | Leader @ ②名下子项目 | Worker @ 自己的子项目 | 任何人 @ ③别人的项目 | ④共享区 |
|---|---|---|---|---|---|
| 读（`ls` / `cat` / `grep` / `git status` / `gh view`） | ✅ | ✅ | ✅ | ✅ | ✅ |
| 改文件 / 新建 / 删除 | ✅ | ❌ 必须派 Worker | ✅ 仅当前 worktree | ❌ | ✅ 受快照守卫 |
| `git add` / `commit` / `push` | ✅ 走 PR | ❌ | ✅ 走 PR | ❌ | — |
| local main 对齐（`fetch` / `merge --ff-only`） | ✅ | ✅ 主工作区 | ❌ | ❌ | — |
| 建 / 删 worktree | ✅ | ✅ | ❌ | ❌ | — |
| 建 PR | ✅ | ❌ | ✅ 自己分支 | ❌ | — |
| **合并 PR** | ✅ 自己的 | ✅ 名下的 | ❌ | ❌ | — |
| **建 Issue / 评论** | ✅ | ✅ | ✅ | ✅（公开，谁都能写） | — |
| **关闭 / 重开 / 改 Issue 正文与标签** | ✅ 自己的 | ✅ 名下的 | ❌ | ❌ | — |
| 打 tag / 发 Release | ✅ | ✅ 名下的 | ❌ | ❌ | — |
| 给子项目派 Worker（`agent_call` 到子项目） | — | ✅ 名下的 | ❌ | ❌ | — |
| 求助另一个 Agent（`agent_call` 到它的 Leader） | ✅ 任意 | ✅ | ✅ | ✅ | — |
| 机器级部署（homebrew 全局包 / launchd / Syncthing / 全局 git 钩子） | 仅 Kit | 仅 Kit | ❌ | ❌ | — |

三条最关键的收敛：

1. **子项目的文件改动一律由 Worker 做。** Leader 对子项目只做 git 层动作（同步 main、建删 worktree、合并 PR）与派活；想改子项目就派 Worker，不存在「就一行、顺手」的例外——留一个例外，工具就强制不了。
2. **Issue 的「写」公开、生命周期归负责人。** 建与评论谁都可以（公开行为，不需要授权）；关闭、重开、改正文、加标签只属于该项目 Leader。理由：关闭代表「项目认为这件事完成了」，那是负责人的判断，不是提案人的判断。
3. **别人的项目只读。** 需要它变，走通道 A（发给它的 Leader）或通道 B（建 Issue 提案），不能自己上手，也不能跳过 Leader 直接给它的 Worker 派活。

## 四、跨项目动作只有三条合法通道

| 通道 | 用途 | 具体做法 |
|---|---|---|
| **A 求助 / 委派** | 让别人做它项目里的事 | `agent_call` 到对方 **Agent 项目根**（Leader），消息自带背景与请求 |
| **B 提案** | 对别人的项目提需求或反馈 | 在该仓库 **建 Issue / 评论**（公开行为，无需授权） |
| **C 用户授权** | 用户当轮明确要求我临时跨项目动手 | 命令末尾加 `# AI_AUTHORIZED_CROSS_PROJECT` |

只有这三条能跨出本家。**没有第四条**——「用户以前授权过」「上次也这么做」「只是个小改动」都不算；授权不跨次复用。

## 五、工具强制

### 5.1 判定库（新增，共享）

`~/.pi/agent/extensions/lib/scope-policy.ts` —— 放 `lib/` 子目录（无 `index.ts`，pi 不会把它当扩展加载，但守卫可以 `import`）。

导出三个函数：

- `roleOf(cwd)` → `{ role: "leader" | "worker" | "none", project, owner }`
- `ownerOf(repoPath)` → 该仓库的负责人 Agent 名（含 worktree 反查）；未登记返回 `undefined`
- `isHome(sessionCwd, targetPath)` → 目标是否落在可写域内（含共享区白名单）

`agent-call.ts` 里已有的 registry 解析与 `gitdir:` 反查逻辑迁到这里，三处守卫共用一份，避免各写一份之后漂移。

### 5.2 `extensions/cross-project-guard.ts`（新增，主力）

拦两类：

| 拦截对象 | 判定 | 行为 |
|---|---|---|
| **文件写**：`write` / `edit` 的 `path`；bash 里的 `>` `>>` 重定向、`rm` / `mv` / `cp` / `tee` / `sed -i` / `truncate` 的目标路径 | 目标不在可写域 | `block`，报错附正确通道 |
| **gh 管理动作**：`gh issue close\|reopen\|edit\|delete\|label`、`gh pr merge\|close`、`gh release create\|delete` | 目标仓库（`-R` 参数，缺省用 cwd）不是本家 | `block`，同上 |

**一律放行**：所有 `gh issue create` / `gh issue comment`、全部 `gh * view` / `list`、只读 shell 命令。

报错文本给出可直接执行的下一条命令，例如：

```text
⛔ 拦下：`~/Developer/quant-agent` 属于 Mark（quant-agent），不是本会话的本家（assistant-agent）。
正确通道：
  A 求助：agent_call name="Mark" dir="/Users/xhq/Developer/quant-agent"
  B 提案：gh issue create -R xhqing/quant-agent
  C 用户当轮授权：命令末尾加 # AI_AUTHORIZED_CROSS_PROJECT
```

### 5.3 `extensions/git-commit-guard.ts`（改造）

现有职责不变（`/commit` 开闸 + 禁裸 commit），**新增一条**：被提交的仓库必须是本家——解析 `git -C <path>` 或 cwd，走 `ownerOf()`，不一致即 `block`。这一条单独就能拦住「改别人的 worktree 并提交」。

### 5.4 `extensions/agent-call.ts`（改造）

现有 dir 合法性校验保留，新增**派单资格**：

- 发给任意 **Agent 项目根**（Leader）→ 放行（那是协作请求，不是派活）；
- 发给**子项目**（派 Worker）→ 调用者必须是该子项目的 Leader，否则拒绝并提示「改发它的 Leader」。

这条堵的就是「跳过项目 Leader、直接给它的 Worker 派活」。

### 5.5 `extensions/worktree-name-guard.ts`（改造）

现有命名校验保留，新增：**Worker 会话的 `git worktree add` / `remove` / `move` 一律 `block`**（建删 worktree 是 Leader 的活；现在只校验名字，不校验角色）。

### 5.6 `patch/scope-audit/scope-audit.py`（新增，第二道防线）

扫最近 N 天的 git 提交、PR、Issue 操作，按「操作者会话归属 vs 仓库归属」列出越界项。理由：文本分析拦不住脚本内部操作，事后审计补这个洞。接进会话收尾巡检（`settle-repo-check` 同批执行），只报告、不自动处置。

### 5.7 守卫职责划分（避免重复与遗漏）

| 守卫 | 管什么 | 不管什么 |
|---|---|---|
| `git-main-guard` | main 上的 `add` / `commit`、push 到 main | 仓库归属 |
| `git-commit-guard` | 开闸、禁裸 commit、**仓库归属（新增）** | 分支归属 |
| `cross-project-guard` | 跨项目文件写、`gh` 管理动作 | 本家内的写（不重复管） |
| `agent-call` | dir 合法性、**派单资格（新增）** | 消息内容 |
| `worktree-name-guard` | worktree 命名、**Worker 禁建删（新增）** | Leader 的命名外校验 |
| `public-text-guard` | 公开文本的敏感扫描 | 操作权限 |
| `snapshot-guard` | 共享目录写前快照 | 项目仓库 |

## 六、边界与例外

- **只读全放行**：别人项目的代码、Issue、PR、diff 都可以读，不设限。
- **共享区照旧**：`~/.pi/agent` 与 `~/.dev/private` 任何会话可写，但受 `snapshot-guard` 快照前置约束；`/tmp`、`~/Backups`、`~/Downloads` 不受限。
- **worktree 是正常战场**：Worker 在自己 worktree 里改、提交、push、建 PR 全部放行，不影响现有自动流程；add-mode / commit-mode 的手动暂停照旧生效。
- **误报**：守卫按命令文本解析路径，`cd` 后相对路径、脚本内部删除、`sh -c` 嵌套等拦不住，属于已知局限；用 `scope-audit` 兜底，文档与报错文案里如实声明「下限防线」。
- **逃生门**：`# AI_AUTHORIZED_CROSS_PROJECT` 只在用户当轮明确要求跨项目动手时使用，且回复里必须写明授权来源与本轮范围。

## 七、方案要拦的四类失效形态

1. Leader 直接改某个子项目的文件并提交（应改为：派 Worker）。
2. 跳过项目 Leader，直接给它的 Worker 派活（应改为：发给 Leader）。
3. 关闭 / 重开别人项目的 Issue（应改为：在该 Issue 里评论，由负责人决定）。
4. 合并别人项目的 PR，或给它打 tag 发 Release（应改为：通道 A）。

## 八、落地顺序与验收

| 步骤 | 内容 | 风险 |
|---|---|---|
| 1 | `lib/scope-policy.ts` + 判定单测（4 个 Agent 项目、2 个子项目、2 个 worktree 各命中一次） | 低 |
| 2 | 改 `agent-call.ts`（改动最小） | 低 |
| 3 | 改 `git-commit-guard.ts` 加归属校验 | 中（可能误伤主工作区对齐，需单测覆盖 `git -C` 与裸命令两种形态） |
| 4 | 新增 `cross-project-guard.ts` | 中（路径解析误报是主要风险，先干跑告警一周再切 block） |
| 5 | 改 `worktree-name-guard.ts`、接 `scope-audit` | 低 |
| 6 | 规则写入 `~/.pi/agent/AGENTS.md` 新节 + `docs/agents-registry.md` 引用 | 低 |

**验收用例**（必须全部通过，取真实目录）：

- ✅ Worker 在自己的 worktree 里 `edit` + `git commit` + `gh pr create` → 放行
- ✅ Leader 在同名子项目主工作区执行 `git fetch` / `merge --ff-only` → 放行
- ❌ Leader 在该子项目里 `edit` 任意文件 → 拦下，提示派 Worker
- ❌ 任一 Agent 会话 `git commit` 在别人的仓库 worktree 里 → 拦下
- ❌ 向别人的子项目 `agent_call` 派 Worker → 拦下，提示发给它的 Leader
- ❌ 关闭别人项目的 Issue / 合并别人项目的 PR → 拦下
- ✅ 任何人 `gh issue create` / `gh issue comment` 在别人仓库 → 放行
- ✅ Worker 执行 `git worktree add` → 拦下

## 九、默认判断（有不同意见直接改）

- **Leader 完全不许改子项目文件**，包括「就一行、顺手」——一律派 Worker。这是唯一能工具强制的划法。
- **机器级部署统一归 Kit**（homebrew 全局包、launchd、Syncthing、全局 git 钩子），各项目只负责自己的构建产物与发布包。
- **Issue 关闭权只给 Leader 与用户**，Worker 即使自己建的 Issue 也不关。
