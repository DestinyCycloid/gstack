# 设计系统 — gstack

## 产品背景
- **这是什么:** gstack 的社区网站 — 一个将 Claude Code 转变为虚拟工程团队的 CLI 工具
- **目标用户:** 发现 gstack 的开发者,现有社区成员
- **领域/行业:** 开发者工具(同类产品:Linear、Raycast、Warp、Zed)
- **项目类型:** 社区仪表板 + 营销网站

## 美学方向
- **方向:** 工业/实用主义 — 功能优先,数据密集,等宽字体作为个性字体
- **装饰程度:** 有意为之 — 在表面使用微妙的噪点/颗粒纹理以增加质感
- **氛围:** 由注重工艺的人打造的严肃工具。温暖,而非冰冷。CLI 传统即是品牌。
- **参考网站:** formulae.brew.sh(竞品,但我们的是实时交互的)、Linear(深色 + 克制)、Warp(温暖的强调色)

## 字体排版
- **展示/主标题:** Satoshi(Black 900 / Bold 700)— 几何感与温暖并存,独特的字母形态(小写的 'a' 和 'g')。不是 Inter,不是 Geist。从 Fontshare CDN 加载。
- **正文:** DM Sans(Regular 400 / Medium 500 / Semibold 600)— 简洁、易读,比几何展示字体稍显友好。从 Google Fonts 加载。
- **UI/标签:** DM Sans(与正文相同)
- **数据/表格:** JetBrains Mono(Regular 400 / Medium 500)— 个性字体。支持 tabular-nums。等宽字体应该突出显示,而不是隐藏在代码块中。从 Google Fonts 加载。
- **代码:** JetBrains Mono
- **加载:** Google Fonts 用于 DM Sans + JetBrains Mono,Fontshare 用于 Satoshi。使用 `display=swap`。
- **字号:**
  - Hero: 72px / clamp(40px, 6vw, 72px)
  - H1: 48px
  - H2: 32px
  - H3: 24px
  - H4: 18px
  - Body: 16px
  - Small: 14px
  - Caption: 13px
  - Micro: 12px
  - Nano: 11px(JetBrains Mono 标签)

## 颜色
- **方法:** 克制 — 琥珀色强调色稀有且有意义。仪表板数据使用颜色;框架保持中性。
- **主色(深色模式):** amber-500 #F59E0B — 温暖、充满活力,读起来像"终端光标"
- **主色(浅色模式):** amber-600 #D97706 — 更深以在白色背景上形成对比
- **主文本强调色(深色模式):** amber-400 #FBBF24
- **主文本强调色(浅色模式):** amber-700 #B45309
- **中性色:** 冷色调锌灰
  - zinc-50: #FAFAFA(最浅)
  - zinc-400: #A1A1AA
  - zinc-600: #52525B
  - zinc-800: #27272A
  - Surface(深色): #141414
  - Base(深色): #0C0C0C
  - Surface(浅色): #FFFFFF
  - Base(浅色): #FAFAF9
- **语义色:** success #22C55E、warning #F59E0B、error #EF4444、info #3B82F6
- **深色模式:** 默认。近黑色基底(#0C0C0C),表面卡片为 #141414,边框为 #262626。
- **浅色模式:** 温暖的石色基底(#FAFAF9),白色表面卡片,石色边框(#E7E5E4)。琥珀色强调色转为 amber-600 以形成对比。

## 间距
- **基础单位:** 4px
- **密度:** 舒适 — 不拥挤(不是 Bloomberg Terminal),不宽松(不是营销网站)
- **比例:** 2xs(2px) xs(4px) sm(8px) md(16px) lg(24px) xl(32px) 2xl(48px) 3xl(64px)

## 布局
- **方法:** 仪表板采用网格规范,落地页采用编辑式主视觉
- **网格:** lg+ 时 12 列,移动端 1 列
- **最大内容宽度:** 1200px(6xl)
- **圆角半径:** sm:4px、md:8px、lg:12px、full:9999px
  - 卡片/面板: lg(12px)
  - 按钮/输入框: md(8px)
  - 徽章/药丸: full(9999px)
  - 技能条: sm(4px)

## 动效
- **方法:** 最小化功能性 — 仅使用有助于理解的过渡效果。仪表板的实时动态即是动效。
- **缓动:** enter(ease-out / cubic-bezier(0.16,1,0.3,1)) exit(ease-in) move(ease-in-out)
- **持续时间:** micro(50-100ms) short(150ms) medium(250ms) long(400ms)
- **动画元素:** 实时动态点脉冲(2s infinite)、技能条填充(600ms ease-out)、悬停状态(150ms)

## 颗粒纹理
在整个页面应用微妙的噪点叠加以增加质感:
- 深色模式: opacity 0.03
- 浅色模式: opacity 0.02
- 在 body::after 上使用 SVG feTurbulence 滤镜作为 CSS background-image
- pointer-events: none、position: fixed、z-index: 9999

## 决策日志
| 日期 | 决策 | 理由 |
|------|----------|-----------|
| 2026-03-21 | 初始设计系统 | 由 /design-consultation 创建。工业美学,温暖的琥珀色强调色,Satoshi + DM Sans + JetBrains Mono。 |
| 2026-03-21 | 浅色模式 amber-600 | amber-500 在白色背景上太亮/褪色;amber-700 太棕/赭色。amber-600 是最佳选择。 |
| 2026-03-21 | 颗粒纹理 | 为平面深色表面增加质感。防止"通用 SaaS 模板"的同质化。 |