# 在 GStack 中使用 GBrain

你的编码代理,拥有真正能保留的记忆。

[GBrain](https://github.com/garrytan/gbrain) 是为 AI 代理设计的持久化知识库。它存储代理学到的内容、你做出的决策、哪些有效哪些无效,并让代理按需搜索所有这些信息。GStack 为你提供了一条从零到"gbrain 正在运行,我的代理可以调用它"的一键路径——涵盖本地试用、团队共享以及介于两者之间的所有场景。

这是完整指南:每个场景、每个标志、每个辅助工具、每个故障排除步骤。快速介绍请参阅 [README 的 GBrain 部分](README.md#gbrain--persistent-knowledge-for-your-coding-agent)。错误代码和同步相关问题请参阅 [docs/gbrain-sync.md](docs/gbrain-sync.md)。

---

## 一键安装

```bash
/setup-gbrain
```

就是这样。该技能会检测你的当前状态,最多询问三个问题,并引导你完成安装、初始化、Claude Code 的 MCP 注册以及每个仓库的信任策略。在一台什么都没安装的全新 Mac 上,五分钟内完成。在已经设置了某些内容的 Mac 上只需几秒钟(它会检测现有状态并跳过已完成的工作)。

## 三条路径

当技能询问"你的大脑应该存放在哪里?"时,你选择其中一条。

### 路径 1:Supabase,你已经有连接字符串

适用于:你(或队友的云代理)已经配置了 Supabase 大脑,你希望这台本地机器使用相同的数据。

**发生什么:**粘贴 Session Pooler URL(Settings → Database → Connection Pooler → Session → 复制 URI,端口 6543)。技能会在关闭回显的情况下读取它,向你显示一个脱敏预览(`aws-0-us-east-1.pooler.supabase.com:6543/postgres` — 主机可见,密码已屏蔽),通过 `GBRAIN_DATABASE_URL` 环境变量将其传递给 `gbrain init`,该 URL 永远不会写入 argv 或你的 shell 历史记录。

**信任警告:**粘贴此 URL 会授予你的本地 Claude Code 对共享大脑中每个页面的完全读写访问权限。如果这不是你想要的信任级别,请改选 PGLite 本地(路径 3),并接受大脑是分离的。

### 路径 2a:Supabase,自动配置新项目

适用于:全新的 Supabase 账户,你想要一个全新的项目,无需任何点击。

**发生什么:**你粘贴一个 Supabase Personal Access Token(PAT)。技能会首先向你显示范围披露 — *该令牌授予对你 Supabase 账户中每个项目的完全访问权限,而不仅仅是我们即将创建的项目*。它列出你的组织,询问哪个组织和哪个区域(默认 `us-east-1`),生成数据库密码,调用 `POST /v1/projects`,每 5 秒轮询一次 `GET /v1/projects/{ref}` 直到项目状态为 `ACTIVE_HEALTHY`(180 秒超时),获取 pooler URL,将其传递给 `gbrain init`。端到端:约 90 秒。

结束时:明确提醒在 https://supabase.com/dashboard/account/tokens 撤销 PAT。技能已经从内存中丢弃了它。

**如果你在配置过程中按 Ctrl-C:**SIGINT 陷阱会打印你正在进行的项目 ref + 恢复命令。你可以在 Supabase 仪表板删除孤立项目,或运行 `/setup-gbrain --resume-provision <ref>` 从中断处继续。

### 路径 2b:Supabase,手动创建

适用于:你宁愿自己在 supabase.com 上点击操作,也不想粘贴 PAT。

**发生什么:**技能会引导你完成四个手动步骤(注册 → 新项目 → 等待约 2 分钟 → 复制 Session Pooler URL),然后从路径 1 的粘贴步骤接管。与路径 1 相同的安全处理。

### 路径 3:PGLite 本地

适用于:先试用、无账户、无云、无共享。或者专用的"这台 Mac 的大脑",与任何云代理保持隔离。

**发生什么:**`gbrain init --pglite`。大脑存放在 `~/.gbrain/brain.pglite`。无网络调用。30 秒内完成。

如果你只是想在承诺使用云之前感受一下 gbrain 是什么样的,这是最佳首选。你以后随时可以使用 `/setup-gbrain --switch` 进行迁移。

## Claude Code 的 MCP 注册

默认情况下,技能会询问"为 Claude Code 提供 gbrain 的类型化工具界面?"如果你同意,它会运行:

```bash
claude mcp add gbrain -- gbrain serve
```

这会向 Claude Code 注册 gbrain 的 stdio MCP 服务器。现在 `gbrain search`、`gbrain put_page`、`gbrain get_page` 等会在每个会话中显示为一流工具,而不是 bash shell 调用。

**如果 `claude` 不在 PATH 中**,技能会优雅地跳过 MCP 注册,并提供手动注册提示。CLI 解析器仍然可以从任何调用 `gbrain` 的技能中工作 — MCP 是升级,不是先决条件。

**其他本地代理**(Cursor、Codex CLI 等)需要自己的 MCP 注册。该技能在 v1 中针对 Claude Code;其他主机可以在自己的 MCP 配置中手动注册 `gbrain serve`。

## 每个远程仓库的信任策略(三元组)

你机器上的每个仓库都会得到一个策略决策:**read-write**、**read-only** 或 **deny**。

- **read-write** — 你的代理可以从此仓库的上下文中 `gbrain search`,并将新页面写回大脑。你自己项目的默认设置。
- **read-only** — 你的代理可以搜索大脑,但永远不会从此仓库的会话中写入新页面。非常适合多客户顾问:搜索共享大脑,但在客户 B 的仓库中工作时不要用客户 A 的代码污染它。
- **deny** — 完全没有 gbrain 交互。该仓库对 gbrain 工具不可见。

技能会在你第一次在那里运行 gstack 技能时为每个仓库询问一次。之后决策是持久的 — 同一 git 远程仓库的每个工作树 + 分支共享相同的策略,所以你只需设置一次,它就会跟随你。

SSH 和 HTTPS 远程变体会折叠为同一个键:`https://github.com/foo/bar.git` 和 `git@github.com:foo/bar.git` 是同一个仓库。

**更改策略:**

```bash
/setup-gbrain --repo      # 仅为此仓库重新提示

# 或直接:
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy set "github.com/foo/bar" read-only
```

**查看所有策略:**

```bash
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy list
```

存储:`~/.gstack/gbrain-repo-policy.json`,模式 0600,带有架构版本,因此未来的迁移保持确定性。

## 稍后切换引擎

选择了 PGLite 现在想加入团队大脑?一条命令:

```bash
/setup-gbrain --switch
```

技能会运行包装在 `timeout 180s` 中的 `gbrain migrate --to supabase --url "$URL"`。迁移是双向的(Supabase → PGLite 也可以)且无损 — 页面、块、嵌入、链接、标签和时间线全部复制。你的原始大脑会作为备份保留。

**如果迁移挂起:**另一个 gstack 会话可能正在通过其序言的 `gstack-brain-sync` 调用持有源大脑的锁。超时会在 3 分钟时触发并显示可操作的消息。关闭其他工作区并重新运行。

## GStack 内存同步(一个独立的关注点)

这与 gbrain 本身不同。你的 gstack 状态(`~/.gstack/` — 学习内容、计划、回顾、时间线、开发者配置文件)默认是机器本地的。"GStack 内存同步"可选地将经过策划、经过秘密扫描的子集推送到私有 git 仓库,这样你的内存就会跟随你跨机器 — 而且,如果你正在运行 gbrain,该 git 仓库也可以在那里被索引。

使用以下命令启用:

```bash
gstack-brain-init
```

你会收到一次性隐私提示:**所有允许列表内容** / **仅工件**(计划、设计、回顾、学习内容 — 跳过时间线等行为数据)/ **关闭**。每次技能运行都会在开始和结束时同步队列 — 没有守护进程,没有后台进程。

类似秘密的内容(AWS 密钥、GitHub 令牌、PEM 块、JWT、bearer 令牌)在离开你的机器之前就被阻止同步。

**在新机器上:**复制 `~/.gstack-brain-remote.txt`,运行 `gstack-brain-restore`,昨天的学习内容就会出现在今天的笔记本电脑上。

完整指南:[docs/gbrain-sync.md](docs/gbrain-sync.md)。错误索引:[docs/gbrain-sync-errors.md](docs/gbrain-sync-errors.md)。

`/setup-gbrain` 会在初始设置结束时为你提供连接此功能的选项 — 这是另一个 AskUserQuestion,它与相同的私有仓库基础设施集成。

## 清理孤立项目

如果你在配置过程中按了 Ctrl-C,在确定一个名称之前尝试了三个不同的名称,或者以其他方式积累了你不使用的类似 gbrain 的 Supabase 项目,有一个子命令可以处理:

```bash
/setup-gbrain --cleanup-orphans
```

技能会重新收集 PAT(一次性,之后丢弃),列出你 Supabase 账户中名称以 `gbrain` 开头且 ref 与你活动的 `~/.gbrain/config.json` pooler URL 不匹配的每个项目。对于每个孤立项目,它会逐个询问:*"删除孤立项目 `<ref>`(`<name>`,创建于 `<date>`)?"* — 没有批处理,没有"全部删除"快捷方式。活动大脑永远不会被提供删除。

## 命令 + 标志参考

### `/setup-gbrain` 入口模式

| 调用 | 作用 |
|---|---|
| `/setup-gbrain` | 完整流程:检测状态、选择路径、安装、初始化、MCP、策略、可选的内存同步 |
| `/setup-gbrain --repo` | 仅翻转当前仓库的每个远程信任策略 |
| `/setup-gbrain --switch` | 迁移引擎(PGLite ↔ Supabase)而不重新运行其他步骤 |
| `/setup-gbrain --resume-provision <ref>` | 恢复在轮询期间中断的路径 2a 自动配置 |
| `/setup-gbrain --cleanup-orphans` | 列出 + 逐个删除孤立的 Supabase 项目 |

### Bin 辅助工具(用于脚本)

| Bin | 用途 |
|---|---|
| `gstack-gbrain-detect` | 将当前状态作为 JSON 输出:PATH 上的 gbrain、版本、配置引擎、doctor 状态、同步模式 |
| `gstack-gbrain-install` | 检测优先的安装程序(探测 `~/git/gbrain`、`~/gbrain`,然后全新克隆)。有 `--dry-run` 和 `--validate-only` 标志。PATH 遮蔽检查以退出码 3 退出并显示修复菜单。 |
| `gstack-gbrain-lib.sh` | 被 source,不执行。提供 `read_secret_to_env VARNAME "prompt" [--echo-redacted "<sed-expr>"]` |
| `gstack-gbrain-supabase-verify` | 结构化 URL 检查。以退出码 3 拒绝直连 URL(`db.*.supabase.co:5432`) |
| `gstack-gbrain-supabase-provision` | 管理 API 包装器。子命令:`list-orgs`、`create`、`wait`、`pooler-url`、`list-orphans`、`delete-project`。所有命令都需要环境中的 `SUPABASE_ACCESS_TOKEN`。`create` 和 `pooler-url` 还需要 `DB_PASS`。每个子命令都有 `--json` 模式。 |
| `gstack-gbrain-repo-policy` | 每个远程信任三元组。子命令:`get`、`set`、`list`、`normalize` |
| `gstack-gbrain-source-wireup` | 通过 `gbrain sources add` + `git worktree` 将你的 `~/.gstack/` 大脑仓库注册为 gbrain 的联合源,然后运行初始 `gbrain sync`。幂等。替换 v1.12.x 中已失效的 `consumers.json + /ingest-repo` HTTP 连接。标志:`--strict`、`--source-id <id>`、`--no-pull`、`--uninstall`、`--probe`。 |

### gbrain CLI(上游工具)

Gbrain 本身附带了 gstack 包装的这些命令:

| 命令 | 用途 |
|---|---|
| `gbrain init --pglite` | 初始化本地 PGLite 大脑 |
| `gbrain init --non-interactive` | 通过环境初始化(`GBRAIN_DATABASE_URL` 或 `DATABASE_URL`)。永远不要将 URL 作为 argv 传递 — 它会泄漏到 shell 历史记录。 |
| `gbrain doctor --json` | 健康检查。返回 `{status: "ok"|"warnings"|"error", health_score: 0-100, checks: [...]}` |
| `gbrain migrate --to supabase --url ...` | 将 PGLite 大脑移动到 Supabase(无损,将源保留为备份) |
| `gbrain migrate --to pglite` | 反向迁移 |
| `gbrain search "query"` | 搜索大脑 |
| `gbrain put_page --title "..." --tags "a,b" <<<"content"` | 写入页面 |
| `gbrain get_page "<slug>"` | 获取页面 |
| `gbrain serve` | 启动 MCP stdio 服务器(由 `claude mcp add` 使用) |

### 配置文件 + 状态

| 路径 | 存放内容 |
|---|---|
| `~/.gbrain/config.json` | 引擎(pglite/postgres)、数据库 URL 或路径、API 密钥。模式 0600。由 `gbrain init` 写入。 |
| `~/.gstack/gbrain-repo-policy.json` | 每个远程信任三元组。架构 v2。模式 0600。 |
| `~/.gstack/.setup-gbrain.lock.d` | 并发运行锁(原子 mkdir)。在正常退出 + SIGINT 时释放。 |
| `~/.gstack/.brain-queue.jsonl` | gstack 内存同步的待处理同步条目 |
| `~/.gstack/.brain-last-push` | 上次同步推送的时间戳(用于 `/health` 评分) |
| `~/.gstack-brain-remote.txt` | 你的 gstack 内存同步远程仓库的 URL(可以在机器之间安全复制) |
| `~/.gstack/.setup-gbrain-inflight.json` | 为未来的 `--resume-provision` 持久化状态保留 |

### 环境变量

| 变量 | 读取位置 | 作用 |
|---|---|---|
| `SUPABASE_ACCESS_TOKEN` | `gstack-gbrain-supabase-provision` | 管理 API 调用的 PAT。每次设置运行后丢弃。 |
| `DB_PASS` | `gstack-gbrain-supabase-provision`(create、pooler-url) | 生成的数据库密码。永远不在 argv 中。 |
| `GBRAIN_DATABASE_URL` | `gbrain init`、`gbrain doctor` 等 | Postgres 连接字符串(对我们来说是 Supabase pooler URL)。环境优先于 `~/.gbrain/config.json`。 |
| `DATABASE_URL` | `gbrain init`(回退) | 与 `GBRAIN_DATABASE_URL` 相同的语义;第二个检查。 |
| `SUPABASE_API_BASE` | `gstack-gbrain-supabase-provision` | 覆盖管理 API 主机。测试用于指向模拟服务器。 |
| `GBRAIN_INSTALL_DIR` | `gstack-gbrain-install` | 覆盖默认安装路径(`~/gbrain`) |
| `GSTACK_HOME` | 每个 bin 辅助工具 | 覆盖 `~/.gstack` 状态目录。大量测试使用。 |

## 安全模型

此技能触及的每个秘密都有一条规则:**仅环境变量,永远不是 argv,永远不记录,永远不由我们写入磁盘。**唯一的持久存储是 gbrain 自己的 `~/.gbrain/config.json`,模式为 0600,这是 gbrain 的规则,不是我们的。

**在代码中强制执行:**

- `test/skill-validation.test.ts` 中的 CI grep 测试如果 `$SUPABASE_ACCESS_TOKEN` 或 `$GBRAIN_DATABASE_URL` 出现在 argv 位置,则构建失败
- 如果 `--insecure`、`-k` 或 `NODE_TLS_REJECT_UNAUTHORIZED=0` 出现在 `bin/gstack-gbrain-supabase-provision` 中,CI grep 测试失败
- 配置辅助工具顶部的 `set +x` 防止调试跟踪泄漏 PAT
- 遥测负载仅包含枚举的分类值(场景、安装结果、MCP 选择加入、信任层级)— 永远不包含可能包含秘密的自由格式字符串

**通过测试强制执行:**

- `test/secret-sink-harness.test.ts` 使用种子秘密运行每个秘密处理 bin,并断言种子永远不会出现在任何捕获的通道中(stdout、stderr、`$HOME` 下的文件、遥测 JSONL)。每个种子四个匹配规则:精确、URL 解码、前 12 个字符前缀、base64。
- 同一测试文件中的正向控制故意在每个覆盖的通道中泄漏种子,并断言工具捕获每一个。如果没有正向控制,一个静默少报的工具看起来与一个工作的工具完全相同。

**你仍然可以泄漏的内容**(v1 的诚实限制):

- 如果你在 `read -s` 之外的普通聊天消息中粘贴秘密,它会在对话记录和任何主机端日志中
- 泄漏工具不会转储子进程环境 — 一个 `env >> ~/.log` 的 bin 会逃避检测(v1 中没有 bin 这样做;grep 测试阻止它)
- 你的 shell 自己的 `HISTFILE` 行为是你的 shell 的,不是我们的 — 我们永远不会将秘密传递给 argv,所以它们不会通过我们的代码到达那里,但没有什么能阻止你自己将一个粘贴到原始 `curl` 命令中

## 故障排除

### 安装期间"检测到 PATH 遮蔽"

另一个 `gbrain` 二进制文件在 PATH 中比安装程序刚刚链接的那个更早。安装程序的版本检查捕获了它。修复其中之一:

- 如果你不需要另一个,`rm $(which gbrain)`
- 在你的 shell rc 中将 `~/.bun/bin` 前置到 PATH,以便链接的二进制文件获胜
- 将 `GBRAIN_INSTALL_DIR` 设置为遮蔽二进制文件的安装目录并重新运行

然后重新运行 `/setup-gbrain`。

### "拒绝直连 URL"

你粘贴了一个 `db.<ref>.supabase.co:5432` URL。这些是仅 IPv6 的,在大多数环境中会失败。改用 Session Pooler URL:Supabase 仪表板 → Settings → Database → Connection Pooler → **Session** → 复制 URI(端口 6543)。

### 自动配置在 180 秒时超时

Supabase 项目仍在初始化。你的 ref 已在退出消息中打印。等待一分钟,然后:

```bash
/setup-gbrain --resume-provision <ref>
```

技能会重新收集 PAT,跳过项目创建,恢复轮询。

### "另一个 `/setup-gbrain` 实例正在运行"

你有一个陈旧的锁目录。如果你确定没有其他实例实际在运行:

```bash
rm -rf ~/.gstack/.setup-gbrain.lock.d
```

然后重新运行。

### 策略文件上"没有跨模型张力"

你手动编辑了 `~/.gstack/gbrain-repo-policy.json` 并使用了旧的 `allow` 值?没问题。在下次读取时,gstack 会自动将 `allow` 迁移为 `read-write` 并添加 `_schema_version: 2`。stderr 上有一行日志,幂等,确定性。

### `gbrain doctor` 显示"warnings"

`/health` 将其视为黄色,而不是红色。检查 `gbrain doctor --json | jq .checks` 以查看哪些子检查是警告。典型原因:解析器 MECE 重叠(技能名称冲突)或数据库连接尚未配置。

### 切换 PGLite → Supabase 挂起

同级 Conductor 工作区中的另一个 gstack 会话可能通过其序言的 `gstack-brain-sync` 调用持有本地 PGLite 文件的锁。关闭其他工作区,重新运行 `/setup-gbrain --switch`。超时限制为 180 秒,所以你永远不会真正永远等待。

## 为什么这样设计

**为什么是每个远程信任三元组而不是二进制允许/拒绝?**多客户顾问需要搜索而不写回。一个自由开发者上午为客户 A 工作,下午为客户 B 工作,不能让 A 的代码洞察泄漏到客户 B 可以搜索的大脑中。只读可以干净地解决这个问题。

**为什么不将 gbrain 捆绑到 gstack 中?**Gbrain 是一个独立的、积极开发的项目,有自己的发布节奏、架构迁移和 MCP 界面。捆绑意味着 gstack 必须控制 gbrain 更新,这会减慢 gbrain 改进到达用户的速度。分离但集成让每个都可以按自己的节奏发布。

**为什么 `gbrain init --non-interactive` 通过环境变量而不是标志?**连接字符串包含数据库密码。将它们作为 argv 传递会将密码放入 `ps`、shell 历史记录和进程列表中。环境变量传递将秘密仅保留在进程内存中。Gbrain 支持 `GBRAIN_DATABASE_URL` 和 `DATABASE_URL`;我们使用前者以避免与非 gbrain 工具冲突。

**为什么在 PATH 遮蔽时硬失败而不是警告并继续?**遮蔽的 `gbrain` 意味着每个后续命令调用的二进制文件与我们刚刚安装的不同。这是一个静默的版本漂移错误,几周后会显现为神秘的功能差距。设置技能有一项工作 — 设置一个工作环境。拒绝安装到损坏的环境是设置技能正确的行为。

**为什么不自动导入每个仓库?**隐私 + 噪音。一个自动导入序言钩子,摄取你触及的每个仓库,会:(a)在未经同意的情况下将工作代码泄漏到共享大脑中,以及(b)用一次性仓库堵塞搜索。每个远程策略使摄取成为明确的、每个仓库的决策。`/setup-gbrain` 今天不安装任何自动导入钩子 — 但策略存储对以后的钩子是向前兼容的。

## 相关技能 + 后续步骤

- `/health` — 在其 0-10 综合评分中包含 GBrain 维度(doctor 状态、同步队列深度、上次推送时间)。当 gbrain 未安装时,该维度被省略;在非 gbrain 机器上运行 `/health` 不会惩罚该选择。
- `/gstack-upgrade` — 保持 gstack 本身最新。不会独立升级 gbrain。要升级 gbrain,更新 `bin/gstack-gbrain-install` 中的 `PINNED_COMMIT` 并重新运行 `/setup-gbrain`。
- `/retro` — 当内存同步开启时,每周回顾会从你的 gbrain 中提取学习内容和计划,让回顾引用跨机器历史。

运行 `/setup-gbrain` 看看效果如何。