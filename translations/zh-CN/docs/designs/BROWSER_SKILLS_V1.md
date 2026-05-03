# Browser-Skills v1 — 编码重复的浏览器流程

**状态:** 第 1 阶段已在 `garrytan/browserharness` 上发布。第 2-4 阶段如下所述。
**最后更新:** 2026-04-26
**作者:** garrytan (经 /plan-eng-review 和 /codex 外部审查)

## 这是什么

Browser-skills 是按任务划分的目录,将重复的浏览器流程编码为确定性的 Playwright 脚本。每个技能包含:

```
browser-skills/<name>/
├── SKILL.md                        # frontmatter + 文本契约
├── script.ts                       # 确定性逻辑
├── _lib/browse-client.ts           # SDK 的供应商副本
├── fixtures/<host>-<date>.html     # 用于测试的捕获页面
└── script.test.ts                  # 针对 fixture 的解析器测试
```

用户(或在第 2 阶段,刚刚完成流程的代理)创建一次技能。未来的调用运行脚本,在 200ms 内返回 JSON,而不是代理通过 `$B` 原语重新探索所需的 30 秒。

已发布的参考是 `hackernews-frontpage`:抓取 HN 首页,以 JSON 形式返回 30 个故事。尝试 `$B skill list` 和 `$B skill run hackernews-frontpage`。

## 为什么这与 domain-skills (v1.8.0.0) 不同

- **Domain-skills** = "代理记住关于站点的事实。" 按主机名键入的 JSONL 注释,在会话开始时注入提示。状态机处理隔离 → 活动 → 全局提升。
- **Browser-skills** = "代理将过程编码为确定性脚本。" 按任务划分的目录,通过 `$B skill run` 执行,守护进程的作用域令牌用于每次生成的能力隔离。

两者使用相同的心智模型(按主机,三层作用域)。过程层是更大生产力提升所在,因为它将抓取和表单自动化从潜在空间推送到可重现的代码中。

## 为什么这不是现有的 P1 ("自创作 `$B` 命令")

原始 P1 被 Codex 的 T1 异议阻止:代理创作的 TypeScript 无法在守护进程*内部*安全运行(环境全局变量、构造函数小工具、批准和执行之间的顶层 await TOCTOU)。正确的设计是"带能力传递 IPC 的进程外工作器隔离"。这是一个可能永远不会发布的困难项目。

Browser-skills 通过将脚本作为独立的 Bun 进程在守护进程*外部*运行来回避整个问题。守护进程从不导入或评估技能代码。技能通过环回 HTTP 与守护进程通信 — 与任何外部客户端使用的线格式相同。

批准的计划取代了现有的 P1。

---

## 分阶段

| 阶段 | 分支 | 范围 |
|-------|--------|-------|
| **1** | `garrytan/browserharness` | SDK、存储、`$B skill list/run/show/test/rm` 子命令、作用域令牌模型、捆绑的 `hackernews-frontpage` 参考。**已发布 (v1.19.0.0,与第 2a 阶段合并)。** |
| **2a** | `garrytan/browserharness` (继续) | `/scrape <intent>` (只读,带匹配/原型路径的单一入口点) + `/skillify` (将原型编码为永久技能)。添加 `browse/src/browser-skill-write.ts` D3 原子写入助手。**正在发布 v1.19.0.0。** |
| **2b** | 新 (`browser-skills-automate`) | `/automate` 技能模板 (`/scrape` 的变更流程兄弟)。重用 `/skillify` 和 D3 助手。运行非编码时的每个变更步骤确认门。TODOS 中的 P0。 |
| **3** | 新 (`browser-skills-resolver`) | 会话开始时的解析器注入(按主机的 browser-skill 发现)。镜像 domain-skill 注入。`gstack-config browser_skillify_prompts` 开关。 |
| **4** | 新 | 评估测试基础设施 (LLM-judge)、fixture 过时检测、针对实时页面的定期重新验证、用于不受信任生成的 OS 级 FS 沙箱。 |

---

## 第 1 阶段架构

### 锁定的决策 (13)

1. **第 1 阶段 = 完整存储 + SDK + 子命令 + 捆绑参考。** 尚无代理创作。第 2 阶段实现 `/scrape` 和 `/automate`。
2. **第 2 阶段的两个动词:`/scrape` (只读) 和 `/automate` (变更)。** 它们共享 skillify 批准门机制,但作为单独的技能模板存在。
3. **替换 TODOS.md 中现有的自创作-`$B` P1。** 相同的用户可见目标,没有守护进程内隔离问题。
4. **SDK 分发:每个技能内的兄弟文件 (选项 E)。** 规范 SDK 位于 `browse/src/browse-client.ts` (~250 LOC)。每个技能在 `<skill>/_lib/browse-client.ts` 处附带一个副本。第 2 阶段的生成器在每个生成的脚本旁边复制当前 SDK。每个技能完全自包含:将目录复制到任何地方,它都能运行。版本漂移不可能(SDK 冻结在技能创作时的版本)。磁盘成本:每个技能约 3KB。
5. **三层查找:捆绑 → 全局 → 项目。** 捆绑技能随 gstack 安装只读发布 (`<gstack-install>/browser-skills/<name>/`)。全局位于 `~/.gstack/browser-skills/<name>/`。每个项目位于 `<project>/.gstack/browser-skills/<name>/`。查找按优先级顺序遍历层级 项目 → 全局 → 捆绑;第一个命中获胜。**`$B skill list` 在每个技能名称旁边打印解析的层级**,因此"为什么运行那个?"永远不会成为调试谜团。
6. **信任模型:生成时的作用域令牌,而不是 env-scrub-as-sandbox。** 见下面的"信任模型"。(在 Codex 将其标记为安全剧场后,从原始 env-scrub 计划修订。)
7. **单一真实来源:仅 SKILL.md frontmatter。** 没有 `meta.json`。Frontmatter 保存 host、triggers、args、version、source、trusted。SHA256/过时性推迟到第 4 阶段作为单独的 `.checksum` 附属文件(如果最终实现)。
8. **没有 INDEX.json。遍历目录。** `$B skill list` 枚举三个层级并解析每个 SKILL.md frontmatter。50 个技能约 5-10ms。消除整个"索引与磁盘漂移"错误类别。
9. **`$B skill run` 输出协议。** stdout = JSON。stderr = 流式日志。退出 0 / 非零。默认 60s 超时,通过 `--timeout=Ns` 覆盖。最大 stdout 1MB(如果超出则截断 + 非零退出)。匹配 `gh` / `kubectl` / `docker` 约定。
10. **Fixture 重放:两种测试类型的两种模式。** SDK 单元测试建立测试内模拟 HTTP 服务器。端到端技能测试通过脚本的导出解析器函数解析捆绑的 HTML fixtures(不需要守护进程)。第 1 阶段仅 fixture 对 `hackernews-frontpage` 足够;第 2 阶段 `/automate` 将需要更丰富的 fixtures。
11. **参考技能:`hackernews-frontpage`。** 抓取 HN 首页(标题、点数、评论)。无认证、稳定 HTML、理想的 fixture 测试目标。
12. **令牌/端口发现:仅为生成的技能使用作用域令牌 env;用于独立调试运行的状态文件回退。** 通过 `$B skill run` 生成时,SDK 从 env 读取 `GSTACK_PORT` + `GSTACK_SKILL_TOKEN`。对于独立的 `bun run script.ts`,SDK 回退到 `<project>/.gstack/browse.json`(根据 `config.ts:50` 的实际状态文件路径)。
13. **CHANGELOG 诚实。** 第 1 阶段引导:人类可以手写 gstack 运行的确定性浏览器脚本。第 1 阶段明确指出代理创作在下一个版本中实现。没有捏造的性能数字 — 第 1 阶段没有前后对比。

### 信任模型(决策 #6 详细说明)

两个正交轴:

| 轴 | 机制 | 默认 |
|------|-----------|---------|
| **守护进程端能力** | 每次生成的作用域令牌绑定到 `read+write` 范围(17 个命令的浏览器驱动表面,减去 `eval`/`js`/`cookies`/`storage` 等管理命令)。单次使用的 clientId 编码技能名称 + 生成 id。生成退出时撤销。 | 始终作用域(从不是守护进程根令牌)。 |
| **进程端 env 访问** | SKILL.md frontmatter `trusted: true` 传递 `process.env` 减去 `GSTACK_TOKEN`。`trusted: false`(默认)删除除最小允许列表(LANG、LC_ALL、TERM、TZ、锁定的 PATH)之外的所有内容,并明确剥离秘密模式键(TOKEN/KEY/SECRET/PASSWORD、AWS_*、AZURE_*、GCP_*、ANTHROPIC_*、OPENAI_*、GITHUB_* 等)。 | 不受信任(必须选择加入)。 |

`GSTACK_PORT` 和 `GSTACK_SKILL_TOKEN` 始终最后注入,因此父进程无法通过在 env 中设置它们来覆盖它们。

**这做对了什么:** 守护进程端作用域令牌可由守护进程强制执行。尝试调用 `eval`(管理范围)的技能会得到 403,即使 SDK 公开了它。能力边界在正确的位置。

**这没有关闭什么:** Bun 没有内置的 FS 沙箱。不受信任的技能仍然可以 `import 'fs'` 并读取 OS 用户可以读取的任何内容(例如 `~/.ssh/id_rsa`)。env scrub 是卫生措施,而不是沙箱。OS 级隔离(`sandbox-exec`、命名空间)是第 4 阶段的工作,并在现有的受信任/不受信任契约后面干净地插入。

原始计划将 env-scrub 称为沙箱。Codex 正确地将其标记为剧场。修订后的计划称其为实际情况:尽力而为的卫生措施加上纵深防御,真正的边界在守护进程端作用域令牌。

### 文件布局

```
browse/src/
├── browse-client.ts                # 规范 SDK (~250 LOC)
├── browser-skills.ts               # 3 层遍历 + frontmatter 解析器 + 墓碑
├── browser-skill-commands.ts       # $B skill list/show/run/test/rm + spawnSkill
└── skill-token.ts                  # mintSkillToken / revokeSkillToken 包装器

browser-skills/
└── hackernews-frontpage/           # 捆绑参考技能
    ├── SKILL.md
    ├── script.ts
    ├── _lib/browse-client.ts        # 规范的字节相同副本
    ├── fixtures/hn-2026-04-26.html
    └── script.test.ts

browse/test/
├── skill-token.test.ts              # mint/revoke 生命周期、范围断言
├── browse-client.test.ts            # 模拟 HTTP 服务器、线格式、认证
├── browser-skills-storage.test.ts   # 3 层遍历、frontmatter、墓碑
└── browser-skill-commands.test.ts   # parseRunArgs、调度、env scrub、生成

test/skill-validation.test.ts       # 扩展:捆绑技能契约检查
```

### 什么不改变

- Domain-skills 存储、状态机或注入。未触及。
- 隧道表面允许列表 (`server.ts:118-123`)。相同的 17 个命令。
- L1-L6 安全堆栈。Browser-skills 在第 1 阶段不向提示注入文本;第 3 阶段的解析器注入将使用现有的 UNTRUSTED 信封。
- `cli.ts` 中 `sendCommand()` 的 HTTP 客户端。SDK 是一个具有不同关注点的单独模块(库 vs CLI 进程)。

---

## Codex 外部审查发现(审查后响应)

/codex 审查标记了 8 个发现。计划如下解决它们:

| # | 发现 | 第 1 阶段响应 |
|---|---------|------------------|
| 1 | 没有 FS 沙箱的信任模型是假的 | **已关闭**,通过上面的决策 #6(作用域令牌)。 |
| 2 | 第 1 阶段对于一个捆绑技能来说过度构建(查找层级、墓碑等) | **已确认但保留。** 用户选择完整的第 1 阶段以在第 2 阶段实现代理创作之前锁定架构。每个子系统都足够小,如果数据后来显示未使用,可以干净地删除。 |
| 3 | `cli.ts:398` 中的现有客户端模式可能使兄弟 SDK 冗余 | **已验证为假。** 第 398 行是 `extractTabId()`(标志解析器)的结尾。实际的 HTTP 客户端是 cli.ts:401-467 的 `sendCommand()`,但它与 CLI 耦合(`process.stdout.write`、`process.exit`、服务器重启恢复)。不可作为库重用。新的 `browse-client.ts` 镜像其线格式但是库形状的。 |
| 4 | "第一个命中获胜"查找不透明 | **已缓解**,通过在 `$B skill list` 和 `$B skill show` 中内联列出解析的层级。未来:如果层级覆盖证明令人困惑,可选的 `--source bundled\|global\|project` 标志。 |
| 5 | 原子技能打包比索引问题更重要;符号链接防御 | **第 1 阶段已关闭**:捆绑技能作为 gstack 安装的一部分发布(无实时写入;由于是安装目录中的只读文件而原子)。第 2 阶段的 `writeBrowserSkill` 将写入临时目录然后重命名,并使用 `realpath`/`lstat` 规则(现有的 `browse/src/path-security.ts`)。 |
| 6 | 从活动源合成的第 2 阶段很弱(有损环形缓冲区) | **第 2 阶段设计的未决问题。** 活动源是遥测,而不是重放 IR。第 2 阶段将需要结构化记录器或重新提示代理使用其自己的上下文从头编写脚本。在第 2 阶段的设计过程中决定。 |
| 7 | Bun 运行时回归:作为独立 Bun 的技能脚本重新引入 Bun 运行时要求 | **第 2 阶段分发的未决问题。** 第 1 阶段回避了这一点,因为捆绑的参考技能在 gstack 安装内发布(已经使用 Bun 构建)。第 2 阶段需要在 (a) 为每个生成的技能附带 Bun 二进制文件,(b) 将技能编译为自包含可执行文件,或 (c) 使用 Node.js 与 `cli.ts` 的 HTTP 模式之间做出决定。 |
| 8 | `file://` fixtures 不能证明时序/认证/导航/延迟水合 | **已记录限制。** 对 `hackernews-frontpage` 足够。第 2 阶段 `/automate` 将需要更丰富的 fixtures(带时序的模拟守护进程、记录的 HAR 重放等)。 |

---

## 第 2a 阶段 — `/scrape` + `/skillify` (正在发布 v1.19.0.0)

两个技能模板加一个助手模块。`/scrape <intent>` 是提取页面数据的单一入口点;对新意图的首次调用通过 `$B` 原语进行原型设计并返回 JSON,对匹配意图的后续调用在约 200ms 内路由到编码的 browser-skill。`/skillify` 将最近成功的原型编码为磁盘上的永久 browser-skill。变更流程兄弟 `/automate` 推迟到第 2b 阶段(TODOS 中的 P0)。

### 在 v1.19.0.0 计划审查期间锁定的决策 (`/plan-eng-review`)

| ID | 决策 | 锁定行为 |
|----|----------|-----------------|
| **D1** | `/skillify` 来源保护 | 回溯 ≤10 个代理回合,寻找明确界定的 `/scrape` 调用(原型的意图行 + 其尾随 JSON 输出)。如果未找到,拒绝并显示:*"在此对话中未找到最近的 /scrape 结果。首先运行 /scrape <intent>,然后说 /skillify。"* 没有静默回退。 |
| **D2** | 合成输入切片 | 模板指示代理仅提取产生用户接受的 JSON 的最终尝试 `$B` 调用,加上用户声明的意图字符串。删除失败的选择器尝试,删除不相关的聊天,删除早期会话内容。通过选择选项 (b)(从代理自己的上下文重新提示,而不是结构化记录器)关闭 Codex 发现 #6。 |
| **D3** | 原子写入规则 | `/skillify` 写入 `~/.gstack/.tmp/skillify-<spawnId>/`,针对临时目录运行 `$B skill test`,并且仅在成功 + 用户批准时重命名到最终层级路径。测试失败或批准拒绝时:`rm -rf` 完全删除临时目录(对于从未批准的技能没有墓碑)。新模块 `browse/src/browser-skill-write.ts`(`stageSkill` / `commitSkill` / `discardStaged`)具有根据 Codex 发现 #5 的 `realpath`/`lstat` 规则。 |
| **D4** | 测试范围 | 5 个门层 E2E(scrape 匹配、scrape 原型、skillify 成功、skillify 来源拒绝、批准门拒绝)+ 1 个单元测试(原子写入助手失败清理)+ 1 个手动验证的冒烟测试(变更意图拒绝)。在 `test/helpers/touchfiles.ts` 中注册。 |

### 延续

- **默认层级:全局。** 对过程倾向全局,在 `/skillify` 时使用每个项目覆盖(镜像 domain-skill 范围)。第 1 阶段存储助手支持两个查找路径。
- **Bun 运行时分发。** Codex 发现 #7 保持开放。第 2a 阶段假设 Bun 在 PATH 上(gstack 已经通过 `setup:6-15` 要求它)。在 `/skillify` SKILL.md "限制"中记录。真正的修复在第 4 阶段实现。

## 第 2b 阶段 — `/automate` 草图

`/scrape` 的变更流程兄弟。相同的 skillify 模式(按原样重用 `/skillify` 和 D3 助手)。区别:运行非编码时,每个变更步骤的 UNTRUSTED 包装摘要 + `AskUserQuestion` 确认门。编码后,技能无人值守运行(编码脚本准确枚举运行哪些 `$B click`/`fill`/`type` 调用)。参见 `TODOS.md` 中的 P0 条目。

## 第 3 阶段草图

会话开始时的解析器注入。镜像 `server.ts:722-743` 的 domain-skill 注入:

```ts
const browserSkillsBlock = await renderBrowserSkillsForHost(hostname, projectSlug);
if (browserSkillsBlock) {
  systemPrompt += `\n\n${browserSkillsBlock}`;
}
```

`renderBrowserSkillsForHost()` 读取 3 个层级,过滤到 `host` 字段匹配的技能,并发出列出它们的 UNTRUSTED 包装块。

`gstack-config browser_skillify_prompts`(默认关闭):打开时,当活动源显示单个主机上的 ≥N 个命令且该主机+意图尚不存在技能时,`/qa`、`/design-review` 等中的任务结束提示触发。

## 第 4 阶段草图

- LLM-judge 评估("代理是否寻求技能而不是重新探索?")。
- Fixture 过时检测 — 将捆绑的 fixture 与实时页面进行比较。
- 用于不受信任生成的 OS 级 FS 沙箱(macOS 上的 `sandbox-exec`,Linux 上的命名空间 / seccomp)。
- `$B skill upgrade <name>` — 当规范 SDK 更改时重新生成兄弟 SDK 副本。

---

## 验证(第 1 阶段)

`bun test` 通过新的测试文件:
- `browse/test/skill-token.test.ts` — 15 个断言
- `browse/test/browse-client.test.ts` — 26 个断言
- `browse/test/browser-skills-storage.test.ts` — 31 个断言
- `browse/test/browser-skill-commands.test.ts` — 29 个断言
- `browser-skills/hackernews-frontpage/script.test.ts` — 13 个断言
- `test/skill-validation.test.ts` — 7 个新的捆绑技能断言

守护进程运行时的端到端:

```bash
$B skill list                            # 显示 hackernews-frontpage (捆绑)
$B skill show hackernews-frontpage       # 打印 SKILL.md
$B skill run hackernews-frontpage        # 返回 30 个故事的 JSON
$B skill test hackernews-frontpage       # 运行 script.test.ts
```