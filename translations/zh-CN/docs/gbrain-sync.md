# 跨机器记忆与 GBrain 同步

gstack 会将大量有用的状态写入 `~/.gstack/` —— 学习记录、回顾、CEO 计划、设计文档、开发者配置文件。默认情况下,当你更换笔记本电脑时,所有这些数据都会丢失。**GBrain 同步**会将精选的子集推送到私有 git 仓库,这样你的记忆就能跨机器跟随你,并可被 GBrain 索引。

## 你能获得什么

- 在机器 A 上工作,在机器 B 上无缝继续。
- 你的学习记录、计划和设计在 GBrain 中可见(如果你使用它)。
- 一个干净的退出方式(`gstack-brain-uninstall`),永远不会触碰你的数据。
- 无守护进程、无系统服务、无后台进程。

## 什么不会离开你的机器

按设计,即使开启同步,这些内容也会保持本地:

- 凭证:`.auth.json`、`auth-token.json`、`sidebar-sessions/`、
  `security/device-salt`、`config.yaml` 中的消费者令牌
- 机器特定状态:Chromium 配置文件、ONNX 模型权重、
  缓存、eval-cache、CDP-profile、一次性提示标记
  (`.welcome-seen`、`.telemetry-prompted`、`.vendoring-warned-*` 等)
- 问题偏好:每台机器的 UX 偏好
  (`question-preferences.json`、`question-log.jsonl`、`question-events.jsonl`)。

确切的允许列表位于 `~/.gstack/.brain-allowlist`。CLI 会管理它;你可以在标记行下方追加自己的条目。

## 首次运行设置(30-90 秒)

```bash
gstack-brain-init
```

该命令:

1. 将 `~/.gstack/` 转换为 git 仓库。
2. 询问远程 URL(默认:`gh repo create --private
   gstack-brain-$USER`)。任何 git 远程都可以 —— GitHub、GitLab、Gitea、
   自托管。
3. 推送仅包含配置的初始提交。
4. 写入 `~/.gstack-brain-remote.txt`(仅 URL,无密钥 ——
   可安全复制到另一台机器)。
5. 将 gstack-brain 仓库作为联合源连接到你的本地 gbrain
   (通过 `gbrain sources add` + `git worktree`),这样 `gbrain search`
   就能索引你同步的学习记录、计划和设计。实现
   位于 `bin/gstack-gbrain-source-wireup`。旧的
   `gstack-brain-reader add --ingest-url ...` HTTP 路径在
   v1.15.1.0 中被移除 —— 它依赖于 gbrain 从未发布的 `/ingest-repo` 端点。

初始化后,**你运行的下一个技能**会问你一个关于隐私模式的问题:

- **所有允许列表内容(推荐)**:学习记录、评审、计划、
  设计、回顾、时间线和开发者配置文件都会同步。
- **仅工件**:计划、设计、回顾、学习记录 —— 跳过
  行为数据(时间线、开发者配置文件)。
- **拒绝**:保持所有内容本地。你可以稍后使用
  `gstack-config set gbrain_sync_mode full` 开启同步。

你的答案会被持久化。不会再次询问。

## 跨机器工作流

在机器 A 上:运行一次 `gstack-brain-init`。就这样 —— 现在每次技能调用都会在其开始和结束边界清空同步队列(每个技能约 200-800 毫秒的网络暂停)。

在机器 B 上:

1. 将 `~/.gstack-brain-remote.txt` 从机器 A 复制到机器 B
   (密码管理器、dotfile 仓库、U 盘 —— 由你决定)。
2. 运行任何 gstack 技能。前导部分会看到 URL 文件并打印:
   ```
   BRAIN_SYNC: brain repo detected: <url>
   BRAIN_SYNC: run 'gstack-brain-restore' to pull your cross-machine memory
   ```
3. 运行 `gstack-brain-restore`。这会克隆仓库,恢复你的
   学习记录/计划/回顾,并重新注册 git 合并驱动程序。
4. 重新输入消费者令牌(它们是机器本地的,不会同步 ——
   `gstack-config set gbrain_token <your-token>`)。
5. 下一个技能:你在机器 A 上昨天的学习记录会浮现。这就是神奇的时刻。

## 状态、健康和队列深度

```bash
gstack-brain-sync --status
```

显示:上次成功推送、待处理队列深度、任何同步阻塞以及当前隐私模式。

每次技能运行都会在前导输出顶部附近打印一行 `BRAIN_SYNC:`。扫描它以查找问题。

## 隐私模式详解

| 模式 | 同步内容 |
|------|----------|
| `off` | 无(默认)。 |
| `artifacts-only` | 计划、设计、回顾、学习记录、评审。跳过时间线 + 开发者配置文件。 |
| `full` | 允许列表中的所有内容,包括行为状态。 |

随时更改:
```bash
gstack-config set gbrain_sync_mode full
gstack-config set gbrain_sync_mode off
```

## 密钥保护

每次提交在离开你的机器之前都会扫描凭证形状的内容。被阻止的模式包括:

- AWS 访问密钥(`AKIA…`)
- GitHub 令牌(`ghp_`、`gho_`、`ghu_`、`ghs_`、`ghr_`、`github_pat_`)
- OpenAI 密钥(`sk-…`)
- PEM 块(`-----BEGIN …-----`)
- JWT(`eyJ…`)
- JSON 中的 Bearer 令牌(`"authorization": "…"`、`"api_key": "…"` 等)

如果扫描命中,同步会停止,队列会被保留,你的前导部分会打印:

```
BRAIN_SYNC: blocked: <pattern-family>:<snippet>
```

修复方法:

1. 检查有问题的文件。
2. 如果匹配是你明确想要同步的内容的误报,运行 `gstack-brain-sync --skip-file <path>` 永久排除该路径。
3. 否则,编辑文件以删除密钥并重新运行任何技能。

在 `~/.gstack/.git/hooks/pre-commit` 有一个纵深防御钩子,如果你手动对仓库执行 `git commit`,它会运行相同的扫描。

## 双机器冲突

如果你在同一天在机器 A 和机器 B 上写入,两者都会推送追加提交。Git 的默认行为会在文件尾部冲突,但 `.jsonl` 和 markdown 文件注册了自定义合并驱动程序:

- JSONL 文件使用排序和去重驱动程序,按 ISO 时间戳排序追加(对于确定性,回退到每行的 SHA-256 哈希)。
- Markdown 工件(回顾、计划、设计)使用联合合并驱动程序,连接双方。

你不应该看到冲突提示。如果看到(真正的语义冲突,比如两台机器编辑同一个计划),git 会停止并提示。

## 跨机器拉取频率

前导部分每 24 小时运行一次 `git fetch` + `git merge --ff-only`(通过 `~/.gstack/.brain-last-pull` 缓存)。你不需要考虑这个 —— 它会在每天第一次技能调用时自动发生。

## 卸载

```bash
gstack-brain-uninstall
```

这会:

- 删除 `~/.gstack/.git/` 和所有 `.brain-*` 配置文件。
- 清除 `gstack-config` 中的 `gbrain_sync_mode`。
- 不会触碰你的学习记录、计划、回顾或开发者配置文件。

添加 `--delete-remote` 也会删除私有 GitHub 仓库(仅 GitHub,使用 `gh repo delete`)。

随时使用 `gstack-brain-init` 重新初始化。

## 故障排除

参见 [gbrain-sync-errors.md](gbrain-sync-errors.md) 了解 gstack-brain 可能打印的每个错误消息的索引,每个都有问题/原因/修复方法。

## 底层原理

关于此功能背后的架构决策(允许列表 vs 拒绝列表、守护进程 vs 前导边界同步、JSONL 合并驱动程序、隐私停止门),请参阅 gstack 计划目录中的[已批准计划](../system-instruction-you-are-working-jaunty-kahn.md)。