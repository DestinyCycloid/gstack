# gbrain-sync 错误查询

`gstack-brain-*` 可能打印的每个错误消息,包含问题、原因和修复方法。

通过 `BRAIN_SYNC:` 后的前缀或命令输出中的二进制文件名搜索此文件。

---

## `BRAIN_SYNC: brain repo detected: <url>`

**问题。** 你在一台有 `~/.gstack-brain-remote.txt`(从另一台机器复制)但在 `~/.gstack/.git` 没有本地 git 仓库的机器上。

**原因。** 你已在其他地方设置了 GBrain 同步,但你的 gstack 尚未在此机器上恢复。

**修复。**
```bash
gstack-brain-restore
```
这会将仓库拉取到 `~/.gstack/` 并重新注册合并驱动程序。

如果你不想在此处恢复,可以使用以下命令关闭提示:
```bash
gstack-config set gbrain_sync_mode_prompted true
```

---

## `BRAIN_SYNC: blocked: <pattern-family>:<snippet>`

**问题。** 同步已停止,因为密钥扫描器在暂存文件中检测到凭据形状的内容。队列已保留;没有任何内容被推送。

**原因。** 其中一个预提交密钥模式匹配了文件内容——可能是嵌入在 JSON 中的 AWS 密钥、GitHub 令牌、OpenAI 密钥、PEM 块、JWT 或 bearer 令牌。

**修复(三个选项)。**

1. **如果是真实密钥**:编辑有问题的文件以删除密钥,然后重新运行任何技能以重试同步。

2. **如果模式是误报**(例如,你的学习内容包含一个你*想要*发布的示例字符串中的 GitHub 令牌模式):
   ```bash
   gstack-brain-sync --skip-file <path>
   ```
   这会永久排除该路径,使其不参与未来的同步。

3. **如果你想完全放弃此同步批次**(重新开始):
   ```bash
   gstack-brain-sync --drop-queue --yes
   ```
   这会清除队列而不提交。未来的写入将正常重新填充队列。

---

## `BRAIN_SYNC: push failed: auth.`

**问题。** Git 推送被拒绝,因为你与远程的身份验证已过期或缺失。

**原因。** 使用当前凭据无法访问远程。

**修复。** 根据你的远程刷新身份验证:

- **GitHub**: `gh auth status`(如果需要,然后运行 `gh auth refresh`)
- **GitLab**: `glab auth status`
- **其他**: `git remote -v` + 检查 SSH 密钥或凭据助手

修复身份验证后,运行任何技能以自动重试同步。

---

## `BRAIN_SYNC: push failed: <first-line-of-error>`

**问题。** 推送因身份验证以外的原因失败。git 错误的第一行出现在冒号之后。

**原因。** 可能是网络问题、推送被拒绝(远程领先)、服务器 500 错误或仓库访问权限被撤销。

**修复。** 查看 `~/.gstack/.brain-sync-status.json` 以获取更多详细信息,或运行:
```bash
cd ~/.gstack && git status && git push origin HEAD
```
以查看 git 的完整错误。队列在任何推送尝试后都会被清除,但你的本地提交仍然存在——下次技能运行将重试推送。

---

## `gstack-brain-init: ~/.gstack/.git is already a git repo pointing at <url>`

**问题。** 你尝试使用与现有远程不匹配的远程 URL 进行初始化。

**原因。** 你已经使用不同的远程运行过 `gstack-brain-init`。

**修复。** 可以:

- 使用现有远程:不带 `--remote` 运行 `gstack-brain-init`,或使用匹配的 URL。
- 切换远程:先运行 `gstack-brain-uninstall`,然后使用新 URL 重新初始化。这不会删除你的数据。

---

## `Remote not reachable: <url>`

**问题。** 初始化无法访问 git 远程以验证连接性。

**原因。** URL 错误、缺少身份验证、网络问题。

**修复。** 手动测试:
```bash
git ls-remote <url>
```
如果失败,检查:
- URL 拼写
- GitHub: `gh auth status`
- GitLab: `glab auth status`
- 专用网络 / VPN / DNS

---

## `gstack-brain-init: failed to create or find '<name>'`

**问题。** 通过 `gh repo create` 自动创建仓库失败,并且通过 `gh repo view` 也无法发现该仓库。

**原因。** `gh` 未经身份验证,具有该名称的仓库已由其他人拥有,或你的 GitHub 账户达到配额限制。

**修复。**
```bash
gh auth status
```
如果未经身份验证,运行 `gh auth login`。如果仓库名称冲突,传递不同的名称:
```bash
gstack-brain-init --remote git@github.com:YOURUSER/custom-name.git
```

---

## `gstack-brain-restore: ~/.gstack/.git already points at <url>`

**问题。** 你尝试从与现有 git 配置不匹配的 URL 恢复。

**原因。** 来自先前使用不同远程初始化的过时 `.git`。

**修复。** 运行 `gstack-brain-uninstall`,然后重新运行 `gstack-brain-restore <url>`。

---

## `gstack-brain-restore: ~/.gstack/ has existing allowlisted files that would be clobbered`

**问题。** 你正在尝试恢复,但 `~/.gstack/` 已包含将被覆盖的学习内容或计划。

**原因。** (a) 此机器已从预同步 gstack 会话中累积了状态,或 (b) 先前失败的恢复留下了部分状态。

**修复(三个选项)。**

1. **如果此机器的状态应成为新的真实状态**:运行 `gstack-brain-init` 而不是恢复——这会从此机器的状态创建一个全新的 brain 仓库。

2. **如果你想采用远程并丢弃此机器的状态**:先备份 `~/.gstack/projects/`,然后删除有问题的文件并重新运行恢复。

3. **如果你想合并**:没有自动合并功能。手动将 `~/.gstack/` 中的学习内容复制到已启用同步的机器上正在运行的 gstack 中,然后在此处恢复。

---

## `gstack-brain-restore: <url> does not look like a gstack-brain repo`

**问题。** 克隆成功,但仓库缺少 `.brain-allowlist` 和 `.gitattributes`。

**原因。** 你将恢复指向了一个随机的 git 仓库,或有人从 brain 仓库中删除了规范配置文件。

**修复。** 验证 URL。如果正确,运行 `gstack-brain-init --remote <url>` 以重新生成规范配置。

---

## 没有任何内容在同步,但我期望它同步

**这不是错误,但是一个常见的陷阱。** 按顺序检查:

1. `gstack-brain-sync --status` — 模式是 `off` 吗?
2. `~/.gstack/.git` 存在吗?
3. `gstack-config get gbrain_sync_mode` — 应该是 `full` 或 `artifacts-only`。
4. 你期望同步的文件 — 它在允许列表中吗?
   `cat ~/.gstack/.brain-allowlist`
5. 隐私类过滤器 — 如果模式是 `artifacts-only`,行为文件(时间线、开发者配置文件)会被有意跳过。

如果所有这些看起来都正确,运行:
```bash
gstack-brain-sync --discover-new
gstack-brain-sync --once
```
以强制排空。