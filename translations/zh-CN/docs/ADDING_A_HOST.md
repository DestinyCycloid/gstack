# 向 gstack 添加新主机

gstack 使用声明式主机配置系统。每个支持的 AI 编码代理
(Claude、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw) 都被定义为
类型化的 TypeScript 配置对象。添加新主机意味着创建一个文件
并重新导出它。生成器、设置或工具无需更改任何代码。

## 工作原理

```
hosts/
├── claude.ts        # 主要主机
├── codex.ts         # OpenAI Codex CLI
├── factory.ts       # Factory Droid
├── kiro.ts          # Amazon Kiro
├── opencode.ts      # OpenCode
├── slate.ts         # Slate (Random Labs)
├── cursor.ts        # Cursor
├── openclaw.ts      # OpenClaw (混合:配置 + 适配器)
└── index.ts         # 注册表:导入所有,派生 Host 类型
```

每个配置文件导出一个 `HostConfig` 对象,告诉生成器:
- 生成的技能放在哪里(路径)
- 如何转换前置元数据(允许列表/拒绝列表字段)
- 要重写哪些 Claude 特定引用(路径、工具名称)
- 检测哪个二进制文件以进行自动安装
- 要抑制哪些解析器部分
- 安装时要符号链接哪些资源

生成器、设置脚本、平台检测、卸载、健康检查、工作树
复制和测试都从这些配置中读取。它们都没有针对每个主机的代码。

## 分步指南:添加新主机

### 1. 创建配置文件

复制现有配置作为起点。`hosts/opencode.ts` 是一个很好的
最小示例。`hosts/factory.ts` 展示了工具重写和条件字段。
`hosts/openclaw.ts` 展示了具有不同工具模型的主机的适配器模式。

创建 `hosts/myhost.ts`:

```typescript
import type { HostConfig } from '../scripts/host-config';

const myhost: HostConfig = {
  name: 'myhost',
  displayName: 'MyHost',
  cliCommand: 'myhost',        // `command -v` 检测的二进制名称
  cliAliases: [],              // 替代二进制名称

  globalRoot: '.myhost/skills/gstack',
  localSkillRoot: '.myhost/skills/gstack',
  hostSubdir: '.myhost',
  usesEnvVars: true,           // 仅对 Claude 为 false(使用字面 ~ 路径)

  frontmatter: {
    mode: 'allowlist',         // 'allowlist' 仅保留列出的字段
    keepFields: ['name', 'description'],
    descriptionLimit: null,    // 对于有限制的主机设置为 1024
  },

  generation: {
    generateMetadata: false,   // 仅对 Codex 为 true (openai.yaml)
    skipSkills: ['codex'],     // codex 技能仅限 Claude
  },

  pathRewrites: [
    { from: '~/.claude/skills/gstack', to: '~/.myhost/skills/gstack' },
    { from: '.claude/skills/gstack', to: '.myhost/skills/gstack' },
    { from: '.claude/skills', to: '.myhost/skills' },
  ],

  runtimeRoot: {
    globalSymlinks: ['bin', 'browse/dist', 'browse/bin', 'gstack-upgrade', 'ETHOS.md'],
    globalFiles: { 'review': ['checklist.md', 'TODOS-format.md'] },
  },

  install: {
    prefixable: false,
    linkingStrategy: 'symlink-generated',
  },

  learningsMode: 'basic',
};

export default myhost;
```

### 2. 在索引中注册

编辑 `hosts/index.ts`:

```typescript
import myhost from './myhost';

// 添加到 ALL_HOST_CONFIGS 数组:
export const ALL_HOST_CONFIGS: HostConfig[] = [
  claude, codex, factory, kiro, opencode, slate, cursor, openclaw, myhost
];

// 添加到重新导出:
export { claude, codex, factory, kiro, opencode, slate, cursor, openclaw, myhost };
```

### 3. 添加到 .gitignore

将 `.myhost/` 添加到 `.gitignore`(生成的技能文档被 gitignore)。

### 4. 生成并验证

```bash
# 为新主机生成技能文档
bun run gen:skill-docs --host myhost

# 验证输出存在且没有 .claude/skills 泄漏
ls .myhost/skills/gstack-*/SKILL.md
grep -r ".claude/skills" .myhost/skills/ | head -5
# (应该为空)

# 为所有主机生成(包括新主机)
bun run gen:skill-docs --host all

# 健康仪表板显示新主机
bun run skill:check
```

### 5. 运行测试

```bash
bun test test/gen-skill-docs.test.ts
bun test test/host-config.test.ts
```

参数化冒烟测试会自动识别新主机。无需编写
测试代码。它们验证:输出存在、无路径泄漏、有效的前置元数据、
新鲜度检查通过、codex 技能被排除。

### 6. 更新 README.md

在适当的部分添加新主机的安装说明。

## 配置字段参考

参见 `scripts/host-config.ts` 获取完整的 `HostConfig` 接口,其中包含
每个字段的 JSDoc 注释。

关键字段:

| 字段 | 用途 |
|-------|---------|
| `frontmatter.mode` | `allowlist`(仅保留列出的)或 `denylist`(删除列出的) |
| `frontmatter.descriptionLimit` | 最大字符数,`null` 表示无限制 |
| `frontmatter.descriptionLimitBehavior` | `error`(构建失败)、`truncate`、`warn` |
| `frontmatter.conditionalFields` | 根据模板值添加字段(例如,sensitive → disable-model-invocation) |
| `frontmatter.renameFields` | 重命名模板字段(例如,voice-triggers → triggers) |
| `pathRewrites` | 对内容进行字面 replaceAll。顺序很重要。 |
| `toolRewrites` | 重写 Claude 工具名称(例如,"use the Bash tool" → "run this command") |
| `suppressedResolvers` | 对此主机返回空的解析器函数 |
| `coAuthorTrailer` | 提交的 Git 共同作者字符串 |
| `boundaryInstruction` | 跨模型调用的反提示注入警告 |
| `adapter` | 用于复杂转换的适配器模块路径 |

## 适配器模式(用于具有不同工具模型的主机)

如果字符串替换工具重写不够用(主机具有根本
不同的工具语义),请使用适配器模式。参见 `hosts/openclaw.ts`
和 `scripts/host-adapters/openclaw-adapter.ts`。

适配器在所有通用重写之后作为后处理步骤运行。它
导出 `transform(content: string, config: HostConfig): string`。

## 验证

`scripts/host-config.ts` 中的 `validateHostConfig()` 函数检查:
- 名称:小写字母数字加连字符
- CLI 命令:字母数字加连字符/下划线
- 路径:仅安全字符(字母数字、`.`、`/`、`$`、`{}`、`~`、`-`、`_`)
- 配置之间没有重复的名称、hostSubdirs 或 globalRoots

运行 `bun run scripts/host-config-export.ts validate` 检查所有配置。