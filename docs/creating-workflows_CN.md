# 创建自定义工作流

本指南介绍如何创建你自己的 Antfarm 工作流。

## 目录结构

```
workflows/
└── my-workflow/
    ├── workflow.yml          # 工作流定义（必需）
    └── agents/
        ├── agent-a/
        │   ├── AGENTS.md     # 代理说明
        │   ├── SOUL.md       # 代理角色
        │   └── IDENTITY.md   # 代理身份
        └── agent-b/
            ├── AGENTS.md
            ├── SOUL.md
            └── IDENTITY.md
```

## workflow.yml

### 最小示例

```yaml
id: my-workflow
name: My Workflow
version: 1
description: 工作流的功能描述。

agents:
  - id: researcher
    name: Researcher
    role: analysis
    description: 研究主题并收集信息。
    workspace:
      baseDir: agents/researcher
      files:
        AGENTS.md: agents/researcher/AGENTS.md
        SOUL.md: agents/researcher/SOUL.md
        IDENTITY.md: agents/researcher/IDENTITY.md

  - id: writer
    name: Writer
    role: coding
    description: 根据研究结果撰写内容。
    workspace:
      baseDir: agents/writer
      files:
        AGENTS.md: agents/writer/AGENTS.md
        SOUL.md: agents/writer/SOUL.md
        IDENTITY.md: agents/writer/IDENTITY.md

steps:
  - id: research
    agent: researcher
    input: |
      Research the following topic:
      {{task}}

      Reply with:
      STATUS: done
      FINDINGS: what you found
    expects: "STATUS: done"

  - id: write
    agent: writer
    input: |
      Write content based on these findings:
      {{findings}}

      Original request: {{task}}

      Reply with:
      STATUS: done
      OUTPUT: the final content
    expects: "STATUS: done"
```

### 顶级字段

| 字段 | 是否必需 | 描述 |
|-------|----------|-------------|
| `id` | 是 | 唯一的工作流标识符（小写、连字符） |
| `name` | 是 | 人类可读名称 |
| `version` | 是 | 整数版本号 |
| `description` | 是 | 工作流的功能说明 |
| `agents` | 是 | 代理定义列表 |
| `steps` | 是 | 按顺序排列的流水线步骤列表 |

### 代理定义

```yaml
agents:
  - id: my-agent            # 在此工作流内唯一
    name: My Agent           # 显示名称
    role: coding             # 控制工具访问（参见下文“代理角色”）
    timeoutSeconds: 900      # 可选：覆盖角色默认超时时间（秒）
    description: What it does.
    timeoutSeconds: 1800     # 可选：覆盖隔离会话超时时间（秒）
    workspace:
      baseDir: agents/my-agent
      files:                 # 为此代理提供的工作区文件
        AGENTS.md: agents/my-agent/AGENTS.md
        SOUL.md: agents/my-agent/SOUL.md
        IDENTITY.md: agents/my-agent/IDENTITY.md
      skills:                # 可选：安装到工作区的技能
        - antfarm-workflows
```

文件路径相对于工作流目录。你也可以引用共享代理：

```yaml
workspace:
  files:
    AGENTS.md: ../../agents/shared/setup/AGENTS.md
```

### 代理角色

角色控制运行期间每个代理可以访问的工具：

| 角色 | 访问 | 典型代理 |
|------|--------|----------------|
| `analysis` | 只读代码探索 | planner、prioritizer、reviewer、investigator、triager |
| `coding` | 完全读/写/执行用于实现 | developer、fixer、setup |
| `verification` | 读取 + 执行但**不写** — 保持验证完整性 | verifier |
| `testing` | 读取 + 执行 + 浏览器/网络用于 E2E 测试，**不写** | tester |
| `pr` | 只读 + 执行 — 运行 `gh pr create` | pr |
| `scanning` | 读取 + 执行 + 网络搜索 CVE，无写权限 | scanner |

每个角色都有默认超时（取决于角色为 20 或 30 分钟）。在代理定义中设置 `timeoutSeconds` 可覆盖。

### 步骤定义

```yaml
steps:
  - id: step-name           # 唯一的步骤标识符
    agent: agent-id          # 该步骤由哪个代理处理
    input: |                 # 提示模板（支持 {{variables}}）
      Do the thing.
      {{task}}               # {{task}} 始终是原始任务字符串
      {{prev_output}}        # 来自前一步的变量（大写 KEY 降为小写）

      Reply with:
      STATUS: done
      MY_KEY: value          # KEY: value 对将成为后续步骤的变量
    expects: "STATUS: done"  # 输出必须包含的字符串，表示成功
    max_retries: 2           # 失败时重试次数（可选）
    on_fail:                 # 重试耗尽时的处理（可选）
      escalate_to: human     # 升级给人工
```

### 代理超时

Antfarm 将工作流代理作为 OpenClaw 中隔离的 cron 作业运行。你可以在代理定义中使用 `timeoutSeconds` 覆盖每个代理会话的超时：

```yaml
agents:
  - id: developer
    timeoutSeconds: 3600
```

如果省略，Antfarm 默认每个代理会话 30 分钟。

### 模板变量

步骤通过输出中的 KEY: value 对进行通信。当代理返回：

```
STATUS: done
REPO: /path/to/repo
BRANCH: feature/my-thing
```

后续步骤可以引用 `{{repo}}` 和 `{{branch}}`（KEY 名称小写）。

`{{task}}` 始终可用——它是传给 `workflow run` 的原始任务字符串。

### 验证循环

步骤可以在失败时重试先前的步骤：

```yaml
- id: verify
  agent: verifier
  input: |
    Check the work...
    Reply STATUS: done or STATUS: retry with ISSUES.
  expects: "STATUS: done"
  on_fail:
    retry_step: implement    # 使用反馈重新运行此步骤
    max_retries: 3
    on_exhausted:
      escalate_to: human
```

当验证出现 `STATUS: retry` 时，`implement` 步骤会再次运行，并且 `{{verify_feedback}}` 会从验证者的 `ISSUES:` 输出中填充。
