# TODO

活跃待办（只放未完成条目；已处理条目移入 `TODO-archive.md`）。条目格式：`- [ ] **T+序号** 内容（记录：YYYY-MM-DD HH:MM）`。

> 个人隐私类待办不写本文件，写 `TODO.local.md`（不进 git，分流规矩见全局 AGENTS.md「待办（TODO）管理」节）。

## 🟠 橙色（边界情况出错、防护缺口、口径不一致可能演化为实际损失）

- [ ] **T6** 落地 `LEADER-WORKER-SCOPE.md` 的工具强制：按该文档第 8 节顺序做——① `extensions/lib/scope-policy.ts` 判定库 + 单测（角色：agentLeader / subLeader / Worker）；② 改 `agent-call.ts` 加派单资格；③ 改 `git-commit-guard.ts` 加仓库归属校验；④ 新增 `cross-project-guard.ts`（先干跑告警一周再切 block）；⑤ 改 `worktree-name-guard.ts` 加 Worker 禁建删 worktree、接 `scope-audit`；⑥ 规则写入全局 `~/.pi/agent/AGENTS.md` 与 `docs/agents-registry.md`（角色与权限部分已于 2026-11-25 先行写入，工具强制部分仍待办）。验收用该文档列出的用例。（记录：2026-11-25 14:21）

### 本机性能治理（2026-11-28 卡顿排查结论）

> 排查数据：uptime 97 天 22 小时、负载 26.79；可用内存 78MB、压缩内存 3.3GB、wired 1.9GB、swap 4.1GB/5GB；10 秒换出 10952 页（约 171MB）；磁盘数据卷 168Gi/228Gi（90% 满，剩 19Gi）。

- [ ] **T13** 重启本机一次，清掉 fseventsd 累积异常与内存碎片：`fseventsd`（PID 357）已跑 97 天、RSS 涨到 951MB（占 11% 物理内存）、持续 70–80% CPU，累计 CPU 时间 12536 分钟（约 209 小时）；wired 内存 1.9GB 也偏高。重启前先确认无在途构建 / 提交 / 长任务。（记录：2026-11-28 19:04）

- [ ] **T14** 关掉两个空转的 pi 面板，立省近 2 个核：PID 63800（ttys001，跑 1 天，CPU 100–137%）、PID 38954（ttys006，跑 1 天 22 小时，累计 CPU 374 分钟）；两者 cwd 均为 `~/Developer/hyperliquid-research-trial`，父链 `pi → zsh → login → cmux.app`，是 cmux 里开着的终端面板（非 tmux 后台会话），当前无人在看。采样栈显示两者都在 `uv__run_timers` 的 JS 定时器回调里空转。需用户确认后关闭（用户终端里的面板不由 AI 自行 kill）。（记录：2026-11-28 19:04）

- [ ] **T15** 不想重启机器时的单点修法：`sudo killall fseventsd`（系统自动拉起，无数据风险），立刻释放 951MB 并降 CPU；需用户输入密码执行。（记录：2026-11-28 19:04）

- [ ] **T16** 清理磁盘缓解换页压力：`~/.pi/agent/sessions` 已累积 1.7GB / 1653 个 `.jsonl` 会话记录，数据卷只剩 19Gi（90% 满）。归档策略二选一——移出到本机其它卷，或按时间保留最近 N 天其余打包压缩；`~/.pi/agent` 属共享目录，动手前按规矩先做飞书快照。（记录：2026-11-28 19:04）

## 🟡 黄色（文档措辞、格式漂移、卫生问题、不影响正确性的优化）

- [ ] **T17** 给 pi 的渲染层提优化：`dist` 里 `stringWidth()` 对每一行文本无缓存地调 `Intl.Segmenter("en", {granularity:"grapheme"})` 逐字素簇算宽度（7912 项 WIDTHS 表逐个比对），中文/emoji 场景成本极高；两个空转样本的采样栈显示每帧整屏重算 + 大量 V8 对象分配 + GC 压力。优化方向：按行缓存宽度结果、内容未变则跳过重算，或用等价的高效宽字符查表替代 Segmenter。属 pi 上游性能问题，需在 pi 仓库提 issue / PR 而非本项目内改。（记录：2026-11-28 19:04）

