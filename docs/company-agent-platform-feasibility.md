# 基于 Pi 的公司级多 Agent 研发平台：源码评估与 PoC 计划

> 状态：架构调研 / 待验证。基线：`leth0x01/pi` 的 `develop` 分支（2026-11-01）。
>
> 范围：只做架构分析和实验设计；未宣称完成部署、性能测试或跨进程可靠性验证。所有业务规则、资源限制和验收阈值均属于本项目的设计提案，而非 Pi 官方承诺。

## 1. 结论与边界

**建议：使用 Pi 作为 Agent 执行核心，不把 Pi CLI 直接当作完整的公司研发管理平台。** 第一阶段在 Pi TypeScript SDK 上搭建独立的任务编排、权限、工件与审计层；并行验证实验性 `pi-durable`，通过故障恢复实测决定是否承担持久化运行时职责。

不要先改写 `packages/agent`、`packages/coding-agent` 或 `packages/ai`。优先使用扩展、SDK、MCP 与独立应用目录；保持与上游同步的可能性。

当前范围：
- 企业需求进入任务池，经产品/规划、技术负责人、开发、代码审查、测试、验收等专职 Agent 流转。
- 同时支持长期维护多个 Git 仓库及新项目；代码修改通过隔离工作区和 Pull/Merge Request 审核。
- 接入 Gitea/GitHub、Jenkins、文档检索与企业聊天工具；后续按需要接入 SVN 同步服务。
- 保留审计、权限边界、失败恢复、上下文与长期知识，而非只实现「多个 Agent 相互聊天」。

暂不纳入首个 PoC：无人监管生产部署、直接修改生产 SVN、完整企业级 Web 门户、跨多数据中心自动调度。

## 2. 已核对的 Pi 源码能力

| 模块 | 代码或文档证据 | 已有能力 | 当前评估 |
|---|---|---|---|
| `packages/ai` | [README](../packages/ai/README.md) | 统一模型/供应商接口、流式模型响应 | 直接复用 |
| `packages/agent` | [Agent Core](../packages/agent/README.md)、[agent-loop.ts](../packages/agent/src/agent-loop.ts) | Agent Loop、状态、工具调用、事件流、并行/顺序执行策略、前后置钩子 | 直接复用 |
| `packages/coding-agent` | [SDK](../packages/coding-agent/docs/sdk.md)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts) | 创建会话、工具控制、会话事件、项目资源发现 | 第一阶段首选接入点 |
| Sessions / Context | [Sessions](../packages/coding-agent/docs/sessions.md)、[How Pi Works](../packages/coding-agent/docs/how-pi-works.md) | JSONL 会话树、分支、续接、历史压缩 | 复用为执行会话；不等价于组织长期记忆 |
| 扩展与 Skills | [Extensions](../packages/coding-agent/docs/extensions.md)、[Skills](../packages/coding-agent/docs/skills.md) | 注册工具、拦截事件、技能指令、扩展状态 | 优先扩展，避免核心 Fork |
| MCP | [MCP](../packages/coding-agent/docs/mcp.md) | stdio/Streamable HTTP 接入工具与资源 | 连接开发工具；必须另加权限策略 |
| RPC | [RPC](../packages/coding-agent/docs/rpc.md) | 通过 JSONL 子进程交互，支持事件/命令 | 非 Node 服务或进程隔离的备选 |
| `packages/durable` | [Durable README](../packages/durable/README.md) | 实验性持久化 Harness、存储、任务/子任务、任务图、恢复机制 | 必须 PoC 验证，不能视为成熟生产编排平台 |
| 运行安全 | [Security](../packages/coding-agent/docs/security.md)、[Containerization](../packages/coding-agent/docs/containerization.md) | Project trust 与隔离方案文档 | Pi **默认没有操作系统级权限沙箱**，安全边界须自行部署 |

### 重要区分

1. **Pi CLI 没有开箱即用的公司级多 Agent 交付流程**，不意味着 Pi 所有组件都没有子任务机制。`pi-durable` 明确提供 owned conversations（子 Agent 场景）、child tasks、task graph，以及 SQLite/JSONL 等存储适配器，但 README 标注为 **Experimental**，API 可能变化。
2. `Agent` 的工具并行执行指的是同一次模型调用中的**工具并发**，不是跨项目、多角色、含审批和依赖的企业任务调度。
3. 会话树、压缩摘要保存的是**会话上下文**。检索型长期知识、可信事实、项目决策、代码索引、权限与过期处理仍需在平台层实现。
4. MCP 服务可被 Pi 调用，不代表具备租户隔离、凭据授权、工具风险分级或审计审批。
5. Pi CLI/RPC/SDK 均是可用接口；对 TypeScript/Node 宿主，官方更推荐 SDK；进程隔离、异构语言宿主可采用 RPC。

## 3. 建议架构

```text
企业聊天 / Web / 工单 / API
             |
     需求入口 + 身份认证
             |
     Task API / 状态机 / 审批
             |
  Coordinator（确定性规则优先）
      |            |
  任务与工件库     事件、成本与审计
      |
  Agent Workers（各自工作区、各自权限）
  ├─ PM / Planner: 需求拆解、验收标准
  ├─ Architect: 方案评审、技术依赖
  ├─ Developer: 实施、测试、提交 PR
  ├─ Reviewer: 独立代码审查
  └─ QA: 构建、测试、回归证据
      |
  Pi SDK / Pi Agent Core
      |
  Skills / Extensions / MCP
      |
 Git 代码仓库 / Jenkins / 文档检索 / 受控工具
```

**核心原则：AI 负责分析与产出，平台状态机负责确定任务是否可以进入下一步。** 不让单个模型仅凭口头表示「已测试」或「已完成」就完成交付。

### 3.1 角色职责和交付物

| 角色 | 输入 | 必须交付 | 禁止自行越权 |
|---|---|---|---|
| PM / Planner | 需求或 Issue | 需求范围、验收标准、未决问题 | 将不明确需求标记为验收通过 |
| Architect | 需求、仓库结构 | 技术方案、影响范围、实施顺序 | 未获授权改动代码或生产配置 |
| Developer | 已确认任务 | feature 分支、代码、测试、PR | 直接推送保护分支 |
| Reviewer | PR、需求、测试结果 | 审查结论、风险、可复现问题 | 审批自己的实现 |
| QA | 构建制品、测试计划 | 运行记录、失败日志、回归结论 | 用模型描述替代实际测试 |
| Release（后续） | 已通过验收的版本 | 发布计划、变更记录、回滚步骤 | 未批准上线或读取生产密钥 |

一项跨端任务可包含 Android、iOS、服务端和固件子任务；先由 Coordinator 构建依赖图，再并行安排可独立实施的部分，统一集成测试。

### 3.2 任务状态

推荐最小状态机：

`NEW -> TRIAGED -> SPEC_READY -> APPROVED -> IN_PROGRESS -> REVIEW -> TEST -> ACCEPTED -> DONE`

任意阶段可流向 `BLOCKED`、`FAILED` 或 `NEEDS_REWORK`；回退原因、责任 Agent、工件和审批人必须持久化。不可使用自由文本作为唯一状态依据。

每个任务最少包括：

```text
task_id, project_id, repository, base_commit, work_branch
request, acceptance_criteria[], dependencies[], assignee_role
status, attempt, deadline?, approval_policy
artifact_refs[], tool_run_refs[], test_results[], review_refs[]
created_at, updated_at, idempotency_key
```

要求可根据 `task_id` 重建「谁执行了什么、使用了哪个提交、经过哪些门禁」。

## 4. 执行、上下文与长期记忆

### 4.1 执行隔离

- 每个写代码的 Agent 使用独立 Git worktree（或单独 clone），禁止多个 Agent 在同一工作目录同时写文件。
- 每个执行单元使用独立容器/沙箱、限定资源和访问目录；默认无生产网络、无生产凭据。
- 开发 Agent 只获取任务所需仓库的最小写权限；Review/QA 优先只读代码及制品。
- 密钥在平台凭据管理层按任务注入，不写入 Prompt、仓库、会话摘要、长期知识。
- 合并与发布至少需要独立审查、可追溯 CI 结果及明确批准；平台验收不能仅依赖 LLM 自评。

### 4.2 四层记忆（平台设计，不是 Pi 开箱即用能力）

1. **工作记忆**：当前 Pi Agent 的短期上下文、工具响应和正在处理的任务。
2. **情景记忆**：任务事件流、会话、PR 讨论、测试日志、失败与重试历史（按 `task_id` 可追溯）。
3. **语义记忆**：已确认的项目规则、接口契约、架构决策、技术知识（带来源与生效范围）。
4. **程序性记忆**：Skills、工程规范、自动化检查与可复用工具配置；建议版本化随仓库管理。

记忆写入策略：观察 -> 摘要/提取 -> 证据与权限校验 -> 去重/冲突处理 -> 发布为项目知识。默认只共享经审核的项目知识；个人会话或其他项目的敏感信息不得跨作用域自动传播。

检索优先级：项目现行规范与代码 > 已批准的架构决策 > 当前任务记录 > 历史经验。任何摘要不能覆盖真实代码、测试输出与提交记录。

## 5. 两条技术路线与取舍

| 方案 | 优点 | 风险 | 决策 |
|---|---|---|---|
| A. Pi SDK + 自研精简 Coordinator | API 清晰、便于按企业验收流程建模、可把状态和权限留在自有服务 | 需要自己做调度、恢复、幂等与审计 | **PoC 主线** |
| B. pi-durable 承担持久运行时 + 外部治理层 | 已有任务、子任务、状态恢复、存储后端与任务图 | Experimental；需验证版本兼容、故障边界、跨进程拓扑、外部副作用恢复 | **PoC 并行实验** |
| C. Pi RPC Worker + 外部调度 | 进程隔离、语言无关、日志流易接 | 进程生命周期、事件处理和恢复需额外实现 | 作为 A/B 的 Worker 备选 |

选型门槛：以实际断点恢复、重复操作防护、事件完整性、开发效率和长期维护成本决定，不因任何工具已部署而优先。

特别注意：`pi-durable` 文档明确一个进程拥有一个 storage，**不能未经验证就把单个 SQLite 文件当作多 Worker 共享队列**；生产分布式一致性、任务迁移与锁策略需要另行设计。调用 Jenkins、创建 PR、提交代码等外部副作用，必须通过业务幂等键及查询校验实现安全重试。

## 6. PoC 环境与网络

建议先在单台 Linux 实验机运行 Coordinator + 一个或两个 Worker，CPU/内存容量按实际模型运行方式、并发工具和构建负载测量，暂不报未经测量的资源需求。Node.js 选已配置的 v24，满足仓库 `>=22.19` 要求。编译任务尽量交给专门 CI 节点，而非长期运行的 Agent 主机。

按现有公司网络规划执行以下原则（具体互通性与防火墙规则上线前实测）：

- Agent 主机：受控访问 Git 服务及必要的外部模型 API。
- 外网 Git：优先用既有 3 段或 40 段 Gitea/Git 路径与最小权限凭据。
- 内网 254 段 SVN/Jenkins：经现有受控同步/构建入口完成交互，**不要**让 Agent Worker 为了方便直接跨网访问编译内网。
- Jenkins：使用专门服务账号和参数受限 Job；Agent 只能请求构建、读取结果，不拥有 Jenkins 全局管理权。
- Web/聊天入口：由入口服务认证用户，再映射权限到任务；不能把聊天消息当作系统命令执行。

PoC 的第一版可以只做单机内网试验，不开放公网管理 API。

## 7. 分阶段实施与验收

### P0：源码可运行与基线（先做）

1. 从 `develop` 建独立试验分支，记录 Pi commit SHA、Node/npm 版本；按根目录 [AGENTS.md](../AGENTS.md) 和 [README](../README.md) 安装。
2. `npm ci --ignore-scripts`、`npm run build:offline`（如模型数据尚未就绪，按仓库指导先补齐数据）、`npm run check`；记录错误，不把未运行当作通过。
3. 通过 SDK 建立单个受控 Agent：只读列文件 -> 输出分析 -> 结束并释放 session。
4. 用 RPC 做同任务的进程隔离对照，验证事件、退出和超时处理。

验收：本地构建与检查日志可复现，至少一条 SDK 任务可产出包含 session ID 和 commit SHA 的记录。

### P1：两 Agent 完整闭环（推荐第一个业务 PoC）

选用一个低风险的真实 Git 项目，建立如下流程：

`Issue -> Planner 拆解 -> Developer 建分支与修改 -> CI -> Reviewer 独立审查 -> QA 验证 -> 人工验收`

最低要求：
- Agent 自身不能合并开发分支；
- 每次任务生成独立工作目录/分支；
- PR、CI 日志、测试结果关联到同一个 task_id；
- 测试失败自动进入 NEEDS_REWORK，阻断 DONE；
- 可以中止任务、查看当前责任人和产出；
- 对未授权写文件和访问非目标仓库的请求要拦截，并由沙箱限制。

验收：一次成功任务、一次测试失败后返工、一次审批拒绝都能按预期流转。

### P2：持久化与安全故障演练

在隔离环境评估 `pi-durable`，至少验证：
1. 模型请求过程中重启，恢复后没有丢失任务状态；
2. 工具调用完成但提交状态前崩溃，重试不造成重复 PR、重复 Jenkins Job 等外部副作用；
3. 父任务等待多个子任务，其中一项失败后，状态和取消传播可解释；
4. 中止/恢复/重试后，任务事件与审计可完整回放；
5. 两个不同项目的知识不会误共享，Agent 无法读取其他项目密钥；
6. 会话历史压缩后仍可从原始记录找回验收证据。

验收：有测试输入、运行日志、失败与恢复截图或结构化记录；若任一关键安全/幂等场景失败，则保持方案 A，不直接采用实验运行时。

### P3：企业系统接入（在 P1、P2 通过后）

逐项接入：Gitea/GitHub 工单 -> Jenkins 构建 -> 内网受控同步服务 -> 企业聊天通知 -> 最后才是 Web 门户与跨项目资源池。

## 8. 首批建议开发文件（尚未创建）

PoC 验证通过后，优先在 Fork 的独立应用目录开发，保持上游包不变：

```text
apps/agent-platform/
  src/
    api/              # 任务入口
    orchestration/    # 状态机、依赖、审批、恢复
    workers/          # Pi SDK/RPC 适配器
    roles/            # 角色及输出结构
    integrations/     # Git、CI、企业通知
    memory/           # 来源、作用域、可信度
    security/         # 凭据、沙箱和权限策略
  test/
  README.md
```

建议优先建立三个实现切片：
- Slice 1：`task_id + state machine + SDK worker + 事件日志`；
- Slice 2：`Git 分支/PR + Jenkins CI + Review/QA 门禁`；
- Slice 3：`重启恢复 + 幂等 + 多 Agent 子任务 + 项目知识`。

## 9. 风险清单与当前状态

| 风险 | 控制手段 |
|---|---|
| 上游 Pi 更新造成大量合并冲突 | 优先外部 app/SDK/extension，不重写上游核心包 |
| pi-durable 破坏性 API 变化 | 独立实验并锁定依赖版本，先做故障演练 |
| 多 Agent 相互覆盖代码 | 按任务/Agent 隔离 worktree，保护主分支 |
| 重试重复触发外部操作 | task_id/idempotency_key + 外部状态反查 |
| 指令注入、工具越权 | 沙箱、最小凭据、网络白名单、人工门禁 |
| 「模型说成功」被误判为成功 | 以 Git commit、CI、测试输出和审批记录为准 |
| 上下文摘要丢失关键信息 | 原始事件保留，知识分层，有来源的决策才能写入长期知识 |
| 大量模型调用造成预算失控 | 每任务 token/费用上限、重试预算、告警和终止条件 |

**目前完成：** 按仓库文档和关键源码进行静态分析，形成实验路线。

**尚未完成：** 实际部署、单元测试、CI、性能和安全实验；本文中的所有验收案例均是待执行事项。

**下一项应执行的任务：** P0 最小 SDK 可运行样例与本地环境检查；随后开发 P1 的最小任务状态机和双 Agent 闭环。
