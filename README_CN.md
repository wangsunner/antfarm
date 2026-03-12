# Antfarm

<img src="https://raw.githubusercontent.com/snarktank/antfarm/main/landing/logo.jpeg" alt="Antfarm" width="80">

**只需一条命令，在 [OpenClaw](https://docs.openclaw.ai) 中构建你的代理团队。**

你不需要招聘开发团队，只需要定义一个。Antfarm 为你提供了一组专业的 AI 代理——规划者、开发者、验证者、测试者、审查者——它们在可靠、可重复的工作流中协同工作。一次安装，无需任何基础设施。

### 安装

```bash
curl -fsSL https://raw.githubusercontent.com/snarktank/antfarm/v0.5.1/scripts/install.sh | bash
```

或者直接告诉你的 OpenClaw 代理：**"install github.com/snarktank/antfarm"**

就这么简单。运行 `antfarm workflow list` 查看可用的工作流。

> **不在 npm 上。** Antfarm 从 GitHub 安装，而不是 npm 注册表。npm 上有一个无关的 `antfarm` 包——那不是这个。

> **需要 Node.js >= 22。** 如果 `antfarm` 出现 `node:sqlite` 错误，请确保你运行的是真实的 Node.js 22+，而不是 Bun 的 Node 包装器（见 [#54](https://github.com/snarktank/antfarm/issues/54)）。

---

## 你将获得：代理团队工作流

### feature-dev `7 个代理`

提交一个功能请求。得到一个经过测试的 PR。规划者将你的任务分解为多个故事。每个故事都会单独实现、验证和测试。失败会自动重试。没有代码审查就不会发布。

```
plan → setup → implement → verify → test → PR → review
```

### security-audit `7 个代理`

指向一个仓库。得到一个带有回归测试的安全修复 PR。扫描漏洞，按严重性排序，为每个漏洞打补丁，修复后重新审计。

```
scan → prioritize → setup → fix → verify → test → PR
```

### bug-fix `6 个代理`

粘贴一个错误报告。得到一个带有回归测试的修复。分类者重现问题，调查者找出根本原因，修复者打补丁，验证者确认。无需人工干预。

```
triage → investigate → setup → fix → verify → PR
```

---

## 为什么它有效

- **确定性工作流**——相同的工作流、相同的步骤、相同的顺序。不再是“希望代理记得测试”。
- **代理相互验证**——开发者不会批改自己的作业。单独的验证者会根据验收标准检查每个故事。
- **每个步骤都有新上下文**——每个代理都会获得一个干净的会话。没有上下文窗口膨胀。没有来源于 50 条消息以前的虚构状态。
- **重试并升级**——失败步骤会自动重试。如果重试耗尽，会升级给你。没有静默失败。

---

## 工作原理

1. **定义**——在 YAML 中定义代理和步骤。每个代理都有一个角色、工作区和严格的验收标准。谁做什么一目了然。
2. **安装**——一条命令即会部署所有内容：代理工作区、cron 轮询、子代理权限。无需 Docker、队列或外部服务。
3. **运行**——代理独立轮询任务。领取步骤、完成工作、将上下文传给下一个代理。SQLite 跟踪状态。cron 保持流程。

### 极简设计

YAML + SQLite + cron。仅此而已。无 Redis、无 Kafka、无容器编排。Antfarm 是一个 TypeScript CLI，没有外部依赖。它可以在任何支持 OpenClaw 的地方运行。

### 基于 Ralph 循环构建

<img src="https://raw.githubusercontent.com/snarktank/ralph/main/ralph.webp" alt="Ralph" width="100">

每个代理在一个干净的会话中运行，带有清晰的上下文。记忆通过 git 历史和进度文件持久化——这与 [Ralph](https://github.com/snarktank/ralph) 中的自主循环模式相同，只是扩展到了多代理工作流。

---

## 快速示例

```bash
$ antfarm workflow install feature-dev
✓ Installed workflow: feature-dev

$ antfarm workflow run feature-dev "Add user authentication with OAuth"
Run: a1fdf573
Workflow: feature-dev
Status: running

$ antfarm workflow status "OAuth"
Run: a1fdf573
Workflow: feature-dev
Steps:
  [done   ] plan (planner)
  [done   ] setup (setup)
  [running] implement (developer)  Stories: 3/7 done
  [pending] verify (verifier)
  [pending] test (tester)
  [pending] pr (developer)
  [pending] review (reviewer)
```

---

## 构建你自己的

捆绑的工作流只是起点。在纯 YAML 和 Markdown 中定义你自己的代理、步骤、重试逻辑和验证门。如果你能写出一个提示，你就能构建一个工作流。

```yaml
id: my-workflow
name: My Custom Workflow
agents:
  - id: researcher
    name: Researcher
    workspace:
      files:
        AGENTS.md: agents/researcher/AGENTS.md

steps:
  - id: research
    agent: researcher
    input: |
      Research {{task}} and report findings.
      Reply with STATUS: done and FINDINGS: ...
    expects: "STATUS: done"
```

完整指南：[docs/creating-workflows_CN.md](docs/creating-workflows_CN.md)

---

## 安全性

你正在安装会在你的机器上运行代码的代理团队。我们对此非常认真。

- **仅维护仓库**——Antfarm 只从官方的 [snarktank/antfarm](https://github.com/snarktank/antfarm) 仓库安装工作流。不接受任意远程来源。
- **经过提示注入审查**——每个工作流在合并前都会经过提示注入攻击和恶意代理文件的审查。
- **欢迎社区贡献**——想添加工作流？提交 PR。所有提交在发布前都要经过仔细的安全审查。
- **默认透明**——每个工作流都是纯 YAML 和 Markdown。安装前你可以完全阅读并了解每个代理的操作。

---

## 仪表板

实时监控运行、跟踪步骤进度并查看代理输出。

![Antfarm dashboard](https://raw.githubusercontent.com/snarktank/antfarm/main/assets/dashboard-screenshot.png)

![Antfarm dashboard detail](https://raw.githubusercontent.com/snarktank/antfarm/main/assets/dashboard-detail-screenshot.png)

```bash
antfarm dashboard              # Start on port 3333
antfarm dashboard stop         # Stop
antfarm dashboard status       # Check status
```

---

## 命令

### 生命周期

| 命令 | 描述 |
|---------|-------------|
| `antfarm install` | 安装所有捆绑的工作流 |
| `antfarm uninstall [--force]` | 完全清理（代理、crons、数据库） |

### 工作流

| 命令 | 描述 |
|---------|-------------|
| `antfarm workflow run <id> <task>` | 开始运行 |
| `antfarm workflow status <query>` | 检查运行状态 |
| `antfarm workflow runs` | 列出所有运行 |
| `antfarm workflow resume <run-id>` | 恢复失败的运行 |
| `antfarm workflow list` | 列出可用工作流 |
| `antfarm workflow install <id>` | 安装单个工作流 |
| `antfarm workflow uninstall <id>` | 删除单个工作流 |

### 管理

| 命令 | 描述 |
|---------|-------------|
| `antfarm dashboard` | 启动 web 仪表板 |
| `antfarm logs [<lines>]` | 查看最近的日志条目 |

---

## 要求

- Node.js >= 22
- 在主机上运行的 [OpenClaw](https://github.com/openclaw/openclaw) **v2026.2.9+**
  - Antfarm 使用 cron 作工作流编排。旧版 OpenClaw 可能无法通过 `/tools/invoke` 暴露 cron 工具。Antfarm 会自动回退到 `openclaw` CLI，但推荐保持 OpenClaw 更新：`npm update -g openclaw`
- 用于 PR 创建步骤的 `gh` CLI

---

## 许可证
