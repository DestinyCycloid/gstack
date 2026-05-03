# 计划:快照下拉菜单/自动完成交互元素检测

## 问题

`snapshot -i` 在现代 Web 应用中会遗漏下拉菜单/自动完成项。这些元素:
1. 通常是带有点击处理器但没有语义化 ARIA 角色的 `<div>`/`<li>`
2. 存在于动态创建的传送门/弹出层(浮动容器)中
3. 不会出现在 Playwright 的可访问性树(`ariaSnapshot()`)中

`-C` 标志(光标交互扫描)是为此设计的,但:
- 需要单独的标志 — 使用 `-i` 的代理不会自动获得它
- 跳过具有 ARIA 角色的元素(即使 ARIA 树遗漏了它们)
- 不优先处理下拉项所在的弹出层/传送门容器

## 根本原因

Playwright 的 `ariaSnapshot()` 从浏览器的可访问性树构建。动态渲染的弹出层(React 传送门、Radix Popover 等)可能不在可访问性树中,如果:
- 组件未设置 ARIA 角色
- 传送门在作用域 `body` 定位器的子树时序之外渲染
- DOM 变更后浏览器尚未更新可访问性树

## 变更

### 1. 使用 `-i` 标志自动启用光标交互扫描

**文件:** `browse/src/snapshot.ts`

当传递 `-i`(交互)时,自动包含光标交互扫描。这意味着代理在请求交互元素时始终能看到可点击的非 ARIA 元素。

`-C` 标志保留作为非交互快照的独立选项。

```
if (opts.interactive) {
  opts.cursorInteractive = true;
}
```

### 2. 添加弹出层/传送门优先扫描

**文件:** `browse/src/snapshot.ts`(在光标交互评估块内)

在通用 cursor:pointer 扫描之前,专门扫描可见的浮动容器(弹出层、下拉菜单、菜单)并将它们的所有直接子元素包含为交互元素:

浮动容器的检测启发式规则:
- `position: fixed` 或 `position: absolute` 且 `z-index >= 10`
- 具有 `role="listbox"`、`role="menu"`、`role="dialog"`、`role="tooltip"`、`[data-radix-popper-content-wrapper]`、`[data-floating-ui-portal]` 等
- 最近出现在 DOM 中(不在初始页面加载时)
- 可见(`offsetParent !== null` 或 `position: fixed`)

对于每个浮动容器,包含满足以下条件的子元素:
- 有文本内容
- 可见
- 具有 cursor:pointer 或 onclick 或 role="option" 或 role="menuitem"
- 为清晰起见,用 `popover-child` 原因标记

### 3. 移除光标交互扫描中的 `hasRole` 跳过

**文件:** `browse/src/snapshot.ts`

当前: `if (hasRole) continue;` — 跳过任何具有 ARIA 角色的元素,假设 ARIA 树已经捕获了它。

问题:如果 ARIA 树遗漏了该元素(时序、传送门、错误的 DOM 结构),它会在两个系统中都被遗漏。

修复:仅当元素的角色在 `INTERACTIVE_ROLES` 中且实际在主 refMap 中被捕获时才跳过。否则包含它。

由于我们无法从 `page.evaluate()` 内部轻松检查 refMap,更简单的修复是:对于检测到的浮动容器内的元素,完全移除 `hasRole` 跳过。对于浮动容器外的元素,保持 `hasRole` 跳过不变(以避免正常页面内容中的重复)。

### 4. 添加下拉菜单测试夹具和测试

**文件:** `browse/test/fixtures/dropdown.html`

HTML 页面包含:
- 一个在聚焦/输入时显示下拉菜单的组合框输入
- 作为带有点击处理器的 `<div>` 的下拉项(无 ARIA 角色)
- 作为带有 `role="option"` 的 `<li>` 的下拉项
- 一个 React 传送门风格的容器(`position: fixed`,高 z-index)

**文件:** `browse/test/snapshot.test.ts`

新测试用例:
- 下拉页面上的 `snapshot -i` 通过光标扫描找到下拉项
- 下拉页面上的 `snapshot -i` 包含 popover-child 元素
- 来自下拉扫描的 `@c` 引用是可点击的
- 即使 ARIA 树遗漏了浮动容器内具有 ARIA 角色的元素,也会被捕获

## 发布风险

**低。** `-C` 扫描是附加的 — 它只添加 `@c` 引用,从不删除 `@e` 引用。使用 `-i` 自动启用它的更改会增加输出大小,但代理已经处理混合引用类型。

**一个担忧:** `-C` 扫描查询所有元素(`document.querySelectorAll('*')`),这在大型页面上可能很慢。对于弹出层特定扫描,我们限制为检测到的浮动容器内的元素,这很快(小子树)。

## 测试

```bash
cd /data/gstack/browse && bun test snapshot
```

## 更改的文件

1. `browse/src/snapshot.ts` — 使用 -i 自动启用 -C,弹出层扫描,移除浮动容器中的 hasRole 跳过
2. `browse/test/fixtures/dropdown.html` — 新测试夹具
3. `browse/test/snapshot.test.ts` — 新下拉菜单/弹出层测试用例