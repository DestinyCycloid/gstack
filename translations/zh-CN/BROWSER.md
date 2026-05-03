# Browser — 完整参考

gstack 的浏览器功能集于一份文档。无头 Chromium 守护进程、约 70+ 条命令、基于 ref 的元素选择、可编码的浏览器技能、带 Chrome 侧边栏的真实浏览器模式、侧边栏内的 Claude PTY、ngrok 配对代理流程,以及分层的提示注入防御 — 全部通过编译的 CLI 实现,向 stdout 输出纯文本。每次调用约 100-200ms。零上下文 token 开销。

如果你在最近一两个版本中使用过 gstack,生产力循环是新的亮点:`/scrape <intent>` 驱动页面一次,`/skillify` 将流程编码为确定性的 Playwright 脚本,下次对相同意图执行 `/scrape` 时在约 200ms 内运行,而不是约 30 秒的代理重新探索。

---

## 快速开始

```bash
# 一次性:构建二进制文件 (browse/dist/browse, ~58MB)
bun install && bun run build

# 设置一次 $B 然后忘记它
B=./browse/dist/browse           # 或 ~/.claude/skills/gstack/browse/dist/browse

# 驱动页面
$B goto https://news.ycombinator.com
$B snapshot -i                   # 你稍后可以点击/填充/检查的 @e refs
$B click @e30                    # 点击快照中的 ref 30
$B text                          # 获取干净的页面文本
$B screenshot /tmp/hn.png

# 编码重复流程
/scrape latest hacker news stories
/skillify                        # 写入 ~/.gstack/browser-skills/hn-front/...
/scrape hacker news front page   # 第二次调用:通过编码技能 200ms

# 实时观看 Claude 工作
$B connect                       # 有头 Chromium + 侧边栏扩展
```

---

## 目录

1. [它是什么](#what-it-is)
2. [生产力循环 — `/scrape` + `/skillify`](#the-productivity-loop)
3. [架构](#architecture)
4. [命令参考](#command-reference)
5. [快照系统 + 基于 ref 的选择](#snapshot-system)
6. [浏览器技能运行时](#browser-skills-runtime)
7. [域技能(每个站点的代理笔记)](#domain-skills)
8. [真实浏览器模式(`$B connect`)](#real-browser-mode)
9. [侧边栏 + 侧边栏代理](#side-panel--sidebar-agent)
10. [配对代理 — 通过 ngrok 隧道的远程代理](#pair-agent)
11. [认证 + tokens](#authentication)
12. [提示注入安全栈(L1–L6)](#security-stack)
13. [截图、PDF、视觉检查](#screenshots-pdfs-visual)
14. [本地 HTML — `goto file://` vs `load-html`](#local-html)
15. [批处理端点](#batch-endpoint)
16. [控制台、网络、对话框捕获](#capture)
17. [JS 执行 — `js` + `eval`](#js-execution)
18. [标签页、框架、状态、监视、收件箱](#tabs-frames-state)
19. [CDP 逃生舱 + CSS 检查器](#cdp)
20. [性能 + 规模](#performance)
21. [多工作区隔离](#multi-workspace)
22. [环境变量](#environment-variables)
23. [源码映射](#source-map)
24. [开发 + 测试](#development)
25. [交叉引用](#cross-references)
26. [致谢](#acknowledgments)

---

## 它是什么

一个编译的 CLI 二进制文件,通过 HTTP 与持久的本地 Chromium 守护进程通信。CLI 是一个瘦客户端 — 它读取状态文件、发送命令、将响应打印到 stdout。守护进程通过 [Playwright](https://playwright.dev/) 完成实际工作。

早期作为 Chrome MCP 服务器的所有功能现在都通过纯 stdout 实现。没有 JSON-schema 框架、没有协议协商、没有持久 WebSocket — Claude 的 Bash 工具已经存在,所以我们使用它。

三种递进模式:

- **无头**(默认)。守护进程运行 Chromium,没有可见窗口。最快、最便宜,`/qa`、`/design-review`、`/benchmark` 等技能默认使用。
- **通过 `$B connect` 有头**。相同的守护进程,但 Chromium 可见(重新品牌为"GStack Browser"),自动加载侧边栏扩展。你可以实时观看每个命令的执行。
- **通过隧道的配对代理**。守护进程绑定第二个监听器,ngrok 转发。远程代理(Codex、OpenClaw、Hermes,任何可以使用 HTTP 的)通过 26 条命令白名单和作用域单次使用 token 驱动你的本地浏览器。

---

## 生产力循环

v1.19.0.0 的发布亮点。两个 gstack 技能包装浏览器技能运行时,因此第二次要求 Claude 抓取页面时,它在约 200ms 内运行。

### `/scrape <intent>`

拉取页面数据的单一入口点。底层三条路径:

1. **匹配路径(约 200ms)** — 代理运行 `$B skill list`,根据每个技能的 `triggers:` 数组 + `description` + `host` 语义匹配意图,如果存在可信匹配则运行 `$B skill run <name>`。
2. **原型路径(约 30s)** — 没有匹配,代理使用 `$B goto`、`$B text`、`$B html`、`$B links` 等驱动页面,返回 JSON,并附加一行"说 `/skillify`"建议。
3. **变更意图拒绝** — 像 *submit*、*click*、*fill* 这样的动词路由到 `/automate`(阶段 2b,`TODOS.md` 中的 P0)。`/scrape` 按约定是只读的。

### `/skillify`

将最近成功的 `/scrape` 原型编码为磁盘上的永久浏览器技能。十一个步骤,三个锁定约定:

- **D1 — 来源保护。** 回溯 ≤10 个代理回合以获得明确界定的 `/scrape` 结果。如果冷启动则拒绝并显示一条特定消息。不从聊天片段静默合成。
- **D2 — 合成输入切片。** 仅提取产生用户接受的 JSON 的最终尝试 `$B` 调用,加上用户的意图字符串。丢弃失败的选择器、丢弃聊天、丢弃早期会话内容。
- **D3 — 原子写入。** 将所有内容暂存到 `~/.gstack/.tmp/skillify-<spawnId>/`,对临时目录运行 `$B skill test`,仅在测试通过 + 用户批准时重命名到最终层路径。测试失败或拒绝:完全 `rm -rf` 临时目录。永远不会出现半写入的技能在 `$B skill list` 中。

变更流程的兄弟 `/automate` 在 `TODOS.md` 中作为 P0 拆分,并在下一个分支上发布 — 相同的 skillify 机制,运行非编码时的每个变更步骤确认门。

完整设计 + 决策轨迹见 [`docs/designs/BROWSER_SKILLS_V1.md`](docs/designs/BROWSER_SKILLS_V1.md)。

---

## 架构

```
┌─────────────────────────────────────────────────────────────────┐
│  Claude Code                                                    │
│                                                                 │
│  $B goto https://staging.myapp.com                              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐    HTTP POST     ┌──────────────┐                 │
│  │ browse   │ ──────────────── │ Bun HTTP     │                 │
│  │ CLI      │  127.0.0.1:rand  │ daemon       │                 │
│  │          │  Bearer token    │              │                 │
│  │ compiled │ ◄──────────────  │  Playwright  │──── Chromium    │
│  │ binary   │  plain text      │  API calls   │    (headless    │
│  └──────────┘                  └──────────────┘     or headed)  │
│   ~1ms startup                  persistent daemon               │
│                                 auto-starts on first call       │
│                                 auto-stops after 30 min idle    │
└─────────────────────────────────────────────────────────────────┘
```

### 守护进程生命周期

1. **首次调用。** CLI 检查 `<project>/.gstack/browse.json` 以查找运行中的服务器。未找到 — 它在后台生成 `bun run browse/src/server.ts`。守护进程通过 Playwright 启动无头 Chromium,选择随机端口(10000–60000),生成 bearer token,写入状态文件(chmod 600),开始接受请求。约 3 秒。
2. **后续调用。** CLI 读取状态文件,使用 bearer token 发送 HTTP POST,打印响应。约 100-200ms 往返。
3. **空闲关闭。** 30 分钟没有命令后,守护进程关闭并清理状态文件。下次调用重新启动它。
4. **崩溃恢复。** 如果 Chromium 崩溃,守护进程立即退出 — 不自我修复,不隐藏失败。CLI 在下次调用时检测到死守护进程并启动新的。

### 多工作区隔离

每个项目根目录(通过 `git rev-parse --show-toplevel` 检测)获得自己的守护进程、端口、状态文件、cookies 和日志。没有跨工作区冲突。状态在 `<project>/.gstack/browse.json`。

| 工作区 | 状态文件 | 端口 |
|-----------|-----------|------|
| `/code/project-a` | `/code/project-a/.gstack/browse.json` | 随机(10000–60000) |
| `/code/project-b` | `/code/project-b/.gstack/browse.json` | 随机(10000–60000) |

---

## 命令参考

约 70 条命令,涵盖读取、写入和元操作。选择器接受 CSS、来自 `snapshot` 的 `@e` refs 或来自 `snapshot -C` 的 `@c` refs。完整表格:

### 读取

| 命令 | 描述 |
|---------|-------------|
| `text [sel]` | 干净的页面文本(或限定到选择器) |
| `html [sel]` | innerHTML,如果没有选择器则为完整页面 HTML |
| `links` | 所有链接为 `text → href` |
| `forms` | 表单字段为 JSON |
| `accessibility` | 完整 ARIA 树 |
| `media [--images\|--videos\|--audio] [sel]` | 带 URL、尺寸、类型的媒体元素 |
| `data [--jsonld\|--og\|--meta\|--twitter]` | 结构化数据:JSON-LD、OG、Twitter Cards、meta 标签 |

### 检查

| 命令 | 描述 |
|---------|-------------|
| `js <expr>` | 在页面上下文中运行内联 JavaScript 表达式,作为字符串返回 |
| `eval <file>` | 从文件运行 JS(路径在 /tmp 或 cwd 下;与 `js` 相同的沙箱) |
| `css <sel> <prop>` | 计算的 CSS 值 |
| `attrs <sel\|@ref>` | 元素属性为 JSON |
| `is <prop> <sel\|@ref>` | 状态检查:visible、hidden、enabled、disabled、checked、editable、focused |
| `console [--clear\|--errors]` | 捕获的控制台消息 |
| `network [--clear]` | 捕获的网络请求 |
| `dialog [--clear]` | 捕获的对话框消息 |
| `cookies` | 所有 cookies 为 JSON |
| `storage` / `storage set <key> <val>` | 读取 localStorage + sessionStorage;设置 localStorage |
| `perf` | 页面加载时间 |
| `inspect [sel] [--all] [--history]` | 通过 CDP 深度 CSS — 完整规则级联、盒模型、计算样式 |
| `ux-audit` | 用于行为分析的页面结构:站点 ID、导航、标题、文本块、交互元素 |
| `cdp <Domain.method> [json-params]` | 原始 CDP 方法调度(默认拒绝;`cdp-allowlist.ts` 中的白名单) |

### 导航

| 命令 | 描述 |
|---------|-------------|
| `goto <url>` | 导航到 URL(`http://`、`https://`、`file://`) |
| `load-html <file>` | 在内存中加载本地 HTML(没有 `file://` URL;在视口缩放变化后存活) |
| `back`、`forward`、`reload` | 标准导航 |
| `url` | 当前页面 URL |
| `wait <sel\|--networkidle\|--load>` | 等待元素、网络空闲或页面加载(15s 超时) |

### 交互

| 命令 | 描述 |
|---------|-------------|
| `click <sel\|@ref>` | 点击元素 |
| `fill <sel> <val>` | 填充输入 |
| `select <sel> <val>` | 选择下拉选项(值、标签或可见文本) |
| `hover <sel>` | 悬停元素 |
| `type <text>` | 输入到聚焦元素 |
| `press <key>` | Playwright 键盘键(区分大小写:Enter、Tab、ArrowUp、Shift+Enter、Control+A、...) |
| `scroll [sel\|@ref]` | 将元素滚动到视图中,如果没有选择器则跳到页面底部 |
| `viewport [<WxH>] [--scale <n>]` | 设置视口大小 + 可选 `deviceScaleFactor` 1-3(retina 截图) |
| `upload <sel> <file> [...]` | 上传文件 |
| `dialog-accept [text]` | 自动接受下一个 alert/confirm/prompt;文本用于 prompts |
| `dialog-dismiss` | 自动关闭下一个对话框 |

### 样式 + 清理

| 命令 | 描述 |
|---------|-------------|
| `style <sel> <prop> <val>` | 修改 CSS 属性(支持撤销) |
| `style --undo [N]` | 撤销最后 N 次样式更改 |
| `cleanup [--ads\|--cookies\|--sticky\|--social\|--all]` | 删除页面杂乱内容 |
| `prettyscreenshot [--scroll-to <sel\|text>] [--cleanup] [--hide <sel>...] [path]` | 带可选清理、滚动、隐藏的干净截图 |

### 视觉

| 命令 | 描述 |
|---------|-------------|
| `screenshot [--selector <css>] [--viewport] [--clip x,y,w,h] [--base64] [sel\|@ref] [path]` | 五种模式:完整页面、视口、元素裁剪、区域剪辑、base64 |
| `pdf [path] [--format letter\|a4\|legal] [...]` | 带完整布局的 PDF:格式、宽度/高度、边距、页眉/页脚模板、页码、--tagged 用于可访问性、--toc 等待 Paged.js |
| `responsive [prefix]` | 三张截图:移动(375x812)、平板(768x1024)、桌面(1280x720) |
| `diff <url1> <url2>` | 两个 URL 之间的文本差异 |

### Cookies + headers

| 命令 | 描述 |
|---------|-------------|
| `cookie <name>=<value>` | 在当前页面域上设置 cookie |
| `cookie-import <json>` | 从 JSON 文件导入 cookies |
| `cookie-import-browser [browser] [--domain d]` | 从已安装的 Chromium 浏览器导入(交互式选择器,或 `--domain` 直接导入) |
| `header <name>:<value>` | 设置自定义请求头(敏感值自动编辑) |
| `useragent <string>` | 设置用户代理(触发上下文重建,使 refs 无效) |

### 标签页 + 框架

| 命令 | 描述 |
|---------|-------------|
| `tabs` | 列出打开的标签页 |
| `tab <id>` | 切换到标签页 |
| `newtab [url] [--json]` | 打开新标签页;`--json` 返回 `{tabId, url}` 用于编程使用 |
| `closetab [id]` | 关闭标签页 |
| `tab-each <command> [args...]` | 在每个打开的标签页上扇出命令;返回 JSON |
| `frame <sel\|@ref\|--name n\|--url pattern\|main>` | 切换到 iframe 上下文(或返回 main);清除 refs |

### 提取

| 命令 | 描述 |
|---------|-------------|
| `download <url\|@ref> [path] [--base64]` | 使用浏览器 cookies 下载 URL 或媒体元素 |
| `scrape <images\|videos\|media> [--selector] [--dir] [--limit]` | 从页面批量下载所有媒体;写入 `manifest.json` |
| `archive [path]` | 通过 CDP 将完整页面保存为 MHTML |

### 快照

| 命令 | 描述 |
|---------|-------------|
| `snapshot [-i] [-c] [-d N] [-s sel] [-D] [-a] [-o path] [-C]` | 带 `@e` refs 的可访问性树;`-i` 仅交互式,`-c` 紧凑,`-d N` 深度,`-s` 范围,`-D` 与之前的差异,`-a` 带注释的截图,`-C` 光标交互式 `@c` refs |

### 服务器生命周期

| 命令 | 描述 |
|---------|-------------|
| `status` | 守护进程健康 + 模式(headless / headed / cdp) |
| `stop` | 关闭守护进程 |
| `restart` | 重启守护进程 |
| `connect` | 启动带侧边栏扩展的有头 GStack Browser |
| `disconnect` | 关闭有头 Chrome,返回无头 |
| `focus [@ref]` | 将有头 Chrome 带到前台(macOS);`@ref` 也滚动到视图中 |
| `state save\|load <name>` | 保存或加载浏览器状态(cookies + URLs) |

### 移交

| 命令 | 描述 |
|---------|-------------|
| `handoff [reason]` | 在当前页面打开可见 Chrome 供用户接管(CAPTCHA、MFA、复杂认证) |
| `resume` | 用户接管后重新快照,将控制权返回给 AI |

### 元 + 链

| 命令 | 描述 |
|---------|-------------|
| `chain`(通过 stdin 的 JSON) | 运行命令序列。将 `[["cmd","arg1",...],...]` 管道到 `$B chain`。在第一个错误时停止。 |
| `inbox [--clear]` | 列出来自侧边栏侦察收件箱的消息 |
| `watch [stop]` | 被动观察 — 用户浏览时定期快照;`stop` 返回摘要 |

### 浏览器技能运行时

| 命令 | 描述 |
|---------|-------------|
| `skill list` | 列出所有浏览器技能及解析的层(project > global > bundled) |
| `skill show <name>` | 打印 SKILL.md |
| `skill run <name> [--arg k=v...] [--timeout=Ns]` | 使用每次生成的作用域 token 生成技能脚本 |
| `skill test <name>` | 对捆绑的 fixtures 运行技能的 `script.test.ts` |
| `skill rm <name> [--global]` | 墓碑化用户层技能 |

### 域技能

| 命令 | 描述 |
|---------|-------------|
| `domain-skill save\|list\|show\|edit\|promote-to-global\|rollback\|rm <host?>` | 每个站点的代理笔记(主机从活动标签页派生)。生命周期:隔离 → 活动(在 N=3 次成功使用后没有分类器标记) → 全局(显式提升) |

别名:`setcontent`、`set-content`、`setContent` → `load-html`(在范围检查之前规范化,因此读取范围的 token 不能使用别名运行写入命令)。

---

## 快照系统

浏览器的关键创新是基于 Playwright 的可访问性树 API 构建的**基于 ref 的元素选择**。没有 DOM 变更。没有注入脚本。只有 Playwright 的原生 AX API。

### `@ref` 如何工作

1. `page.locator(scope).ariaSnapshot()` 返回类似 YAML 的可访问性树。
2. 快照解析器为每个元素分配 refs(`@e1`、`@e2`、...)。
3. 对于每个 ref,它构建一个 Playwright `Locator`(使用 `getByRole` + nth-child)。
4. ref→Locator 映射存储在 `BrowserManager` 上。
5. 稍后的命令如 `click @e3` 查找 Locator 并调用 `locator.click()`。

### Ref 过期检测

SPA 可以在没有导航的情况下改变 DOM(React router、标签页切换、模态框)。发生这种情况时,从之前的 `snapshot` 收集的 refs 可能指向不再存在的元素。`resolveRef()` 在使用任何 ref 之前运行异步 `count()` 检查 — 如果元素计数为 0,它立即抛出一条消息,告诉代理重新运行 `snapshot`。快速失败(约 5ms),而不是等待 Playwright 的 30 秒操作超时。

### 扩展快照功能

- **`--diff`(`-D`)。** 将每个快照存储为基线。在下一次 `-D` 调用时,返回显示更改内容的统一差异。使用它来验证操作(点击、填充等)是否真正起作用。
- **`--annotate`(`-a`)。** 在每个 ref 的边界框处注入临时覆盖 div,拍摄带有可见 ref 标签的截图,然后删除覆盖。使用 `-o <path>` 控制输出。
- **`--cursor-interactive`(`-C`)。** 使用 `page.evaluate` 扫描非 ARIA 交互元素(带 `cursor:pointer`、`onclick`、`tabindex>=0` 的 div)。使用确定性 `nth-child` CSS 选择器分配 `@c1`、`@c2`... refs。这些是 ARIA 树遗漏但用户仍然可以点击的元素。

---

## 浏览器技能运行时

将重复的浏览器流程编码为确定性 Playwright 脚本的每任务目录。复合层。

### 浏览器技能的解剖

```
browser-skills/<name>/
├── SKILL.md                        # frontmatter + 散文约定
├── script.ts                       # 通过 browse-client 的确定性 Playwright 逻辑
├── _lib/browse-client.ts           # SDK 的供应副本(约 3KB,与规范字节相同)
├── fixtures/<host>-<date>.html     # 用于 fixture-replay 测试的捕获页面
└── script.test.ts                  # 针对 fixture 的解析器测试(不需要守护进程)
```

捆绑的参考是 `browser-skills/hackernews-frontpage/`:抓取 HN 首页,返回 30 个故事为 JSON。试试:

```bash
$B skill list                            # 显示 hackernews-frontpage(bundled)
$B skill show hackernews-frontpage
$B skill run hackernews-frontpage        # 约 200ms 内 30 个故事的 JSON
$B skill test hackernews-frontpage       # 对 fixture 运行 script.test.ts
```

### 三层存储

`$B skill list` 按优先级顺序遍历所有三层;第一个命中获胜。解析的层打印在每个技能名称旁边:

| 层 | 路径 | 何时 |
|------|------|------|
| **Project** | `<project>/.gstack/browser-skills/<name>/` | 项目特定技能(已提交或 gitignored) |
| **Global** | `~/.gstack/browser-skills/<name>/` | 每个用户的技能,所有项目 |
| **Bundled** | `<gstack-install>/browser-skills/<name>/` | 随 gstack 一起发布,只读 |

### 信任模型

两个正交轴 — 守护进程端能力和进程端环境 — 独立配置。

| 轴 | 机制 | 默认 |
|------|-----------|---------|
| **守护进程端能力** | 每次生成的作用域 token 绑定到读+写范围(浏览器驱动命令减去管理员:`eval`、`js`、`cookies`、`storage`)。单次使用 clientId 编码技能名称 + 生成 id。生成退出时撤销。 | 始终作用域 — 永远不是守护进程根 token |
| **进程端环境** | `trusted: true` frontmatter 传递 `process.env` 减去 `GSTACK_TOKEN`。`trusted: false`(默认)丢弃除最小白名单(LANG、LC_ALL、TERM、TZ)之外的所有内容,并模式剥离秘密(TOKEN/KEY/SECRET/PASSWORD、AWS_*、ANTHROPIC_*、OPENAI_*、GITHUB_*等)。 | 不受信任(必须选择加入) |

`GSTACK_PORT` 和 `GSTACK_SKILL_TOKEN` 最后注入,因此父进程无法覆盖它们。

### 输出协议

stdout = JSON。stderr = 流式日志。退出 0 / 非零。默认 60s 超时,通过 `--timeout=Ns` 覆盖。最大 stdout 1MB(如果超过则截断 + 非零退出)。匹配 `gh` / `kubectl` / `docker` 约定。

### SDK 分发如何工作

每个技能在 `_lib/browse-client.ts` 处附带自己的 `browse-client.ts` 副本,与规范 `browse/src/browse-