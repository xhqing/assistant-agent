# TODO

活跃待办（只放未完成条目；已处理条目移入 `TODO-archive.md`）。条目格式：`- [ ] **T+序号** 内容（记录：YYYY-MM-DD HH:MM）`。

> 个人隐私类待办不写本文件，写 `TODO.local.md`（不进 git，分流规矩见全局 AGENTS.md「待办（TODO）管理」节）。

## 🟠 橙色（边界情况出错、防护缺口、口径不一致可能演化为实际损失）

- [ ] **T6** 落地 `LEADER-WORKER-SCOPE.md` 的工具强制：按该文档第 8 节顺序做——① `extensions/lib/scope-policy.ts` 判定库 + 单测；② 改 `agent-call.ts` 加派单资格；③ 改 `git-commit-guard.ts` 加仓库归属校验；④ 新增 `cross-project-guard.ts`（先干跑告警一周再切 block）；⑤ 改 `worktree-name-guard.ts` 加 Worker 禁建删 worktree、接 `scope-audit`；⑥ 规则写入全局 `~/.pi/agent/AGENTS.md` 与 `docs/agents-registry.md`。验收用该文档列出的 8 条用例。（记录：2026-11-15 20:40）

