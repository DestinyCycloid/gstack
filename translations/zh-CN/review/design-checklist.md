# 设计审查清单（精简版）

> **DESIGN_METHODOLOGY 的子集** — 在此处添加项目时，也需要更新 `scripts/gen-skill-docs.ts` 中的 `generateDesignMethodology()`，反之亦然。

## 说明

此清单适用于 **diff 中的源代码** — 而非渲染输出。阅读每个更改的前端文件（完整文件，而非仅 diff 片段）并标记反模式。

**触发条件：** 仅当 diff 涉及前端文件时运行此清单。使用 `gstack-diff-scope` 检测：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

如果 `SCOPE_FRONTEND=false`，则静默跳过整个设计审查。

**DESIGN.md 校准：** 如果仓库根目录存在 `DESIGN.md` 或 `design-system.md`，首先阅读它。所有发现都根据项目声明的设计系统进行校准。DESIGN.md 中明确认可的模式不会被标记。如果不存在 DESIGN.md，则使用通用设计原则。

---

## 置信度层级

每个项目都标记有检测置信度级别：

- **[HIGH]** — 可通过 grep/模式匹配可靠检测。确定性发现。
- **[MEDIUM]** — 可通过模式聚合或启发式检测。标记为发现但预期会有一些噪音。
- **[LOW]** — 需要理解视觉意图。呈现为："可能的问题 — 请视觉验证或运行 /design-review。"

---

## 分类

**AUTO-FIX**（仅机械性 CSS 修复 — HIGH 置信度，无需设计判断）：
- `outline: none` 无替代方案 → 添加 `outline: revert` 或 `&:focus-visible { outline: 2px solid currentColor; }`
- 新 CSS 中的 `!important` → 移除并修复特异性
- 正文文本的 `font-size` < 16px → 提升至 16px

**ASK**（其他所有 — 需要设计判断）：
- 所有 AI 生成痕迹发现、排版结构、间距选择、交互状态缺失、DESIGN.md 违规

**LOW 置信度项目** → 呈现为"可能：[描述]。请视觉验证或运行 /design-review。"永不 AUTO-FIX。

---

## 输出格式

```
Design Review: N issues (X auto-fixable, Y need input, Z possible)

**AUTO-FIXED:**
- [file:line] 问题 → 已应用修复

**NEEDS INPUT:**
- [file:line] 问题描述
  Recommended fix: 建议修复

**POSSIBLE (verify visually):**
- [file:line] 可能的问题 — 使用 /design-review 验证
```

可选：`test_stub` — 使用项目测试框架为此发现提供的骨架测试代码。

如果未发现问题：`Design Review: No issues found.`

如果没有前端文件更改：静默跳过，无输出。

---

## 类别

### 1. AI 生成痕迹检测（6 项）— 最高优先级

这些是 AI 生成 UI 的明显标志，任何受尊敬工作室的设计师都不会发布。

- **[MEDIUM]** 紫色/紫罗兰/靛蓝渐变背景或蓝到紫的配色方案。查找 `linear-gradient`，其值在 `#6366f1`–`#8b5cf6` 范围内，或解析为紫色/紫罗兰的 CSS 自定义属性。

- **[LOW]** 三列特性网格：彩色圆圈中的图标 + 粗体标题 + 2 行描述，对称重复 3 次。查找恰好有 3 个子元素的 grid/flex 容器，每个子元素包含圆形元素 + 标题 + 段落。

- **[LOW]** 彩色圆圈中的图标作为区域装饰。查找具有 `border-radius: 50%` + 背景色的元素，用作图标的装饰容器。

- **[HIGH]** 所有内容居中：所有标题、描述和卡片上的 `text-align: center`。Grep `text-align: center` 密度 — 如果 >60% 的文本容器使用居中对齐，则标记。

- **[MEDIUM]** 每个元素统一的圆润 border-radius：相同的大半径（16px+）统一应用于卡片、按钮、输入框、容器。聚合 `border-radius` 值 — 如果 >80% 使用相同的 ≥16px 值，则标记。

- **[MEDIUM]** 通用英雄文案："Welcome to [X]"、"Unlock the power of..."、"Your all-in-one solution for..."、"Revolutionize your..."、"Streamline your workflow"。在 HTML/JSX 内容中 Grep 这些模式。

### 2. 排版（4 项）

- **[HIGH]** 正文文本 `font-size` < 16px。在 `body`、`p`、`.text` 或基础样式上 Grep `font-size` 声明。低于 16px（或当基础为 16px 时的 1rem）的值会被标记。

- **[HIGH]** diff 中引入超过 3 个字体系列。计算不同的 `font-family` 声明。如果更改的文件中出现 >3 个唯一系列，则标记。

- **[HIGH]** 标题层级跳级：`h1` 后跟 `h3`，同一文件/组件中没有 `h2`。检查 HTML/JSX 中的标题标签。

- **[HIGH]** 黑名单字体：Papyrus、Comic Sans、Lobster、Impact、Jokerman。在 `font-family` 中 Grep 这些名称。

### 3. 间距与布局（4 项）

- **[MEDIUM]** 当 DESIGN.md 指定间距比例时，任意间距值不在 4px 或 8px 比例上。根据声明的比例检查 `margin`、`padding`、`gap` 值。仅在 DESIGN.md 定义比例时标记。

- **[MEDIUM]** 固定宽度无响应式处理：容器上的 `width: NNNpx` 没有 `max-width` 或 `@media` 断点。在移动端存在水平滚动风险。

- **[MEDIUM]** 文本容器缺少 `max-width`：正文文本或段落容器没有设置 `max-width`，允许行 >75 个字符。检查文本包装器上的 `max-width`。

- **[HIGH]** 新 CSS 规则中的 `!important`。在添加的行中 Grep `!important`。几乎总是应该正确修复的特异性逃生舱。

### 4. 交互状态（3 项）

- **[MEDIUM]** 交互元素（按钮、链接、输入框）缺少 hover/focus 状态。检查新交互元素样式是否存在 `:hover` 和 `:focus` 或 `:focus-visible` 伪类。

- **[HIGH]** `outline: none` 或 `outline: 0` 没有替代焦点指示器。Grep `outline:\s*none` 或 `outline:\s*0`。这会移除键盘可访问性。

- **[LOW]** 交互元素上的触摸目标 < 44px。检查按钮和链接上的 `min-height`/`min-width`/`padding`。需要从多个属性计算有效大小 — 仅从代码判断置信度低。

### 5. DESIGN.md 违规（3 项，有条件）

仅在存在 `DESIGN.md` 或 `design-system.md` 时应用：

- **[MEDIUM]** 颜色不在声明的调色板中。将更改的 CSS 中的颜色值与 DESIGN.md 中定义的调色板进行比较。

- **[MEDIUM]** 字体不在声明的排版部分中。将 `font-family` 值与 DESIGN.md 的字体列表进行比较。

- **[MEDIUM]** 间距值超出声明的比例。将 `margin`/`padding`/`gap` 值与 DESIGN.md 的间距比例进行比较。

---

## 抑制

不要标记：
- DESIGN.md 中明确记录为有意选择的模式
- 第三方/供应商 CSS 文件（node_modules、vendor 目录）
- CSS 重置或 normalize 样式表
- 测试固件文件
- 生成的/压缩的 CSS