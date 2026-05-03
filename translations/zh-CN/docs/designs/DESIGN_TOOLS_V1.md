# 设计:gstack 视觉设计生成(`design` 二进制文件)

由 /office-hours 生成于 2026-03-26
分支:garrytan/agent-design-tools
仓库:gstack
状态:草稿
模式:内部创业

## 背景

gstack 的设计技能(/office-hours、/design-consultation、/plan-design-review、/design-review)都生成**文本描述**的设计——包含十六进制颜色代码的 DESIGN.md 文件、在文档中用像素规格描述的计划文档、ASCII 艺术线框图。创建者是一位在 OmniGraffle 中手工设计了 HelloSign 的设计师,觉得这很尴尬。

价值单元是错误的。用户不需要更丰富的设计语言——他们需要一个可执行的视觉产物,将对话从"你喜欢这个规格吗?"转变为"这是那个屏幕吗?"

## 问题陈述

设计技能用文本描述设计而不是展示它。Argus UX 改造计划就是例子:487 行详细的情感弧线规格、排版选择、动画时序——零视觉产物。一个"设计"的 AI 编码代理应该产生你可以看到并本能反应的东西。

## 需求证据

创建者/主要用户觉得当前输出令人尴尬。每次设计技能会话都以散文结束,而本应是模型。GPT Image API 现在生成像素完美的 UI 模型,具有准确的文本渲染——证明纯文本输出合理性的能力差距不再存在。

## 最窄楔子

一个编译的 TypeScript 二进制文件(`design/dist/design`),封装 OpenAI Images/Responses API,可通过 `$D` 从技能模板调用(镜像现有的 `$B` browse 二进制模式)。优先集成顺序:/office-hours → /plan-design-review → /design-consultation → /design-review。

## 已同意的前提

1. GPT Image API(通过 OpenAI Responses API)是正确的引擎。Google Stitch SDK 是备份。
2. **视觉模型是设计技能的默认开启**,有简单的跳过路径——不是选择加入。(根据 Codex 挑战修订。)
3. 集成是共享实用程序(不是每个技能重新实现)——任何技能都可以调用的 `design` 二进制文件。
4. 优先级:/office-hours 优先,然后是 /plan-design-review、/design-consultation、/design-review。

## 跨模型视角(Codex)

Codex 独立验证了核心论点:"失败不是 markdown 内的输出质量;而是当前的价值单元是错误的。"主要贡献:
- 挑战前提 #2(选择加入 → 默认开启)——已接受
- 提出基于视觉的质量门:使用 GPT-4o 视觉验证生成的模型是否有不可读文本、缺失部分、布局损坏,自动重试一次
- 范围界定 48 小时原型:共享 `visual_mockup.ts` 实用程序,仅 /office-hours + /plan-design-review,主模型 + 2 个变体

## 推荐方法:`design` 二进制文件(方法 B)

### 架构

**共享 browse 二进制文件的编译和分发模式**(bun build --compile、setup 脚本、技能模板中的 $VARIABLE 解析),但架构上更简单——没有持久守护进程服务器、没有 Chromium、没有健康检查、没有令牌认证。design 二进制文件是一个无状态 CLI,进行 OpenAI API 调用并将 PNG 写入磁盘。会话状态(用于多轮迭代)是一个 JSON 文件。

**新依赖:** `openai` npm 包(添加到 `devDependencies`,不是运行时依赖)。Design 二进制文件与 browse 分开编译,这样 openai 不会使 browse 二进制文件膨胀。

```
design/
├── src/
│   ├── cli.ts            # 入口点,命令分发
│   ├── commands.ts        # 命令注册表(文档 + 验证的真实来源)
│   ├── generate.ts        # 从结构化简报生成模型
│   ├── iterate.ts         # 对现有模型进行多轮迭代
│   ├── variants.ts        # 从简报生成 N 个设计变体
│   ├── check.ts           # 基于视觉的质量门(GPT-4o)
│   ├── brief.ts           # 结构化简报类型 + 组装助手
│   └── session.ts         # 会话状态(用于多轮的响应 ID)
├── dist/
│   ├── design             # 编译的二进制文件
│   └── .version           # Git 哈希
└── test/
    └── design.test.ts     # 集成测试
```

### 命令

```bash
# 从结构化简报生成主模型
$D generate --brief "编码评估工具的仪表板。深色主题,奶油色点缀。显示:构建者名称、分数徽章、叙述信、分数卡。目标:技术用户。" --output /tmp/mockup-hero.png

# 生成 3 个设计变体
$D variants --brief "..." --count 3 --output-dir /tmp/mockups/

# 根据反馈迭代现有模型
$D iterate --session /tmp/design-session.json --feedback "使分数卡更大,将叙述移到分数上方" --output /tmp/mockup-v2.png

# 基于视觉的质量检查(返回 PASS/FAIL + 问题)
$D check --image /tmp/mockup-hero.png --brief "带有构建者名称、分数徽章、叙述的仪表板"

# 带质量门 + 自动重试的一次性操作
$D generate --brief "..." --output /tmp/mockup.png --check --retry 1

# 通过 JSON 文件传递结构化简报
$D generate --brief-file /tmp/brief.json --output /tmp/mockup.png

# 生成用于用户审查的比较板 HTML
$D compare --images /tmp/mockups/variant-*.png --output /tmp/design-board.html

# 引导式 API 密钥设置 + 冒烟测试
$D setup
```

**简报输入模式:**
- `--brief "纯文本"` — 自由格式文本提示(简单模式)
- `--brief-file path.json` — 匹配 `DesignBrief` 接口的结构化 JSON(丰富模式)
- 技能构建 JSON 简报文件,将其写入 /tmp,并传递 `--brief-file`

**所有命令都在 `commands.ts` 中注册**,包括 `--check` 和 `--retry` 作为 `generate` 的标志。

### 设计探索工作流(来自工程审查)

工作流是顺序的,不是并行的。PNG 用于视觉探索(面向人类),HTML 线框图用于实现(面向代理):

```
1. $D variants --brief "..." --count 3 --output-dir /tmp/mockups/
   → 生成 2-5 个 PNG 模型变体

2. $D compare --images /tmp/mockups/*.png --output /tmp/design-board.html
   → 生成 HTML 比较板(下面的规格)

3. $B goto file:///tmp/design-board.html
   → 用户在有头 Chrome 中审查所有变体

4. 用户选择最喜欢的,评分,评论,点击 [提交]
   代理轮询:$B eval document.getElementById('status').textContent
   代理读取:$B eval document.getElementById('feedback-result').textContent
   → 没有剪贴板,没有粘贴。代理直接从页面读取反馈。

5. Claude 通过 DESIGN_SKETCH 生成 HTML 线框图,匹配批准的方向
   → 代理从可检查的 HTML 实现,而不是不透明的 PNG
```

### 比较板设计规格(来自 /plan-design-review)

**分类器:APP UI**(以任务为中心的实用页面)。没有产品品牌。

**布局:单列,全宽模型。**每个变体获得完整的视口
宽度以获得最大的图像保真度。用户垂直滚动浏览变体。

```
┌─────────────────────────────────────────────────────────────┐
│  标题栏                                                      │
│  "设计探索" . 项目名称 . "3 个变体"                          │
│  模式指示器:[广泛探索] | [匹配 DESIGN.md]                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              变体 A(全宽)                              │  │
│  │         [ 模型 PNG,max-width: 1200px ]                │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ (●) 选择   ★★★★☆   [你喜欢/不喜欢什么?____]          │  │
│  │            [更像这样]                                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              变体 B(全宽)                              │  │
│  │         [ 模型 PNG,max-width: 1200px ]                │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ ( ) 选择   ★★★☆☆   [你喜欢/不喜欢什么?____]          │  │
│  │            [更像这样]                                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ...(滚动查看更多变体)                                      │
│                                                             │
│  ─── 分隔符 ─────────────────────────────────────────       │
│  整体方向(可选,默认折叠)                                    │
│  [textarea,3 行,聚焦时展开]                                │
│                                                             │
│  ─── 重新生成栏(#f7f7f7 背景)───────────────────────       │
│  "想探索更多?"                                              │
│  [完全不同]  [匹配我的设计]  [自定义:______]                │
│                                          [重新生成 ->]      │
│  ─────────────────────────────────────────────────────────  │
│                                        [ ✓ 提交 ]           │
└─────────────────────────────────────────────────────────────┘
```

**视觉规格:**
- 背景:#fff。没有阴影,没有卡片边框。变体分隔:1px #e5e5e5 线。
- 排版:系统字体堆栈。标题:16px 半粗体。标签:14px 半粗体。反馈占位符:13px 常规 #999。
- 星级评分:5 个可点击的星星,填充=#000,未填充=#ddd。不着色,不动画。
- 单选按钮"选择":明确的最爱选择。每个变体一个,互斥。
- "更像这样"按钮:每个变体,触发以该变体的样式作为种子的重新生成。
- 提交按钮:#000 背景,白色文本,右对齐。单个 CTA。
- 重新生成栏:#f7f7f7 背景,与反馈区域视觉上不同。
- 最大宽度:模型图像居中 1200px。边距:24px 两侧。

**交互状态:**
- 加载(图像准备好之前页面打开):骨架脉冲,每张卡片显示"正在生成变体 A..."。星星/textarea/选择禁用。
- 部分失败(3 个中 2 个成功):显示好的,失败的错误卡片,每个变体有 [重试]。
- 提交后:"反馈已提交!返回你的编码代理。"页面保持打开。
- 重新生成:平滑过渡,淡出旧变体,骨架脉冲,淡入新的。滚动重置到顶部。之前的反馈清除。

**反馈 JSON 结构**(写入隐藏的 #feedback-result 元素):
```json
{
  "preferred": "A",
  "ratings": { "A": 4, "B": 3, "C": 2 },
  "comments": {
    "A": "喜欢间距,标题感觉对",
    "B": "太忙,但颜色调色板不错",
    "C": "情绪完全错误"
  },
  "overall": "选择 A,使 CTA 更大",
  "regenerated": false
}
```

**可访问性:**星级评分键盘可导航(箭头键)。Textarea 已标记("变体 A 的反馈")。提交/重新生成键盘可访问,具有可见的焦点环。所有文本在白色上 #333+。

**响应式:**>1200px:舒适的边距。768-1200px:更紧的边距。<768px:全宽,无水平滚动。

**截图同意(仅首次用于 $D evolve):**"这将向 OpenAI 发送你的实时站点的截图以进行设计演化。[继续] [不再询问]"存储在 ~/.gstack/config.yaml 中作为 design_screenshot_consent。

为什么是顺序的:Codex 对抗性审查确定光栅 PNG 对代理是不透明的(没有 DOM,没有状态,没有可比较的结构)。HTML 线框图保留了回到代码的桥梁。PNG 是为了让人类说"是的,那是对的。"HTML 是为了让代理说"我知道如何构建这个。"

### 关键设计决策

**1. 无状态 CLI,不是守护进程**
Browse 需要持久的 Chromium 实例。Design 只是 API 调用——没有理由需要服务器。多轮迭代的会话状态是写入 `/tmp/design-session-{id}.json` 的 JSON 文件,包含 `previous_response_id`。
- **会话 ID:**从 `${PID}-${timestamp}` 生成,通过 `--session` 标志传递
- **发现:**`generate` 命令创建会话文件并打印其路径;`iterate` 通过 `--session` 读取它
- **清理:**/tmp 中的会话文件是临时的(操作系统清理);不需要显式清理

**2. 结构化简报输入**
简报是技能散文和图像生成之间的接口。技能从设计上下文构建它:
```typescript
interface DesignBrief {
  goal: string;           // "编码评估工具的仪表板"
  audience: string;       // "技术用户,YC 合伙人"
  style: string;          // "深色主题,奶油色点缀,极简"
  elements: string[];     // ["构建者名称","分数徽章","叙述信"]
  constraints?: string;   // "最大宽度 1024px,移动优先"
  reference?: string;     // 现有截图或 DESIGN.md 摘录的路径
  screenType: string;     // "desktop-dashboard" | "mobile-app" | "landing-page" | 等
}
```

**3. 设计技能中默认开启**
技能默认生成模型。模板包含跳过语言:
```
正在生成提议设计的视觉模型...(如果不需要视觉效果,请说"跳过")
```

**4. 视觉质量门**
生成后,可选地通过 GPT-4o 视觉传递图像以检查:
- 文本可读性(标签/标题是否清晰?)
- 布局完整性(是否存在所有请求的元素?)
- 视觉连贯性(它看起来像真实的 UI,而不是拼贴?)
失败时自动重试一次。如果仍然失败,仍然呈现但带有警告。

**5. 输出位置:探索在 /tmp,批准的最终版本在 `docs/designs/`**
- 探索变体进入 `/tmp/gstack-mockups-{session}/`(临时,不提交)
- 只有**用户批准的最终**模型保存到 `docs/designs/`(签入)
- 默认输出目录可通过 CLAUDE.md `design_output_dir` 设置配置
- 文件名模式:`{skill}-{description}-{timestamp}.png`
- 如果不存在则创建 `docs/designs/`(mkdir -p)
- 设计文档引用提交的图像路径
- 始终通过 Read 工具向用户显示(在 Claude Code 中内联渲染图像)
- 这避免了仓库膨胀:只提交批准的设计,而不是每个探索变体
- 回退:如果不在 git 仓库中,保存到 `/tmp/gstack-mockup-{timestamp}.png`

**6. 信任边界确认**
默认开启生成将设计简报文本发送到 OpenAI。这是与现有 HTML 线框图路径(完全本地)相比的新外部数据流。简报仅包含抽象设计描述(目标、样式、元素),从不包含源代码或用户数据。来自 $B 的截图不会发送到 OpenAI(DesignBrief 中的 reference 字段是代理使用的本地文件路径,不会上传到 API)。在 CLAUDE.md 中记录这一点。

**7. 速率限制缓解**
变体生成使用交错并行:通过 `Promise.allSettled()` 和延迟,每个 API 调用间隔 1 秒开始。这避免了图像生成的 5-7 RPM 速率限制,同时仍然比完全串行更快。如果任何调用 429,使用指数退避重试(2s、4s、8s)。

### 模板集成

**添加到现有解析器:**`scripts/resolvers/design.ts`(不是新文件)
- 为 `{{DESIGN_SETUP}}` 占位符添加 `generateDesignSetup()`(镜像 `generateBrowseSetup()`)
- 为 `{{DESIGN_MOCKUP}}` 占位符添加 `generateDesignMockup()`(完整探索工作流)
- 将所有设计解析器保留在一个文件中(与现有代码库约定一致)

**新的 HostPaths 条目:**`types.ts`
```typescript
// claude host:
designDir: '~/.claude/skills/gstack/design/dist'
// codex host:
designDir: '$GSTACK_DESIGN'
```
注意:Codex 运行时设置(`setup` 脚本)还必须导出 `GSTACK_DESIGN` 环境变量,类似于 `GSTACK_BROWSE` 的设置方式。

**`$D` 解析 bash 块**(由 `{{DESIGN_SETUP}}` 生成):
```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
D=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/design/dist/design" ] && D="$_ROOT/.claude/skills/gstack/design/dist/design"
[ -z "$D" ] && D=~/.claude/skills/gstack/design/dist/design
if [ -x "$D" ]; then
  echo "DESIGN_READY: $D"
else
  echo "DESIGN_NOT_AVAILABLE"
fi
```
如果 `DESIGN_NOT_AVAILABLE`:技能回退到 HTML 线框图生成(现有的 `DESIGN_SKETCH` 模式)。设计模型是渐进增强,不是硬性要求。

**现有解析器中的新函数:**`scripts/resolvers/design.ts`
- 为 `{{DESIGN_SETUP}}` 添加 `generateDesignSetup()` — 镜像 `generateBrowseSetup()` 模式
- 为 `{{DESIGN_MOCKUP}}` 添加 `generateDesignMockup()` — 完整的生成+检查+呈现工作流
- 将所有设计解析器保留在一个文件中(与现有代码库约定一致)

### 技能集成(优先顺序)

**1. /office-hours** — 替换视觉草图部分
- 在方法选择(阶段 4)之后,生成主模型 + 2 个变体
- 通过 Read 工具呈现所有三个,要求用户选择
- 如果请求则迭代
- 将选择的模型与设计文档一起保存

**2. /plan-design-review** — "更好的样子"
- 当评分设计维度 <7/10 时,生成显示 10/10 样子的模型
- 并排:当前(通过 $B 截图)vs. 提议(通过 $D 模型)

**3. /design-consultation** — 设计系统预览
- 生成提议设计系统的视觉预览(排版、颜色、组件)
- 用适当的模型替换 /tmp HTML 预览页面

**4. /design-review** — 设计意图比较
- 从计划/DESIGN.md 规格生成"设计意图"模型
- 与实时站点截图比较以获得视觉差异

### 要创建的文件

| 文件 | 目的 |
|------|---------|
| `design/src/cli.ts` | 入口点,命令分发 |
| `design/src/commands.ts` | 命令注册表 |
| `design/src/generate.ts` | 通过 Responses API 生成 GPT 图像 |
| `design/src/iterate.ts` | 带会话状态的多轮迭代 |
| `design/src/variants.ts` | 生成 N 个设计变体 |
| `design/src/check.ts` | 基于视觉的质量门 |
| `design/src/brief.ts` | 结构化简报类型 + 助手 |
| `design/src/session.ts` | 会话状态管理 |
| `design/src/compare.ts` | HTML 比较板生成器 |
| `design/test/design.test.ts` | 集成测试(模拟 OpenAI API) |
| (无 — 添加到现有的 `scripts/resolvers/design.ts`) | `{{DESIGN_SETUP}}` + `{{DESIGN_MOCKUP}}` 解析器 |

### 要修改的文件

| 文件 | 更改 |
|------|--------|
| `scripts/resolvers/types.ts` | 将 `designDir` 添加到 `HostPaths` |
| `scripts/resolvers/index.ts` | 注册 DESIGN_SETUP + DESIGN_MOCKUP 解析器 |
| `package.json` | 添加 `design` 构建命令 |
| `setup` | 与 browse 一起构建 design 二进制文件 |
| `scripts/resolvers/preamble.ts` | 为 Codex 主机添加 `GSTACK_DESIGN` 环境变量导出 |
| `test/gen-skill-docs.test.ts` | 为新解析器更新 DESIGN_SKETCH 测试套件 |
| `setup` | 添加 design 二进制文件构建 + Codex/Kiro 资产链接 |
| `office-hours/SKILL.md.tmpl` | 用 `{{DESIGN_MOCKUP}}` 替换视觉草图部分 |
| `plan-design-review/SKILL.md.tmpl` | 为低分维度添加 `{{DESIGN_SETUP}}` + 模型生成 |

### 要重用的现有代码

| 代码 | 位置 | 用于 |
|------|----------|----------|
| Browse CLI 模式 | `browse/src/cli.ts` | 命令分发架构 |
| `commands.ts` 注册表 | `browse/src/commands.ts` | 单一真实来源模式 |
| `generateBrowseSetup()` | `scripts/resolvers/browse.ts` | `generateDesignSetup()` 的模板 |
| `DESIGN_SKETCH` 解析器 | `scripts/resolvers/design.ts` | `DESIGN_MOCKUP` 解析器的模板 |
| HostPaths 系统 | `scripts/resolvers/types.ts` | 多主机路径解析 |
| 构建管道 | `package.json` 构建脚本 | `bun build --compile` 模式 |

### API 详细信息

**生成:**带 `image_generation` 工具的 OpenAI Responses API
```typescript
const response = await openai.responses.create({
  model: "gpt-4o",
  input: briefToPrompt(brief),
  tools: [{ type: "image_generation", size: "1536x1024", quality: "high" }],
});
// 从响应输出项中提取图像
const imageItem = response.output.find(item => item.type === "image_generation_call");
const base64Data = imageItem.result; // base64 编码的 PNG
fs.writeFileSync(outputPath, Buffer.from(base64Data, "base64"));
```

**迭代:**带 `previous_response_id` 的相同 API
```typescript
const response = await openai.responses.create({
  model: "gpt-4o",
  input: feedback,
  previous_response_id: session.lastResponseId,
  tools: [{ type: "image_generation" }],
});
```
**注意:**通过 `previous_response_id` 进行多轮图像迭代是一个需要原型验证的假设。Responses API 支持对话线程,但它是否保留生成图像的视觉上下文以进行编辑式迭代在文档中未确认。**回退:**如果多轮不起作用,`iterate` 回退到在单个提示中使用原始简报 + 累积反馈重新生成。

**检查:**GPT-4o 视觉
```typescript
const check = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{
    role: "user",
    content: [
      { type: "image_url", image_url: { url: `data:image/png;base64,${imageData}` } },
      { type: "text", text: `检查这个 UI 模型。简报:${brief}。文本可读吗?所有元素都存在吗?它看起来像真实的 UI 吗?返回 PASS 或 FAIL 及问题。` }
    ]
  }]
});
```

**成本:**每个设计会话约 $0.10-$0.40(1 个主模型 + 2 个变体 + 1 个质量检查 + 1 次迭代)。与每次技能调用中已有的 LLM 成本相比可以忽略不计。

### 认证(通过冒烟测试验证)

**Codex OAuth 令牌不适用于图像生成。**测试于 2026-03-26:Images API 和 Responses API 都拒绝 `~/.codex/auth.json` access_token,显示"缺少范围:api.model.images.request"。Codex CLI 也没有原生 imagegen 功能。

**认证解析顺序:**
1. 读取 `~/.gstack/openai