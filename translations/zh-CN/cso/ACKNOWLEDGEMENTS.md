# 致谢

/cso v2 的开发参考了安全审计领域的多项研究成果。特别感谢:

- **[Sentry Security Review](https://github.com/getsentry/skills)** — 基于置信度的报告系统(仅报告 HIGH 置信度的发现)以及"研究后报告"方法论(追踪数据流、检查上游验证)验证了我们的 8/10 日常置信度门槛。TimOnWeb 将其评为测试的 5 个安全技能中唯一值得安装的。
- **[Trail of Bits Skills](https://github.com/trailofbits/skills)** — 审计上下文构建方法论(在寻找漏洞前先建立心智模型)直接启发了 Phase 0。他们的变体分析概念(发现一个漏洞?在整个代码库中搜索相同模式)启发了 Phase 12 的变体分析步骤。
- **[Shannon by Keygraph](https://github.com/KeygraphHQ/shannon)** — 自主 AI 渗透测试工具,在 XBOW 基准测试中达到 96.15% 的成绩(100/104 个漏洞利用)。验证了 AI 可以进行真正的安全测试,而不仅仅是清单扫描。我们的 Phase 12 主动验证是 Shannon 实时操作的静态分析版本。
- **[afiqiqmal/claude-security-audit](https://github.com/afiqiqmal/claude-security-audit)** — AI/LLM 特定的安全检查(提示注入、RAG 投毒、工具调用权限)启发了 Phase 7。他们的框架级自动检测(检测"Next.js"而不仅仅是"Node/TypeScript")启发了 Phase 0 的框架检测步骤。
- **[Snyk ToxicSkills Research](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)** — 研究发现 36% 的 AI 代理技能存在安全缺陷,13.4% 是恶意的,这启发了 Phase 8(Skill 供应链扫描)。
- **[Daniel Miessler's Personal AI Infrastructure](https://github.com/danielmiessler/Personal_AI_Infrastructure)** — 事件响应手册和保护文件概念为修复和 LLM 安全阶段提供了参考。
- **[McGo/claude-code-security-audit](https://github.com/McGo/claude-code-security-audit)** — 生成可共享报告和可操作史诗的想法影响了我们的报告格式演进。
- **[Claude Code Security Pack](https://dev.to/myougatheaxo/automate-owasp-security-audits-with-claude-code-security-pack-4mah)** — 模块化方法(独立的 /security-audit、/secret-scanner、/deps-check 技能)验证了这些是不同的关注点。我们的统一方法牺牲了模块化以换取跨阶段推理能力。
- **[Anthropic Claude Code Security](https://www.anthropic.com/news/claude-code-security)** — 多阶段验证和置信度评分验证了我们的并行发现验证方法。在开源项目中发现了 500+ 个零日漏洞。
- **[@gus_argon](https://x.com/gus_aragon/status/2035841289602904360)** — 识别了 v1 的关键盲点:无技术栈检测(运行所有语言模式)、使用 bash grep 而非 Claude Code 的 Grep 工具、`| head -20` 静默截断结果,以及前言冗余。这些直接塑造了 v2 的技术栈优先方法和 Grep 工具强制要求。